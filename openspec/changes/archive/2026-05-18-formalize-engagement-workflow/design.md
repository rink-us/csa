## Context

`/netd:engagement` was built to orchestrate the network-discovery and reporting skills into a complete client deliverable. After running it (manually, walked-through) against the user's own home network during the prior session, three implicit assumptions surfaced that the slash command's prose documentation can't enforce:

- The sidecar JSON is treated as the audit artifact for the engagement, but its schema is described in narrative rather than declared. Two engagements can produce structurally different sidecars if the operator follows the slash command literally vs. interprets it loosely.
- The phase order (precheck → scope confirmation → execute → sidecar) prevents wrong-network mistakes. But because the slash command is prose, an operator under time pressure can mentally collapse the phases and skip the scope-confirmation prompt — and there's no after-the-fact way to tell from the sidecar whether they did.
- The artifact set (sidecar + technical report + client letter) is "what good looks like" rather than "what's required." A `no-letter` engagement is valid today but produces an audit record that's ambiguous about whether the letter was intentionally skipped vs. forgotten.

A formal capability spec for the engagement workflow turns each of these from convention into contract.

## Goals / Non-Goals

**Goals:**
- Define `network-assessment-engagement` as a capability with a normalized sidecar JSON schema, a required phase order, and a defined artifact set
- Make the sidecar a self-attesting audit record: by reading it, an auditor can tell whether each required phase happened, when, and with what input
- Keep the implementation orchestration-light — the slash command stays the operator entry point; the spec governs WHAT happens, not which command runs it
- Preserve backwards compatibility — existing sidecars written before this change remain valid (the new required fields are additive)

**Non-Goals:**
- Not creating a "pentest" or "compliance" engagement variant. The scope is the existing network-assessment workflow only. Other engagement types are future changes.
- Not specifying the technical report or client letter shape — those already have their own capability specs (`assessment-report`, `client-report`). This spec composes them, doesn't redefine them.
- Not adding new external dependencies. The spec describes orchestration; the underlying skills handle execution.
- Not specifying how engagements get scheduled, billed, or archived to long-term storage. Those are operator-policy concerns outside the capability.

## Decisions

**1. The capability name is `network-assessment-engagement`, not `engagement`.**
- **Decision**: Use the qualified name. Allow for future `pentest-engagement`, `compliance-engagement`, etc. as separate capabilities.
- **Rationale**: `engagement` alone is too generic — it could refer to a sales engagement, a customer-success engagement, an incident-response engagement. Qualifying with the playbook type keeps the spec library legible.

**2. The sidecar schema is declared as a named output type, alongside the other shared shapes.**
- **Decision**: Add `EngagementRecord` to [OUTPUT_SCHEMAS.md](../../../.agents/skills/network-discovery/OUTPUT_SCHEMAS.md) so it sits next to `Host`, `Port`, `Cve`, etc. The spec references it by name.
- **Rationale**: Treating the sidecar as a first-class shape (not a one-off file format) lets downstream tooling consume it with the same schema-driven approach as the other artifacts. Also makes future schema evolution explicit (a new field is a delta to a named shape, not a prose edit).

**3. Phase order is enforced via required sidecar fields.**
- **Decision**: The sidecar has fields like `precheck_completed_at`, `scope_confirmed_at`, `execution_started_at`, `execution_finished_at`. Each is required. If a phase is genuinely skipped (e.g. precheck override), the field carries the reason explicitly (`"skipped: operator override - reason X"`).
- **Rationale**: An auditor reading the sidecar can tell at a glance whether each phase happened. A missing timestamp = the implementation broke the contract; a "skipped" value = the operator made a deliberate choice on the record.
- **Alternative considered**: trust the slash command to enforce order, no sidecar tracking. Rejected — provides no after-the-fact verifiability, and any other implementation of the workflow (e.g. a future Python script) could silently skip phases.

**4. The artifact set is required, with explicit "skipped" markers.**
- **Decision**: The sidecar has `artifacts: { technical_report: <path>, client_letter: <path or "skipped: reason"> }`. If the operator passes `no-letter`, the sidecar records `"skipped: --no-letter"` rather than omitting the field.
- **Rationale**: Same auditability argument. Distinguishes "letter was not generated" from "letter was lost or never written."

**5. Authorization reference is required and free-form.**
- **Decision**: The sidecar field `authorization.reference` is required and accepts any string (SOW ID, ticket number, email subject, "self" for own-infra engagements). The spec does NOT validate the content — that's an operator-policy concern.
- **Rationale**: A required field forces the operator to make an explicit choice. Free-form value avoids dictating documentation systems (SOW tooling varies wildly across consultancies). For the same-network case where the operator owns the infra, "self" is a documented acceptable value.

**6. Engagement IDs are derived deterministically from client slug + date.**
- **Decision**: `engagement_id` is `<slugified-client-name>-<YYYY-MM-DD>` (UTC). Two engagements for the same client on the same day collide intentionally — operator must resolve by appending a suffix (`-2`, `-followup`, etc.).
- **Rationale**: Predictable IDs make the sidecar filename and the engagement_id agree (the filename pattern is `engagement-<engagement_id>.json`). Collision handling is the operator's call because the right resolution depends on context (same client got two visits, or a re-run of a failed engagement).

## Risks / Trade-offs

**Risk: The sidecar schema becomes a maintenance burden as engagement formats evolve.**
- **Mitigation**: Versioned via `playbook_version` field already present. Additive evolution is safe; breaking changes require a delta change that updates the schema.

**Risk: An operator under time pressure ignores the spec and writes ad-hoc sidecars by hand.**
- **Mitigation**: Out of scope. The spec describes what conforming implementations must produce; non-conforming sidecars are detectable by schema validation and the audit trail breaks at that point.

**Risk: The required-phase fields invite false-attestation (operator writes a precheck_completed_at without actually running the precheck).**
- **Mitigation**: The spec can't prevent operator dishonesty. It can make the dishonesty traceable — if the technical report's `tools_used` doesn't match what the precheck would have surfaced, the inconsistency is visible. This is the same trust model as code-review attestations.

**Risk: Backwards compatibility with the existing sidecars created during the prior session.**
- **Mitigation**: New required fields with sensible defaults can be backfilled by a one-time migration (or the spec can declare that pre-v1.0 sidecars are grandfathered without the required-phase fields). The current sidecar count is small (zero in `reports/` — the existing files are reports, not engagement records), so the migration cost is effectively zero.

## Migration Plan

1. **Phase 1**: Author the `network-assessment-engagement` spec as `ADDED Requirements`. Define each required phase, each sidecar field, each artifact-set member.
2. **Phase 2**: Add `EngagementRecord` to `OUTPUT_SCHEMAS.md` as a delta in this change (the SKILL.md / shared-docs files are repo-relative, so the modification is recorded in tasks.md).
3. **Phase 3**: Update the slash command at `.agents/commands/netd/engagement.md` to write the sidecar in conformance with the new schema — adding the required phase-timestamp fields and the explicit "skipped" markers.
4. **Phase 4**: Update the playbook at `playbooks/network-assessment.md` to point at the new spec as source of truth and document the "skipped" marker convention for operators.
5. **Phase 5**: Archive with `/opsx:archive` to promote the new spec to the main library.

No code changes are needed in the underlying network-discovery skills — the engagement spec composes them, doesn't modify them.

## Resolved Questions

- **Engagement-type taxonomy** — `network-assessment-engagement` is the first; future types (`pentest-engagement`, `compliance-engagement`, `webapp-engagement`) are their own capabilities and their own follow-up changes. This change does not create the taxonomy machinery; it just names the first member.
- **Schema versioning** — `playbook_version` field in the sidecar carries the version. Bumped only when the schema breaks (adding fields with sensible defaults is non-breaking).
- **Operator-policy enforcement** — out of scope. The spec describes the contract; whether to require notarized authorization vs. an email thread vs. "self" is a per-consultancy or per-operator decision.

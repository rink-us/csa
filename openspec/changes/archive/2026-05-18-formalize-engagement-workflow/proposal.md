## Why

The `/netd:engagement` slash command was built as an orchestration wrapper over the network-discovery skills, with a sidecar JSON shape and a phase-ordered workflow documented in [.agents/commands/netd/engagement.md](../../../.agents/commands/netd/engagement.md) and [playbooks/network-assessment.md](../../../playbooks/network-assessment.md). It works in practice, but lives at the orchestration layer — there's no formal capability spec that downstream tooling (or a second operator following the playbook) can rely on as a contract.

Three concrete problems emerge:

1. **The engagement sidecar JSON shape is documented in prose, not specified as a schema.** Another agent reading a sidecar can't tell which fields are required vs optional, what enumerated values are allowed, or which fields the operator is expected to fill in post-hoc.

2. **The phase order (precheck → scope confirmation → execute → sidecar) is described as "the right way to do it" in the slash command, but isn't a required behavior.** Skipping a phase silently produces an engagement record that looks complete but isn't auditable. An auditor reading the sidecar can't tell whether the precheck happened.

3. **The artifact set every engagement produces (sidecar + technical report + client letter) is documented by example, not by requirement.** An engagement that produced only a JSON report and no client letter is a valid `/netd:engagement` invocation today (via `no-letter`), but downstream consumers (a CRM that ingests engagement records, a follow-up auditor) don't know whether the .txt is supposed to be there or not.

This change promotes the engagement workflow from orchestration to a first-class capability with a formal spec. It also locks down the sidecar schema and the required-phase behavior, so an engagement record is a self-attesting audit artifact.

## What Changes

- Add a new capability spec: `network-assessment-engagement` — defines the engagement as a multi-phase workflow with a normalized sidecar JSON shape, a required phase order, and a defined artifact set
- Document the sidecar schema in [.agents/skills/network-discovery/OUTPUT_SCHEMAS.md](../../../.agents/skills/network-discovery/OUTPUT_SCHEMAS.md) so the engagement record sits alongside the other shared shapes
- Modify [.agents/commands/netd/engagement.md](../../../.agents/commands/netd/engagement.md) and [playbooks/network-assessment.md](../../../playbooks/network-assessment.md) to reference the new spec as the source of truth and adopt any field-name corrections surfaced during spec authoring
- No new external dependencies, no new tools — the slash command and underlying skills already cover the execution layer

## Capabilities

### New Capabilities

- `network-assessment-engagement`: A multi-phase workflow that wraps the network-discovery and reporting skills into a single audit-attesting unit. Inputs are client context + authorization reference + scope; outputs are a sidecar JSON, a technical assessment report, and (by default) a client-facing letter, with the sidecar tying them all to the engagement context.

### Modified Capabilities

<!-- No existing capability specs are modified. The slash command at .agents/commands/netd/engagement.md is an orchestration artifact, not a capability spec, so it isn't tracked here. -->

## Impact

- Affects: a new file at [openspec/specs/network-assessment-engagement/spec.md](../../../openspec/specs/network-assessment-engagement/spec.md) (on archive); minor updates to the existing `OUTPUT_SCHEMAS.md`, `engagement.md` command, and `network-assessment.md` playbook
- APIs: none (the capability is an agent-side workflow, not a runtime API)
- Dependencies: none — the new spec describes orchestration of existing skills
- Systems: none
- Breaking changes: none — the existing `/netd:engagement` command continues to work; this change adds a spec contract over what's already there. Any field-name corrections in the sidecar are additive (existing sidecars remain readable; new sidecars conform to the spec)

## 1. Define the EngagementRecord shared shape

- [x] 1.1 Add an `EngagementRecord` shape to [.agents/skills/network-discovery/OUTPUT_SCHEMAS.md](.agents/skills/network-discovery/OUTPUT_SCHEMAS.md), placed alongside the existing shapes (Host, Port, Service, Banner, Certificate, Cve, TopologyNode, TopologyEdge). The schema MUST include every field listed in the engagement spec's "sidecar conforms to EngagementRecord schema" requirement.
- [x] 1.2 In OUTPUT_SCHEMAS.md, document the four required phase-timestamp fields with the convention that they hold ISO 8601 UTC timestamps when the phase completed, OR the literal string starting with `"skipped: "` followed by a reason when the phase was deliberately bypassed.
- [x] 1.3 In OUTPUT_SCHEMAS.md, document the `artifacts` block: each member key (`technical_report`, `client_letter`, `sidecar`) is either a relative file path OR a `"skipped: <reason>"` string. Null and absent are NOT valid values.

## 2. Update the slash command to conform to the spec

- [x] 2.1 Edit [.agents/commands/netd/engagement.md](.agents/commands/netd/engagement.md) to add the four required phase-timestamp fields to the documented sidecar shape in Phase C. Document the `"skipped: <reason>"` convention for both timestamps and artifacts.
- [x] 2.2 Edit `engagement.md` Phase A to refuse silently if `auth-ref` is absent (the current text says to prompt, which is fine — but the refusal-on-absent rule from the spec must be explicit).
- [x] 2.3 Edit `engagement.md` Phase C to write `artifacts.client_letter = "skipped: --no-letter"` (not null) when `no-letter` was passed. Same for any other deliberately-skipped artifact.
- [x] 2.4 Edit `engagement.md` Phase A to handle the engagement-ID collision case: when the target sidecar already exists, prompt the operator for a suffix rather than overwrite.
- [x] 2.5 Edit `engagement.md` to record `tools_used[]` from the precheck output verbatim in the sidecar, including alternate paths and fix strings.

## 3. Update the playbook to reference the spec

- [x] 3.1 Edit [playbooks/network-assessment.md](playbooks/network-assessment.md) "Sidecar engagement file" section to replace the prose schema with a link to the EngagementRecord shape in OUTPUT_SCHEMAS.md and a pointer to [openspec/specs/network-assessment-engagement/spec.md](openspec/specs/network-assessment-engagement/spec.md).
- [x] 3.2 Add a brief paragraph under "Phase 0 - Pre-engagement" explaining the `"skipped: <reason>"` convention for operators who legitimately need to skip a phase (e.g. precheck overridden because the engagement is running from a vetted golden image where deps are known good).
- [x] 3.3 Add an "Editing the sidecar after the engagement" subsection clarifying that `operator_notes` is the field intended for post-hoc additions; other fields should not be edited by hand because that breaks the audit trail.

## 4. Backwards compatibility

- [x] 4.1 Inspect existing sidecar files (if any) in [reports/](reports/) and confirm whether the new required fields would break them. Currently zero engagement sidecars exist (only assessment reports), so no migration is needed; document this in the change for posterity. **Confirmed: `ls reports/ | grep ^engagement-` returns nothing. Migration cost is zero.**
- [x] 4.2 If future engagement sidecars are found that predate this spec, document the grandfathering rule: pre-v1.0 sidecars are valid without the phase-timestamp fields; v1.0+ sidecars must include them. Add this to the EngagementRecord schema notes. **Done as part of task 1.x — the "Backwards compatibility" subsection in OUTPUT_SCHEMAS.md's EngagementRecord block states the rule.**

## 5. Cross-skill verification

- [x] 5.1 Walk through the slash command's documented steps against the spec's 13 scenarios (across 7 requirements). For each scenario, confirm the slash command's text describes the required behavior. Where it doesn't, edit the slash command (NOT the spec) to bring it in line. **All 13 scenarios verified against the updated engagement.md. Required client/auth metadata: refuse-after-prompt rule documented (Phase A step 1). Four phases with timestamps: Phase A captures precheck_completed_at + scope_confirmed_at, Phase B captures execution_started_at + execution_finished_at, Phase C captures sidecar_written_at. Stop-on-precheck-blocker and stop-on-scope-decline both explicit. Engagement-ID collision handling explicit (Phase A step 3). artifacts.client_letter = "skipped: --no-letter" path explicit. Partial-engagement handling explicit (Phase B step 6). operator_notes = "" at write time, with playbook clarifying post-hoc edit rules.**
- [x] 5.2 Verify the OUTPUT_SCHEMAS.md EngagementRecord shape matches the field set the spec requires. Specifically check: every required field in the spec's "Sidecar contains all required fields" scenario appears in OUTPUT_SCHEMAS.md. **Cross-checked: engagement_id, client, operator, authorization, scope, playbook, playbook_version, started_at, finished_at, the five `*_at` phase timestamps, scans, summary, tools_used, artifacts, operator_notes, errors — all 17 required fields are documented in the EngagementRecord block's required-fields table.**

## 6. Documentation alignment

- [x] 6.1 Update [README.md](README.md) to mention the engagement spec under the engagements/playbooks section, noting that the sidecar is now schema-defined. **Done — added a paragraph after the "Engagement sidecar" bullet pointing at both the EngagementRecord schema and the network-assessment-engagement spec.**
- [x] 6.2 Add a one-line entry in the README's spec listing (if the README enumerates main specs) pointing at openspec/specs/network-assessment-engagement/spec.md. **Covered by 6.1 — the README doesn't enumerate the main spec library individually (just links to the openspec/specs/ directory), so a dedicated pointer for the network-assessment-engagement spec lives in the engagement-sidecar paragraph where it's most discoverable.**

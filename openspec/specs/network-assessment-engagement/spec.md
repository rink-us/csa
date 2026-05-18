## ADDED Requirements

### Requirement: Engagement accepts required client and authorization metadata
The agent skill SHALL require `client` (object with `name` string field) and `authorization` (object with `reference` string field) as inputs, and SHALL refuse to start an engagement when either is missing or empty.

#### Scenario: Engagement with full required metadata
- **WHEN** the operator invokes the engagement with `client.name="Acme Corp"` and `authorization.reference="SOW-2026-04-Acme"`
- **THEN** the engagement proceeds and the sidecar records both values

#### Scenario: Missing client name refused
- **WHEN** the operator invokes the engagement without `client.name`
- **THEN** the engagement refuses to start with a validation error and no sidecar is written

#### Scenario: Missing authorization reference refused
- **WHEN** the operator invokes the engagement without `authorization.reference`
- **THEN** the engagement refuses to start with a validation error and no sidecar is written

#### Scenario: "self" is an accepted authorization value
- **WHEN** the operator invokes the engagement with `authorization.reference="self"` against infrastructure they own
- **THEN** the engagement proceeds normally; the value is recorded verbatim

### Requirement: Engagement executes four phases in order
The agent skill SHALL execute the following phases in this order: (1) precheck — verify required CLI dependencies; (2) scope_confirmation — present the parsed scope plus the local network context (operator IP, default gateway) and require an affirmative confirmation before proceeding; (3) execution — run the documented scan/report chain; (4) sidecar — write the engagement record JSON. Each phase SHALL produce a sidecar timestamp recording when it completed.

#### Scenario: All four phases run successfully
- **WHEN** an engagement runs end-to-end with no errors
- **THEN** the sidecar has `precheck_completed_at`, `scope_confirmed_at`, `execution_started_at`, `execution_finished_at`, and `sidecar_written_at` populated with ISO 8601 UTC timestamps in chronological order

#### Scenario: Precheck blocker stops the engagement before execution
- **WHEN** the precheck phase finds a `missing` blocker (a required CLI is absent)
- **THEN** the engagement stops, no sidecar is written, and the operator is told what's missing

#### Scenario: Operator does not confirm scope
- **WHEN** the scope_confirmation prompt is not affirmatively answered (the operator cancels)
- **THEN** the engagement stops, no sidecar is written, no scan packets leave the operator's machine

### Requirement: Engagement sidecar conforms to the EngagementRecord schema
The agent skill SHALL write a sidecar JSON file conforming to the `EngagementRecord` shape (defined in [OUTPUT_SCHEMAS.md](../../../.agents/skills/network-discovery/OUTPUT_SCHEMAS.md)) at `./reports/engagement-<engagement_id>.json`.

#### Scenario: Sidecar file path matches engagement_id
- **WHEN** an engagement runs for `client.name="Acme Corp"` on 2026-05-17 UTC
- **THEN** `engagement_id="acme-corp-2026-05-17"` and the file is written to `./reports/engagement-acme-corp-2026-05-17.json`

#### Scenario: Sidecar contains all required fields
- **WHEN** the sidecar is written
- **THEN** the file contains (at minimum): `engagement_id`, `client`, `operator`, `authorization`, `scope`, `playbook`, `playbook_version`, `started_at`, `finished_at`, the four phase-timestamp fields, `scans[]`, `summary`, `tools_used[]`, `artifacts`, `operator_notes`

#### Scenario: Sidecar is the audit artifact
- **WHEN** a third party reads only the sidecar (without the linked technical report)
- **THEN** they can determine: who ran the engagement, when, against what scope, under what authorization, which phases completed, what tools were used, where the produced artifacts live

### Requirement: Engagement produces a defined artifact set
The agent skill SHALL produce, at minimum, a technical assessment report (the JSON written by `assessment-report`) and a sidecar JSON. The skill SHALL ALSO produce a client letter (the .txt written by `client-report`) by default. The `artifacts` block in the sidecar SHALL reference each produced file by path AND SHALL explicitly mark any skipped artifact with a `"skipped: <reason>"` string instead of omitting the field.

#### Scenario: Standard engagement produces all three artifacts
- **WHEN** an engagement runs with default flags
- **THEN** the sidecar's `artifacts` field contains `technical_report`, `client_letter`, and `sidecar` keys, each populated with a file path

#### Scenario: --no-letter records the deliberate skip
- **WHEN** the operator passes the equivalent of `no-letter`
- **THEN** the sidecar's `artifacts.client_letter` is the string `"skipped: --no-letter"` rather than null or absent

#### Scenario: An execution failure preserves partial artifacts
- **WHEN** the execution phase fails partway (e.g. a per-host scan errors out) but produced enough data for a partial report
- **THEN** the sidecar is written with `summary.partial=true`, the `errors[]` array describes what failed, and `artifacts.technical_report` references whatever the report skill wrote (potentially an incomplete report)

### Requirement: Engagement records the tools-used inventory from precheck
The agent skill SHALL record the output of the precheck phase in the sidecar's `tools_used[]` array. Each entry SHALL have `name`, `version`, `status` (`ok`, `outdated`, `degraded_shadowed`, or `missing`), and where applicable a `fix` or `alternate` field.

#### Scenario: Tools-used array reflects precheck output
- **WHEN** an engagement runs on a machine where openssl is LibreSSL but brew openssl@3 is present
- **THEN** the sidecar's `tools_used[]` includes an entry for openssl with `status="degraded_shadowed"` and an `alternate` field pointing at the brew binary

### Requirement: Engagement IDs are deterministic and collision-aware
The agent skill SHALL derive `engagement_id` as `<slugified-client-name>-<YYYY-MM-DD>` (UTC). When a target sidecar filename already exists, the skill SHALL refuse to overwrite and SHALL prompt the operator for a suffix (e.g. `-2`, `-followup`, `-take2`).

#### Scenario: Engagement ID slug
- **WHEN** `client.name="Acme Corp & Sons, LLC"` on 2026-05-17 UTC
- **THEN** `engagement_id="acme-corp-and-sons-llc-2026-05-17"` (slug rules: lowercase, ampersands → "and", spaces and punctuation → hyphens, collapse multiple hyphens)

#### Scenario: Same-day collision prompts for suffix
- **WHEN** an engagement is invoked for `client.name="Acme Corp"` and `./reports/engagement-acme-corp-2026-05-17.json` already exists
- **THEN** the engagement does not overwrite; the operator is prompted for a suffix; if they enter `followup`, the new sidecar is at `./reports/engagement-acme-corp-2026-05-17-followup.json`

### Requirement: Engagement sidecar field operator_notes starts empty and is editable
The agent skill SHALL initialize `operator_notes` as an empty string in the sidecar. The spec acknowledges that the operator may edit this field by hand after the engagement to add context the automation could not capture.

#### Scenario: operator_notes starts empty
- **WHEN** the sidecar is initially written
- **THEN** `operator_notes` is `""` (empty string), not null or absent

#### Scenario: Post-engagement edits are valid
- **WHEN** an operator manually edits the sidecar JSON to add `operator_notes="Client mentioned plans to add a new VLAN by Q3; recommend re-assessment after that change"`
- **THEN** the sidecar remains valid per the schema (no other field was changed)

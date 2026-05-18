## ADDED Requirements

### Requirement: Assessment report merges per-skill envelopes into a single JSON document
The agent skill SHALL accept a map of `steps` (keyed by step name, valued with the envelope output of the corresponding network-discovery skill) and a `target` string, and produce a single JSON document combining all step outputs with a top-level summary and a recommendations array.

#### Scenario: Merge four standard chain steps
- **WHEN** the operator invokes the skill with `target="example.com"` and `steps={ step1_recon, step2_port_scan, step3_services, step4_tls }` where each value is the envelope output from the respective skill
- **THEN** the result is a single JSON object containing every input step under its key, plus a derived `summary` block and a `recommendations` array

#### Scenario: Empty steps map refused
- **WHEN** the operator invokes the skill with `steps={}`
- **THEN** the skill refuses with a validation error (nothing to merge)

### Requirement: Assessment report derives a summary from step contents
The agent skill SHALL populate a `summary` block including `fronted_by` (CDN detection), `resolved_ipv4`, `resolved_ipv6`, `primary_ip_scanned`, `open_ports`, `tls_grade`, `tls_score`, and a `notable` bullet list of 4–8 human-readable findings.

#### Scenario: CDN detection from WHOIS registrar
- **WHEN** `step1_recon.whois.registrar` matches "Cloudflare", "Akamai", or "Fastly"
- **THEN** `summary.fronted_by` is set to the matched CDN name

#### Scenario: Open ports extracted from port-scan
- **WHEN** the report includes `step2_port_scan`
- **THEN** `summary.open_ports` is the list of port numbers where `status="open"`

### Requirement: Assessment report derives recommendations from finding triggers
The agent skill SHALL emit a `recommendations` array where each entry has `id`, `severity`, `title`, `description`, and `remediation`, generated deterministically from triggers in the step data (e.g. missing HSTS header → medium recommendation "Enable HSTS"; TLS 1.0/1.1 supported → high recommendation "Disable TLS 1.0/1.1").

#### Scenario: HSTS-missing triggers medium recommendation
- **WHEN** `step3_services` includes a port-443 banner without a `strict-transport-security` header
- **THEN** `recommendations` includes a `medium`-severity entry titled "Enable HSTS" with concrete remediation steps

#### Scenario: No triggers produce a single informational recommendation
- **WHEN** no recommendation trigger matches
- **THEN** the array contains exactly one `informational` entry recommending periodic re-assessment

### Requirement: Assessment report saves to a timestamped filename in ./reports/
The agent skill SHALL write the merged JSON to `./reports/report-<sanitized-target>-<YYYY-MM-DD-HHMMZ>.json` by default. The output directory SHALL be overridable via an `output_dir` input.

#### Scenario: Default save location
- **WHEN** the skill completes for `target="example.com"` without an explicit `output_dir`
- **THEN** the file is written to `./reports/report-example.com-<YYYY-MM-DD-HHMMZ>.json`

#### Scenario: Output directory override
- **WHEN** the skill is invoked with `output_dir="./out/clients/acme/"`
- **THEN** the file is written under that directory using the same filename convention

#### Scenario: Output directory not writable
- **WHEN** the configured `output_dir` cannot be written
- **THEN** the skill emits the merged report into the agent conversation as a fallback and reports the underlying error

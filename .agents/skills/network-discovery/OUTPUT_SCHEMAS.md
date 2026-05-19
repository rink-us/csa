# Shared Output Schemas

These record shapes are produced by the network-discovery skills (`port-scanner`, `network-reconnaissance`, `service-enumerator`, `tls-analyzer`, `network-mapper`) and consumed by each other when skills are chained. A skill MUST emit JSON conforming to the relevant shape so downstream skills can ingest results without reparsing tool-native output.

All timestamps are ISO 8601 UTC (`2026-05-17T12:34:56Z`).

## `Host`

```json
{
  "ip": "192.0.2.10",
  "hostname": "host.example.com",
  "status": "up",
  "os_guess": "Linux 5.x",
  "discovered_at": "2026-05-17T12:34:56Z"
}
```

- `ip` (string, required) — IPv4 or IPv6
- `hostname` (string, nullable) — resolved or supplied
- `status` (`"up"` | `"down"` | `"unreachable"` | `"filtered"`)
- `os_guess` (string, nullable) — best-effort OS family
- `discovered_at` (string, required)

## `Port`

```json
{
  "port": 22,
  "protocol": "tcp",
  "status": "open",
  "service": { /* Service */ }
}
```

- `port` (integer, 1–65535)
- `protocol` (`"tcp"` | `"udp"`)
- `status` (`"open"` | `"closed"` | `"filtered"` | `"open|filtered"`)
- `service` (`Service`, nullable)

## `Service`

```json
{
  "name": "ssh",
  "product": "OpenSSH",
  "version": "9.6p1",
  "confidence": 0.9,
  "banner": { /* Banner */ },
  "cves": [ /* Cve */ ]
}
```

- `name` (string) — protocol name (`ssh`, `http`, `smtp`, `unknown`, `unresponsive`)
- `product` (string, nullable)
- `version` (string, nullable)
- `confidence` (number, 0.0–1.0)
- `banner` (`Banner`, nullable)
- `cves` (array of `Cve`, default `[]`)

## `Banner`

```json
{
  "raw": "SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5",
  "captured_at": "2026-05-17T12:34:56Z",
  "method": "tcp_read"
}
```

- `raw` (string) — verbatim banner bytes decoded as UTF-8 with replacement
- `captured_at` (string)
- `method` (`"tcp_read"` | `"http_head"` | `"smtp_ehlo"` | `"nmap_sv"`)

## `Certificate`

```json
{
  "subject": "CN=example.com",
  "issuer": "CN=Example CA, O=Example",
  "not_before": "2026-01-01T00:00:00Z",
  "not_after": "2027-01-01T00:00:00Z",
  "key_algorithm": "RSA",
  "key_size": 2048,
  "signature_algorithm": "sha256WithRSAEncryption",
  "san_list": ["example.com", "www.example.com"],
  "serial": "0a1b2c...",
  "is_wildcard": false,
  "is_self_signed": false,
  "weakness_flags": ["sha1_signature"]
}
```

- `weakness_flags` enum values: `expired`, `expiring_soon`, `weak_rsa`, `weak_ecc`, `sha1_signature`, `self_signed`, `hostname_mismatch`

## `Cve`

```json
{
  "id": "CVE-2024-12345",
  "severity": "high",
  "cvss": 8.1,
  "source": "nvd",
  "fetched_at": "2026-05-17T12:34:56Z",
  "confidence": "version_match"
}
```

- `severity` (`"low"` | `"medium"` | `"high"` | `"critical"`)
- `source` (`"nvd"` | `"circl"` | `"manual"`)
- `confidence` (`"version_match"` | `"banner_hint"` | `"product_only"`)

## `TopologyNode`

```json
{
  "host": { /* Host */ },
  "device_type": "linux_server",
  "device_confidence": 0.85,
  "ports": [ /* Port */ ],
  "certificates": [ /* Certificate */ ],
  "role": "host"
}
```

- `device_type` (string, free-form) — `linux_server`, `windows_workstation`, `network_device`, `iot`, `unknown`
- `role` (`"host"` | `"gateway"` | `"router"` | `"unknown"`)

## `TopologyEdge`

```json
{
  "source": "192.0.2.10",
  "target": "192.0.2.1",
  "type": "default_route"
}
```

- `type` (`"default_route"` | `"same_subnet"` | `"observed_hop"`)

## `EngagementRecord`

The audit-attesting sidecar artifact produced by the `network-assessment-engagement` capability. Read alone (without the linked technical report or client letter), it answers: who ran the engagement, when, against what scope, under what authorization, which phases completed, what tools were used, and where each produced file lives.

```json
{
  "engagement_id": "acme-corp-2026-05-17",
  "client": {
    "name": "Acme Corp",
    "primary_contact": "jane@acme.example",
    "site_address": null
  },
  "operator": {
    "name": "Kevin Doe",
    "email": "k@example.com"
  },
  "authorization": {
    "reference": "SOW-2026-04-Acme",
    "date": "2026-04-15"
  },
  "scope": {
    "ranges": ["192.168.1.0/24"],
    "exclusions": []
  },
  "playbook": "network-assessment",
  "started_at": "2026-05-17T19:42:00Z",
  "finished_at": "2026-05-17T19:48:00Z",
  "precheck_completed_at": "2026-05-17T19:42:18Z",
  "scope_confirmed_at":    "2026-05-17T19:42:55Z",
  "execution_started_at":  "2026-05-17T19:42:58Z",
  "execution_finished_at": "2026-05-17T19:47:54Z",
  "sidecar_written_at":    "2026-05-17T19:48:00Z",
  "scans": [
    {
      "scope": "192.168.1.0/24",
      "report_json":   "reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.json",
      "client_letter": "reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.txt"
    }
  ],
  "summary": {
    "hosts_discovered": 13,
    "hosts_scanned": 4,
    "total_cves": 12,
    "highest_severity": "high",
    "partial": false
  },
  "tools_used": [
    { "name": "nmap",    "version": "7.99",            "status": "ok" },
    { "name": "openssl", "version": "LibreSSL 3.3.6",  "status": "degraded_shadowed",
      "alternate": { "path": "/opt/homebrew/opt/openssl@3/bin/openssl", "version": "OpenSSL 3.6.2" },
      "fix": "export PATH=\"/opt/homebrew/opt/openssl@3/bin:$PATH\"" }
  ],
  "artifacts": {
    "technical_report": "reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.json",
    "client_letter":    "reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.txt",
    "sidecar":          "reports/engagement-acme-corp-2026-05-17.json"
  },
  "operator_notes": "",
  "errors": []
}
```

### Required vs optional fields

| Field | Required? | Notes |
| --- | --- | --- |
| `engagement_id` | required | Derived as `<slugified-client-name>-<YYYY-MM-DD>` UTC. Collisions resolved by operator-supplied suffix. |
| `client.name` | required | Free-form string. |
| `client.primary_contact` / `client.site_address` | optional | May be `null`. |
| `operator.name` | required | Free-form string. |
| `operator.email` | optional | May be `null`. |
| `authorization.reference` | required | Free-form. `"self"` is acceptable for own-infra engagements. |
| `authorization.date` | optional | ISO 8601 date or `null`. |
| `scope.ranges` | required | Non-empty array of CIDRs / ranges / IPs. |
| `scope.exclusions` | required | Array (may be empty). |
| `playbook` | required | Currently always `"network-assessment"`. |
| `started_at` / `finished_at` | required | ISO 8601 UTC. |
| `precheck_completed_at` / `scope_confirmed_at` / `execution_started_at` / `execution_finished_at` / `sidecar_written_at` | required | ISO 8601 UTC when the phase completed, OR the literal string starting with `"skipped: "` followed by a reason when the phase was deliberately bypassed. |
| `scans` | required | Array (may be empty if execution failed before producing any scan output). |
| `summary` | required | Object. `partial` flag is required and indicates whether the engagement completed all planned scans. |
| `tools_used` | required | Array of `{name, version, status, alternate?, fix?}` from the precheck phase. |
| `artifacts` | required | Object with three keys: `technical_report`, `client_letter`, `sidecar`. Each value is either a relative file path OR a `"skipped: <reason>"` string. **Null and absent are NOT valid.** |
| `operator_notes` | required | Empty string `""` at write time; intended for post-hoc operator edits. |
| `errors` | required | Array (may be empty). Same shape as the standard envelope errors. |

### Phase timestamp convention

Each of the five `*_at` phase fields holds either:
- An ISO 8601 UTC timestamp recording when the phase completed (the happy path)
- A string literal starting with `"skipped: "` followed by an operator-supplied reason (e.g. `"skipped: operator override - running from vetted golden image"`)

`null` and absent are NOT valid values — an auditor reading the sidecar must always be able to tell whether a phase happened, was deliberately bypassed, or was never reached.

### Pentest extensions (when `playbook="pentest"`)

When the engagement is a pentest (not a network-assessment), the sidecar includes these additional required fields beyond the base EngagementRecord:

```json
{
  "playbook": "pentest",
  "scope": ["10.0.0.0/24", "192.168.50.0/24"],
  "sessions": [
    {
      "session_id": "s1",
      "started_at": "2026-05-18T13:00:00Z",
      "finished_at": "2026-05-18T19:30:00Z",
      "evidence_record_ids": ["rec-...", "rec-..."]
    }
  ],
  "evidence_directory": "reports/pentest-evidence/<engagement_id>/",
  "narrative_path": "reports/<engagement_id>-narrative.md",
  "exploit_attempts_count": 12,
  "successful_exploits_count": 3,
  "mitre_attack_techniques_used": ["T1190", "T1059", "T1078"]
}
```

**Session shape.** Multi-day pentests use `sessions[]` to record each contiguous work block separately. Single-session engagements still emit a one-element `sessions[]` array for schema consistency.

**Field rules.**
- `scope` is required and non-empty; it is the operator-supplied list of CIDRs / IPs / hostnames that `exploit-correlator` re-validates targets against
- `sessions[]` is required and has at least one entry

## `PentestEvidenceRecord`

Produced by the `exploit-correlator` capability — one file per exploit attempt (including declined and refused-out-of-scope attempts). Append-only: once written, NEVER edited. Corrections happen via a new record with `correction_of` reference.

```json
{
  "record_id": "rec-2026-05-18-a3f1",
  "engagement_id": "acme-corp-2026-05-18",
  "session_id": "s1",
  "attempted_at": "2026-05-18T14:23:47Z",
  "target": {
    "ip": "10.0.0.5",
    "hostname": "intranet.acme.local",
    "port": 80,
    "service": "http"
  },
  "technique": "Authentication Bypass via SQL Injection",
  "mitre_attack_id": "T1190",
  "exploit_source": "exploitdb",
  "exploit_reference": "EDB-12345",
  "disclosure_reference": null,
  "command_run": "searchsploit -p 12345 | bash",
  "outcome": "success",
  "stdout": "<truncated to 64 KiB; if truncated: '... [truncated, original was N bytes]'>",
  "stderr": "",
  "exit_code": 0,
  "screenshot_path": "reports/pentest-evidence/acme-corp-2026-05-18/rec-2026-05-18-a3f1.png",
  "operator_notes": "Got admin session cookie; logged in as user 'svc-backup'.",
  "correction_of": null,
  "correction_reason": null,
  "refusal_acknowledgment_reason": null
}
```

### Field rules

| Field | Required? | Notes |
| --- | --- | --- |
| `record_id` | required | Unique within the engagement. Convention: `rec-<YYYY-MM-DD>-<short-hex>`. |
| `engagement_id` | required | Matches the parent engagement's `engagement_id`. |
| `session_id` | required | References `sessions[].session_id` in the parent sidecar. |
| `attempted_at` | required | ISO 8601 UTC when the attempt started. |
| `target.ip` | required | The actual target IP. Used by the in-scope re-validator. |
| `target.hostname` / `target.port` / `target.service` | optional | Populated when known. |
| `technique` | required | Short human-readable description of the exploit class. |
| `mitre_attack_id` | required | At minimum the broad technique ID (e.g. `T1190`); sub-technique IDs (e.g. `T1078.001`) preferred when available. `unknown` is acceptable for exploits without a clear ATT&CK mapping. |
| `exploit_source` | required | One of `exploitdb`, `metasploit`, `nvd_reference`, `operator_supplied`. |
| `exploit_reference` | required for db sources | The ExploitDB ID, Metasploit module path, or NVD reference URL. |
| `disclosure_reference` | required when `exploit_source="operator_supplied"` | URL to CVE coordination record, vendor advisory, or evidence of vendor notification. |
| `command_run` | required for executed attempts | Verbatim command line. Empty string for `dry_run_only`/`declined`/`refused_out_of_scope`. |
| `outcome` | required | One of `success`, `partial`, `failure`, `declined`, `refused_out_of_scope`, `dry_run_only`. |
| `stdout` / `stderr` | optional | Truncated to 64 KiB each. When truncated, suffix with `[... truncated, original was N bytes]` so readers know. |
| `exit_code` | optional | Integer when the exploit produced one. Null when not applicable. |
| `screenshot_path` | optional | Relative path to a PNG when the exploit yielded UI output. |
| `operator_notes` | required for executed attempts | Operator's observation of what happened. Free text. |
| `correction_of` | optional | Set on correction records; references the prior `record_id` being corrected. The original record stays in place — never modified. |
| `correction_reason` | required when `correction_of` is set | Brief explanation of what was wrong. |
| `refusal_acknowledgment_reason` | required when `outcome="refused_out_of_scope"` | Operator's reason for the attempt (operator error / pivot discovery / scope ambiguity). |

### Append-only rule

Evidence records are immutable once written. The file path `reports/pentest-evidence/<engagement_id>/<record_id>.json` is set at write time and the file is not modified afterward. Corrections happen as NEW records that reference the original via `correction_of` and explain the change in `correction_reason`. This preserves the audit trail — an auditor can reconstruct what the operator believed at each point in time.

### Truncation rule

Both `stdout` and `stderr` are capped at 64 KiB. When truncation happens, the field's value ends with `[... truncated, original was N bytes]` (where N is the pre-truncation byte length). Downstream readers can detect truncation by checking for this suffix. The full output may be optionally preserved as a sibling file `<record_id>.stdout.full` / `<record_id>.stderr.full` if the operator opts in (a `preserve_full_output: true` flag on the exploit-correlator invocation).

## Top-level result envelope

Every skill returns a single JSON object with at least:

```json
{
  "skill": "port-scanner",
  "target": "...",
  "started_at": "2026-05-17T12:34:00Z",
  "finished_at": "2026-05-17T12:35:42Z",
  "result": { /* skill-specific payload using shapes above */ },
  "errors": []
}
```

Errors are non-fatal occurrences (one host unreachable, one probe timed out) shaped as `{ "code": "...", "message": "...", "context": {...} }`. Fatal failures (missing binary) are reported as a single-element `errors` array with an empty `result`.

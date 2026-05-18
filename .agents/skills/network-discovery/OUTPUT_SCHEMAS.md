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
  "playbook_version": "1.0",
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
| `playbook_version` | required | Schema version, currently `"1.0"`. |
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

### Backwards compatibility

Sidecars produced before this schema version may lack the five `*_at` phase timestamp fields, the `artifacts` block, or the explicit `operator_notes` field. Such sidecars are valid pre-v1.0 records but produce a non-fatal warning when read by spec-aware tooling. v1.0+ sidecars MUST include all required fields.

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

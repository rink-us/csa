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

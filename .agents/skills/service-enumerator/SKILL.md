---
name: service-enumerator
description: Identify what services are running on already-discovered open ports — grab banners, fingerprint products and versions, and optionally correlate against CVE data. Use after port-scanner has found open ports and the user wants to know what's actually listening.
license: MIT
compatibility: Requires curl >= 7.79 and nmap >= 7.80. CVE correlation requires outbound HTTPS to nvd.nist.gov or cve.circl.lu.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Probe known-open ports with protocol-aware payloads, extract banners and version strings, and (optionally) correlate detected versions against public CVE data. Returns `Service` records that the agent can attach to the `Port` records from `port-scanner` (per [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md)).

Deliberately scoped narrower than `port-scanner` (which already does light `-sV` detection): this skill does deeper, protocol-specific probing.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target` | string | — | Single host: IPv4, IPv6, or hostname |
| `ports` | array | — | Typically the output of `port-scanner.ports` filtered to `status="open"` |
| `cve_lookup` | bool | `false` | Off by default — slow and noisy |
| `cve_source` | enum | `"circl"` | `"circl"` (cve.circl.lu, no key needed) or `"nvd"` (services.nvd.nist.gov, rate-limited without API key) |
| `custom_probes` | array | `[]` | Operator-supplied raw-protocol probes (see below) |
| `per_port_timeout_seconds` | int | `10` | Cap per port |

If `target` or `ports` is missing, refuse with a validation error.

## Per-protocol banner-grab playbook

Dispatch on the port's known protocol (from the input `Port.service.name` or, failing that, by port number).

| Protocol | Probe | Parse |
| --- | --- | --- |
| `http` (80, 8080, …) | `curl -sI --max-time 10 http://<target>:<port>/` | `Server:` header → `product`/`version`; full response → `banner.raw`; `method = "http_head"` |
| `https` (443, 8443, …) | `curl -skI --max-time 10 https://<target>:<port>/` | as `http` |
| `ssh` (22) | TCP-connect, read first 256 bytes, close | Match `^SSH-(\d\.\d)-(\S+)`; group 1 = protocol, group 2 = product/version; `method = "tcp_read"` |
| `smtp` (25, 587) | TCP-connect, read greeting, send `EHLO example.com\r\n`, read response, send `QUIT\r\n` | Greeting line `^220 \S+ ESMTP (\S+)` → product; `method = "smtp_ehlo"` |
| `ftp` (21) | TCP-connect, read greeting, close | `^220 (.*)` → banner; `method = "tcp_read"` |
| Anything else | Fall through to nmap fallback | — |

For raw TCP reads, prefer a portable invocation (e.g. `printf '' \| nc -w 5 <target> <port>` if `nc` is present; otherwise have the agent open the socket via its own tooling). Decode bytes as UTF-8 with replacement.

## nmap fallback

For ports the playbook doesn't cover, or when the playbook returns empty banner:

```sh
nmap -sV --version-intensity 7 -p <port> -oX - <target>
```

Map output to `Service` per the port-scanner rules ([SKILL.md](../port-scanner/SKILL.md)).

## CVE correlation (optional — `cve_lookup=true`)

Only attempted when `product` AND `version` are both known. Two sources:

**CIRCL (default — no API key):**

```sh
curl -s --max-time 15 "https://cve.circl.lu/api/search/<product>/<version>"
```

**NVD:**

```sh
curl -s --max-time 30 \
  "https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=cpe:2.3:a:<vendor>:<product>:<version>"
```

For each returned CVE, emit a `Cve` record (per `OUTPUT_SCHEMAS.md`) with:

- `id`, `severity` (map CVSS → low/med/high/crit: <4 / <7 / <9 / ≥9), `cvss`
- `source`: `"circl"` or `"nvd"`
- `confidence`: `"version_match"` (default for this skill — exact version match), or `"banner_hint"` if only a vendor string matched

CVE correlation is best-effort intelligence, not a vulnerability scan. Surface it as a hint in the output and avoid framing as a verdict.

## Custom probes

Operator may pass:

```json
{
  "custom_probes": [
    { "port": 9100, "payload_hex": "0d0a", "match_regex": "^.*", "label": "raw-printer-probe" }
  ]
}
```

The skill sends `payload_hex` (decoded) to the port, reads up to 1 KiB or 5 s, and emits a `Service` with the matched substring as `banner.raw`. If `match_regex` doesn't match, the service is recorded as `name="unknown"` with the raw bytes still in the banner.

## Result envelope

```json
{
  "skill": "service-enumerator",
  "target": "...",
  "started_at": "...",
  "finished_at": "...",
  "result": {
    "target": "...",
    "services": [
      { "port": 22, "protocol": "tcp", "service": { /* Service */ } },
      ...
    ]
  },
  "errors": []
}
```

## Failure paths

| Condition | Behavior |
| --- | --- |
| `curl` or `nmap` not found | Fatal `missing_binary` error |
| Port no longer open / connection refused | `service.name = "closed"`, no error |
| Per-port timeout reached | `service.name = "unresponsive"`, non-fatal error `{"code":"timeout","port":N}` |
| CVE source unreachable | Continue without CVEs; non-fatal `{"code":"cve_source_unreachable"}` |

## Example invocations

```
service-enumerator: target=scanme.nmap.org, ports=[{"port":22,"protocol":"tcp"},{"port":80,"protocol":"tcp"}]
service-enumerator: target=10.0.0.5, ports=[...], cve_lookup=true, cve_source=circl
service-enumerator: target=printer.lan, ports=[{"port":9100,"protocol":"tcp"}], custom_probes=[{"port":9100,"payload_hex":"0d0a","match_regex":"^.*"}]
```

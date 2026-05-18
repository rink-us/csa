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

Direct banner grabs are the **preferred path for all embedded admin UIs** (routers, printers, IoT controllers, ISP CPE, network devices). These devices respond to focused single-probe requests in seconds but exhaust nmap's heavy fingerprinting timeouts (see [port-scanner SKILL.md "Anti-pattern: aggressive fingerprinting against embedded devices"](../port-scanner/SKILL.md)). Always prefer the playbook below over `nmap -sV --version-intensity` against an embedded device.

Dispatch on the port's known protocol (from the input `Port.service.name` or, failing that, by port number).

| Protocol | Probe | Parse |
| --- | --- | --- |
| `http` (80, 8080, …) | `curl -sI --max-time 5 http://<target>:<port>/` | `Server:` header → `product`/`version`; full response → `banner.raw`; capture all security headers (HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy); `method = "http_head"` |
| `https` (443, 8443, …) | `curl -skI --max-time 5 https://<target>:<port>/` | as `http`; ALSO run the TLS-cert probe below in parallel |
| TLS cert (any TLS port) | `openssl s_client -connect <target>:<port> -servername <sni> -showcerts < /dev/null 2>/dev/null \| openssl x509 -noout -subject -issuer -dates` | Subject/issuer commonly reveal device vendor even when HTTP layer hides everything (e.g. `O=Verizon` ↔ Verizon Fios router). `method = "tls_cert"` |
| `ssh` (22) | `printf '' \| nc -w 3 <target> 22` | Match `^SSH-(\d\.\d)-(\S+)`; group 1 = protocol, group 2 = product/version; `method = "tcp_read"` |
| `smtp` (25, 587) | TCP-connect, read greeting, send `EHLO example.com\r\n`, read response, send `QUIT\r\n` | Greeting line `^220 \S+ ESMTP (\S+)` → product; `method = "smtp_ehlo"` |
| `ftp` (21) | TCP-connect, read greeting, close | `^220 (.*)` → banner; `method = "tcp_read"` |
| `domain` (53) | `dig +short +time=3 chaos txt version.bind @<target>` and `dig +short +time=3 chaos txt hostname.bind @<target>` | If responses come back, capture as version banner. Empty response is good (server hardened against fingerprinting) — record as `version_disclosure: "hardened"`. `method = "dns_chaos"` |
| `ipp` (631) | `curl -sI --max-time 5 http://<target>:631/` | 405 Method Not Allowed confirms IPP server. Header set may reveal CUPS version. `method = "http_head"` |
| `printer` / LPR (515) | `printf '' \| nc -w 3 <target> 515` | LPR rarely volunteers a banner — empty response is normal. Service presence on this port is the finding itself (legacy protocol, often no auth by default). `method = "tcp_read"` |
| `snmp` (161/udp) | `snmpget -v2c -c public -t 2 <target> 1.3.6.1.2.1.1.1.0` | If response received with community "public", that itself is a finding (default community = unauthenticated read). The sysDescr OID reveals the device firmware string. `method = "snmp"` |
| Anything else | Fall through to nmap fallback | — |

For raw TCP reads, prefer a portable invocation (e.g. `printf '' \| nc -w 3 <target> <port>` if `nc` is present; otherwise have the agent open the socket via its own tooling). Decode bytes as UTF-8 with replacement.

### Parallel execution

When probing multiple ports on the same host, **launch all banner grabs in parallel** (bash `&` + `wait`, or the equivalent in the calling skill). Each one is an independent network call with a short timeout; serializing them wastes wall time.

### Embedded admin UI: what to record when no version is exposed

It's common for embedded device admin UIs to deliberately omit the `Server:` header to avoid fingerprinting. The TLS cert + security-header inventory is then the most useful signal. Record:

- `tls.subject_org` and `tls.issuer_org` (often identifies vendor: `O=Verizon`, `O=Canon`, `O=Hikvision`, etc.)
- `tls.validity_window_years` (a 50-year validity window is a signal of ISP-managed CPE; a 90-day window is typical of an actively-managed cert)
- `tls.handshake_result` — `"ok"` when the chain was retrieved successfully, `"failed"` when the openssl probe could not extract a certificate. On `failed`, also set `tls.interpretation` with a short human-readable explanation (e.g. `"Either TLS is misconfigured on this port, only certain TLS versions accepted, or the printer responds with non-TLS data when probed on 443"`).
- `headers.security_headers_present` and `headers.security_headers_missing` — checklist against HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy
- `behavior.redirect_to_https` (HTTP returns 301 → HTTPS is a positive signal; HTTP serves content directly is a recommendation surface)
- `headers.last_modified_year` — when explicit `Server:` is missing, the `Last-Modified` header on static assets is a useful indirect age signal (a printer admin UI returning `Last-Modified: 2007` means the firmware likely hasn't been updated since then)

These signals don't go through CIRCL/NVD, but they DO go to the [vuln-correlator skill](../vuln-correlator/SKILL.md) which has a fallback analysis path for hosts without product+version data.

## nmap fallback

For ports the playbook doesn't cover, or when the playbook returns empty banner AND the target is NOT an embedded device:

```sh
nmap -sV --version-intensity 7 -p <port> -oX - <target>
```

Map output to `Service` per the port-scanner rules ([SKILL.md](../port-scanner/SKILL.md)). Skip this fallback for embedded devices — use `--version-intensity 5` or stick with the direct banner grab.

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

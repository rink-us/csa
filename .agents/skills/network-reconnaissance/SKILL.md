---
name: network-reconnaissance
description: Perform passive network reconnaissance against a domain or IP — DNS resolution (forward, reverse, nameservers), WHOIS lookup, and traceroute. Use as the first step in network workflows or when the user wants registration/routing context for a host before scanning.
license: MIT
compatibility: Requires dig (bind 9.16+) or drill; whois; traceroute or mtr.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Resolve names, look up registration data, and trace network paths. All operations are read-only against public infrastructure (recursive DNS, WHOIS servers, transit routers); no probe lands on the target itself except for `traceroute`.

Output uses the standard envelope from [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) and adds reconnaissance-specific shapes documented below.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `query` | string | — | Domain or IP (v4/v6) |
| `operations` | array | `["dns_forward","dns_reverse","nameservers","whois","traceroute"]` | Any subset; order = execution order |
| `timeout_seconds` | int | `30` per operation | Per-op cap |

If `query` is missing or empty, refuse with a validation error.

## Invocations

### `dns_forward`

```sh
dig +short +time=5 +tries=2 A   <query>
dig +short +time=5 +tries=2 AAAA <query>
```

Parse each non-empty line as an IP. Emit:

```json
{ "type": "dns_forward", "query": "...", "records": [{ "type": "A", "value": "1.2.3.4", "ttl": 300 }, ...] }
```

TTL comes from a second pass with `dig +noshort` if needed.

### `dns_reverse`

```sh
dig +short +time=5 +tries=2 -x <ip>
```

Empty result → `{ "type": "dns_reverse", "query": "...", "records": [] }` (not an error).

### `nameservers`

```sh
dig +short +time=5 NS <query>
```

NS records live at the zone apex. If `<query>` is a host (`scanme.nmap.org`) rather than a zone (`nmap.org`), this returns empty — which is correct, not an error. The agent SHOULD optionally re-query the parent zone when the initial NS query is empty and the user wanted "the nameservers for this host's domain."

### `whois`

```sh
whois <query>
```

Parse the response into key/value pairs. WHOIS output is unstructured and registry-specific — the skill extracts a best-effort subset:

```json
{
  "type": "whois",
  "query": "...",
  "registrar": "...",
  "registrant_org": "...",
  "creation_date": "...",
  "expiration_date": "...",
  "nameservers": ["ns1.example.com", "ns2.example.com"],
  "raw": "<full whois text>"
}
```

Field-name normalization: match case-insensitively against common keys (`Registrar:`, `Registrar Name:`, `OrgName:`, `Creation Date:`, `Created:`, `Registry Expiry Date:`, etc.). If a field can't be extracted, omit it and rely on `raw`.

### `traceroute`

Portable invocation (works on Linux and macOS):

```sh
traceroute -n -w 2 -q 1 <query>
```

- `-n` no DNS, `-w 2` 2-second per-hop wait, `-q 1` one probe per hop
- macOS `traceroute` uses UDP by default (unprivileged); Linux varies — if `traceroute` fails on Linux with permission errors, fall back to `mtr -n -c 1 -r <query>`

Parse each hop line into:

```json
{ "type": "traceroute", "query": "...", "hops": [{ "index": 1, "ip": "10.0.0.1", "rtt_ms": 1.2 }, { "index": 2, "ip": "*", "rtt_ms": null }, ...] }
```

`*` IPs indicate non-responding hops; keep them in the sequence.

## In-conversation reuse

If the same `query`/`operations` pair was run earlier in the current conversation and the agent has the result in context, reuse it instead of re-querying. This is a soft guideline — when in doubt (e.g. >5 min has passed, the user explicitly asked to refresh), re-run.

There is no on-disk cache.

## Failure paths

| Condition | Behavior |
| --- | --- |
| `dig`/`whois`/`traceroute` not found | Fatal `missing_binary` error per envelope |
| NXDOMAIN | `{ "type": "dns_forward", "records": [], "errors": [{"code": "nxdomain"}] }` |
| WHOIS rate-limited | Non-fatal error `{"code": "whois_rate_limit"}`, empty result for that op |
| Traceroute timeout / network unreachable | Non-fatal error, hops collected so far returned |

None of the above should abort other operations in the same invocation.

## Example invocations

```
network-reconnaissance: query=example.com
network-reconnaissance: query=8.8.8.8, operations=["dns_reverse","whois"]
network-reconnaissance: query=example.com, operations=["traceroute"]
```

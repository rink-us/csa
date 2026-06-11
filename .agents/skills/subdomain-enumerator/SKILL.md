---
name: subdomain-enumerator
description: Discover subdomains of a target domain using certificate transparency logs (crt.sh), DNS brute-force, zone transfer attempts, and DNS record enumeration (A, AAAA, MX, NS, TXT, CNAME). Use as a pre-scan recon step to expand the attack surface beyond the apex domain.
license: MIT
compatibility: Requires dig (bind 9.16+) or drill; curl. Optional: subfinder or amass for enhanced brute-force coverage.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Discover subdomains and DNS records associated with a target domain. All operations are read-only against public infrastructure (crt.sh API, public recursive resolvers, authoritative nameservers).

Output uses the standard envelope from [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) and adds subdomain-specific shapes documented below.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `domain` | string | — | The root domain to enumerate (e.g. `example.com`). Must be a bare domain — no protocol, no path. |
| `sources` | array | `["crt","dns_bruteforce","zone_transfer","record_types"]` | Which sources to query. Order = execution order. Subset: `crt`, `dns_bruteforce`, `zone_transfer`, `record_types`. |
| `wordlist` | string | `"top-5000"` | Wordlist for brute-force. `"top-1000"`, `"top-5000"`, `"top-10000"`, or a path to a custom wordlist file. Built-in wordlists are under `.agents/skills/subdomain-enumerator/wordlists/`. |
| `resolver` | string | `"8.8.8.8"` | DNS resolver for brute-force lookups. Defaults to Google Public DNS; can be set to an internal resolver for private domains. |
| `timeout_seconds` | int | `30` | Per-operation timeout. |
| `max_concurrent` | int | `50` | Max concurrent DNS lookups during brute-force phase. |

If `domain` is missing, empty, or contains a protocol/path separator, refuse with a validation error.

---

## Invocations

### `crt` — Certificate Transparency log query

```sh
curl -s --max-time 15 "https://crt.sh/?q=%25.<domain>&output=json"
```

Parse the JSON response. Each entry has `name_value` — split newlines within to get individual subdomains. Deduplicate by normalized lowercase.

Extract the subdomain label from the FQDN (e.g. `www`, `api`, `mail`). Exclude the bare apex (`@` or `*.` prefix matches that expire as wildcards).

Output:

```json
{
  "type": "crt",
  "domain": "example.com",
  "entries": [
    { "subdomain": "www.example.com", "label": "www", "source": "crt.sh", "fetched_at": "..." },
    { "subdomain": "api.example.com", "label": "api", "source": "crt.sh", "fetched_at": "..." }
  ],
  "total_found": 2,
  "errors": []
}
```

If crt.sh is unreachable or returns non-JSON, emit a non-fatal error and continue to the next source — do not abort.

### `dns_bruteforce` — Wordlist-driven subdomain resolution

For each word in the wordlist, query:

```sh
dig +short +time=3 +tries=1 <word>.<domain>
```

Parse non-empty lines as resolved IPs. Launch lookups in parallel with `max_concurrent` concurrency using bash backgrounding (`&` + `wait` batches). Only emit entries that resolved to at least one IP.

```json
{
  "type": "dns_bruteforce",
  "domain": "example.com",
  "entries": [
    { "subdomain": "www.example.com", "label": "www", "ips": ["93.184.216.34"], "source": "dns_bruteforce", "ttl": 300 },
    { "subdomain": "mail.example.com", "label": "mail", "ips": ["93.184.216.35"], "source": "dns_bruteforce", "ttl": 300 }
  ],
  "words_tried": 5000,
  "total_found": 2,
  "errors": []
}
```

A "no such record" (NXRSPONSE or empty output) is not an error — it's the expected case. Do not record entries for unresolvable names.

Wordlist selection:
- `"top-1000"` covers the most common subdomains (`www`, `mail`, `remote`, `blog`, `webmail`, `server`, `ns1`, `ns2`, `smtp`, `secure`, `vpn`, `admin`, `portal`, `cpanel`, `ftp`, `api`, `dev`, `test`, `stage`, `prod`, `backup`, `gw`, `proxy`, `dns`, `mx`, `autodiscover`, `owa`, `lyncdiscover`, `sip`)
- `"top-5000"` extends with less common but security-relevant names (`jenkins`, `confluence`, `jira`, `gitlab`, `grafana`, `prometheus`, `kibana`, `sonarqube`, `nexus`, `artifactory`, `docker`, `k8s`, `kubernetes`, `dashboard`, `swagger`, `api-docs`, `graphql`, `phpmyadmin`, `phpPgAdmin`, `adminer`, `webmin`, `usermin`, `pma`)
- `"top-10000"` extends further for thorough coverage

### `zone_transfer` — Attempt DNS zone transfer

```sh
dig +short +time=5 +tries=1 NS <domain>
```

For each returned nameserver, attempt an AXFR query:

```sh
dig +time=10 +tries=1 AXFR <domain> @<nameserver>
```

If ANY nameserver returns a non-empty AXFR response, that is a CRITICAL finding — record all entries with a `zone_transfer_available: true` flag.

```json
{
  "type": "zone_transfer",
  "domain": "example.com",
  "zone_transfer_available": false,
  "nameservers_tested": ["ns1.example.com", "ns2.example.com"],
  "entries": [],
  "errors": []
}
```

If zone transfer is available, populate `entries[]` with every record returned. An empty `entries[]` with `zone_transfer_available: false` is the normal case.

### `record_types` — Common DNS record enumeration

Query each of these record types in parallel:

```sh
dig +short +time=5 +tries=2 <domain> -t A
dig +short +time=5 +tries=2 <domain> -t AAAA
dig +short +time=5 +tries=2 <domain> -t MX
dig +short +time=5 +tries=2 <domain> -t NS
dig +short +time=5 +tries=2 <domain> -t TXT
dig +short +time=5 +tries=2 <domain> -t CNAME
dig +short +time=5 +tries=2 <domain> -t SOA
```

Return a flat list of found records per type.

```json
{
  "type": "record_types",
  "domain": "example.com",
  "records": {
    "A": [{ "value": "93.184.216.34", "ttl": 300 }],
    "AAAA": [],
    "MX": [{ "value": "10 mail.example.com.", "ttl": 300 }],
    "NS": [{ "value": "ns1.example.com.", "ttl": 300 }],
    "TXT": [{ "value": "v=spf1 include:_spf.example.com ~all", "ttl": 300 }],
    "CNAME": [],
    "SOA": [{ "value": "ns1.example.com. admin.example.com. 2026051701 7200 3600 1209600 86400", "ttl": 300 }]
  },
  "errors": []
}
```

---

## Output structure (merged envelope)

```json
{
  "skill": "subdomain-enumerator",
  "domain": "example.com",
  "started_at": "2026-06-10T12:00:00Z",
  "finished_at": "2026-06-10T12:01:30Z",
  "result": {
    "subdomains": [
      { "subdomain": "www.example.com", "label": "www", "ips": ["93.184.216.34"], "sources": ["crt.sh", "dns_bruteforce"] },
      { "subdomain": "api.example.com", "label": "api", "ips": ["93.184.216.35"], "sources": ["crt.sh"] }
    ],
    "record_types": { /* from record_types output */ },
    "zone_transfer": { "available": false, "nameservers": ["ns1.example.com"] },
    "summary": {
      "unique_subdomains": 2,
      "sources_used": ["crt", "dns_bruteforce", "zone_transfer", "record_types"],
      "zone_transfer_available": false
    }
  },
  "errors": []
}
```

---

## Failure paths

| Condition | Behavior |
| --- | --- |
| `dig`/`curl` not found | Fatal `missing_binary` error |
| crt.sh unreachable or rate-limited | Non-fatal error per source entry; continue to next source |
| DNS resolution fails for all queries | The brute-force phase returns zero entries (empty array, not an error) |
| Nameserver refuses zone transfer connection | Non-fatal; mark that nameserver as `refused` in the zone_transfer result, continue to next NS |
| One source times out | Non-fatal; continue to next source |

---

## Example invocations

```
subdomain-enumerator: domain=example.com
subdomain-enumerator: domain=example.com, sources=["crt","dns_bruteforce"], wordlist=top-1000
subdomain-enumerator: domain=intranet.acme.local, sources=["dns_bruteforce","record_types"], resolver=10.0.0.1, wordlist=top-5000
```

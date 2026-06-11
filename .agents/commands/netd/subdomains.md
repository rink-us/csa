---
name: "netd:subdomains"
description: Discover subdomains and DNS records for a domain via crt.sh, DNS brute-force, zone transfer, and record enumeration.
category: Network Discovery
tags: [network, dns, recon, subdomains]
---

Expand the attack surface by discovering subdomains of a target domain. Use before `/netd:scan` to identify additional hosts to scan.

**Input**: `/netd:subdomains <domain> [wordlist] [sources]`

- `<domain>` (required) — bare domain, no protocol or path, e.g. `example.com`
- `[wordlist]` — `top-1000`, `top-5000` (default), or `top-10000`
- `[sources]` — comma-separated subset of `crt,dns_bruteforce,zone_transfer,record_types` (default: all four)

If no domain is supplied, ask the user.

**Steps**

1. Parse `domain`, optional `wordlist`, and optional `sources`.
2. Read [.agents/skills/subdomain-enumerator/SKILL.md](../../skills/subdomain-enumerator/SKILL.md) and follow each requested source's invocation:
   - `crt` → `curl https://crt.sh/?q=%25.<domain>&output=json`
   - `dns_bruteforce` → `dig` each wordlist entry in parallel
   - `zone_transfer` → `dig AXFR` against discovered nameservers
   - `record_types` → `dig` for A, AAAA, MX, NS, TXT, CNAME, SOA
3. Deduplicate across sources by normalized subdomain.
4. Return a single envelope with the merged `subdomains[]` list and per-source results.

**Example**

```
/netd:subdomains example.com
/netd:subdomains example.com top-10000 crt,dns_bruteforce
/netd:subdomains acme.local top-5000
```

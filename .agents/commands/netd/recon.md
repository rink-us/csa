---
name: "netd:recon"
description: DNS, WHOIS, and traceroute reconnaissance against a host or domain.
category: Network Discovery
tags: [network, dns, whois, traceroute]
---

Passive reconnaissance — pulls registration and routing data without touching the target beyond traceroute.

**Input**: `/netd:recon <query> [operations]`

- `<query>` (required) — domain or IP
- `[operations]` — comma-separated subset of `dns_forward,dns_reverse,nameservers,whois,traceroute` (default: all five)

If no argument is supplied, ask the user for the query.

**Steps**

1. Parse `query` and (optionally) `operations`. If the user gave only the operations name (e.g. `/netd:recon example.com traceroute`), accept that.
2. Read [.agents/skills/network-reconnaissance/SKILL.md](.agents/skills/network-reconnaissance/SKILL.md) and follow each requested operation's documented invocation:
   - `dns_forward` → `dig +short A` and `AAAA`
   - `dns_reverse` → `dig +short -x`
   - `nameservers` → `dig +short NS` (empty result is correct for non-apex queries — note this back to the user)
   - `whois` → `whois`, parsed per the SKILL.md's field-extraction rules
   - `traceroute` → portable `traceroute -n -w 2 -q 1` (or `mtr` fallback on Linux)
3. Reuse any in-conversation result for the same `query` rather than re-querying.
4. Return a single envelope with one result entry per operation. Failures (NXDOMAIN, WHOIS rate limit, traceroute timeout) map to typed results — never abort the other operations.

**Example**

```
/netd:recon example.com
/netd:recon 8.8.8.8 dns_reverse,whois
/netd:recon scanme.nmap.org traceroute
```

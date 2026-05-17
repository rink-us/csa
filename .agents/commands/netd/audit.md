---
name: "netd:audit"
description: Full network audit chain — recon → scan → services → TLS — against a single host.
category: Network Discovery
tags: [network, audit, chain, workflow]
---

Run the four single-host skills in sequence against one target and merge the results into one report.

**Input**: `/netd:audit <target> [port-range]`

- `<target>` (required) — hostname or IP
- `[port-range]` — passed to `/netd:scan` (default `1-1024`)

If no target is supplied, ask for one. Confirm before running against a target that looks like a non-test internet host (the user owns it / it's lab infra / it's `scanme.nmap.org`).

**Steps**

1. Parse `target` and optional `port_range`.
2. **Recon** — invoke the playbook in [.agents/skills/network-reconnaissance/SKILL.md](.agents/skills/network-reconnaissance/SKILL.md) with `operations=["dns_forward","dns_reverse","whois","traceroute"]`. Keep the resolved IPv4/IPv6 addresses.
3. **Scan** — invoke [.agents/skills/port-scanner/SKILL.md](.agents/skills/port-scanner/SKILL.md) against the resolved primary address, port range as supplied or default.
4. **Services** — invoke [.agents/skills/service-enumerator/SKILL.md](.agents/skills/service-enumerator/SKILL.md) against the open ports from step 3. CVE lookup defaults off; only enable if the user explicitly asked for it earlier in the conversation.
5. **TLS** — for each port where the service is HTTPS / TLS-wrapped (443, 465, 587 STARTTLS, 8443, etc.) invoke [.agents/skills/tls-analyzer/SKILL.md](.agents/skills/tls-analyzer/SKILL.md). Skip silently if none.
6. Merge all four envelopes into one report keyed by step name, plus a `summary` block listing: resolved IPs, open-port count, service-version highlights, TLS grade (if applicable).
7. If any step fails fatally (e.g. `missing_binary`), report what's missing and continue with the remaining steps where possible.

**Example**

```
/netd:audit example.com
/netd:audit scanme.nmap.org 1-65535
/netd:audit 10.0.0.5
```

---
name: "netd:services"
description: Banner-grab and fingerprint services on already-discovered open ports.
category: Network Discovery
tags: [network, banner, fingerprint, cve]
---

Probe known-open ports with protocol-aware payloads. Use after `/netd:scan` has found open ports.

**Input**: `/netd:services <target> <ports> [cve]`

- `<target>` (required) — host the ports belong to
- `<ports>` (required) — comma-separated `port` or `port/proto` (e.g. `22,80,443` or `22/tcp,53/udp`); also accepts a JSON `Port[]` array pasted from `/netd:scan` output
- `[cve]` — pass `cve` to enable optional CVE correlation (off by default — slow + noisy)

If `target` or `ports` is missing, ask the user.

**Steps**

1. Parse `target`, `ports` (accept either CSV form or a pasted `Port[]` JSON array), and optional `cve` flag.
2. Read [.agents/skills/service-enumerator/SKILL.md](.agents/skills/service-enumerator/SKILL.md) and follow its **Per-protocol banner-grab playbook** dispatched on the port's known protocol (or port number heuristic):
   - HTTP/HTTPS → `curl -sI` / `curl -skI`
   - SSH → raw TCP read, match `^SSH-(\d\.\d)-(\S+)`
   - SMTP → greeting + `EHLO`
   - FTP → greeting read
   - Anything else → `nmap -sV --version-intensity 7` fallback
3. If `cve` is set, run the optional CVE correlation step against the SKILL.md's documented source (default CIRCL).
4. Return one envelope with a `services` array — each entry is a `Port` + `Service` record from [OUTPUT_SCHEMAS.md](.agents/skills/network-discovery/OUTPUT_SCHEMAS.md).

**Example**

```
/netd:services scanme.nmap.org 22,80
/netd:services 10.0.0.5 22/tcp,53/udp,443/tcp cve
/netd:services example.com [{"port":443,"protocol":"tcp"}]
```

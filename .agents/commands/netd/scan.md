---
name: "netd:scan"
description: Port scan a host using the port-scanner skill (nmap-based).
category: Network Discovery
tags: [network, scan, nmap]
---

TCP/UDP port scan against a single host.

**Input**: `/netd:scan <target> [port-range] [protocol] [privileged]`

- `<target>` (required) — IPv4, IPv6, or hostname
- `[port-range]` — e.g. `1-1024` (default), `80,443,8080`, `1-65535`
- `[protocol]` — `tcp` (default), `udp`, or `both`
- `[privileged]` — pass `privileged` to use `nmap -sS` (requires sudo)

If no argument is supplied, ask the user for the target. Never scan a target the user hasn't named.

**Steps**

1. Parse the user's arguments into `target`, `port_range`, `protocol`, `privileged` per the input grammar above. If `target` is missing or malformed, ask for it.
2. Read [.agents/skills/port-scanner/SKILL.md](.agents/skills/port-scanner/SKILL.md) and follow its **Invocation** and **Output normalization** sections exactly.
3. Execute the documented `nmap` command with the parsed arguments. Default to the unprivileged `-sT` variant unless `privileged` was passed.
4. Normalize nmap's XML output onto the `Host` + `Port` + `Service` schema from [.agents/skills/network-discovery/OUTPUT_SCHEMAS.md](.agents/skills/network-discovery/OUTPUT_SCHEMAS.md) and present the result as JSON wrapped in the standard envelope.
5. Honor the failure paths in the SKILL.md (missing nmap → `missing_binary`; unreachable host → `Host.status = "unreachable"`; timeout → partial results + non-fatal error).

**Example**

```
/netd:scan scanme.nmap.org
/netd:scan 10.0.0.5 22,80,443,3306 tcp
/netd:scan fd00::1 1-100 both privileged
```

---
name: "netd:map"
description: Sweep a CIDR range, classify devices by type, and emit a topology graph.
category: Network Discovery
tags: [network, topology, cidr, inventory]
---

Discover live hosts across a scope and build a topology (nodes + edges). Optionally merges in prior `/netd:scan`, `/netd:services`, `/netd:tls` results.

**Input**: `/netd:map <scope> [arp] [no-classify] [allow-wide]`

- `<scope>` (required) — CIDR (`10.0.0.0/24`), range (`10.0.0.1-10.0.0.50`), single IP, or comma-separated list
- `[arp]` — use ARP discovery (`-PR`); requires sudo and same broadcast domain
- `[no-classify]` — skip the per-host fingerprint probe (faster, less detail)
- `[allow-wide]` — required to scan wider than `/16` (or IPv6 wider than `/112`)

If `scope` is missing or fails the validation rules in the SKILL.md, ask the user.

**Steps**

1. Parse `scope` and flags. Apply scope-validation rules from [.agents/skills/network-mapper/SKILL.md](.agents/skills/network-mapper/SKILL.md): reject `/0`, reject prefixes wider than `/16` unless `allow-wide` is set.
2. Run host discovery — `nmap -sn -PE` (ICMP, default) or `nmap -sn -PR` (ARP, if `arp` flag is set and sudo confirmed).
3. For each up host, run the light fingerprint probe and classify by `device_type` per the SKILL.md heuristics — unless `no-classify` is set.
4. Detect the local gateway (`route -n get default` on macOS; `ip route get 1.1.1.1` on Linux) and emit `default_route` edges; emit `same_subnet` edges sparingly (representative, not N²).
5. If the user has prior `/netd:scan` / `/netd:services` / `/netd:tls` results in the conversation, ask whether to merge them via `correlate_with` per the SKILL.md's merge rules.
6. Emit `TopologyNode` records incrementally as hosts complete; return final envelope with `nodes`, `edges`, `statistics`, and (on request) `export.dot` / `export.csv`.

**Example**

```
/netd:map 10.0.0.0/24
/netd:map 192.168.1.0/24 arp
/netd:map 10.0.0.0/16 allow-wide no-classify
```

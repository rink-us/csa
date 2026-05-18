## Why

Network discovery, reconnaissance, and TLS posture review are recurring tasks. Today an agent has to be re-taught the right tool invocations, flag combinations, and output-normalization patterns every time these jobs come up. Packaging them as Claude Code skills under [.agents/skills/](.agents/skills/) gives the agent reusable, opinionated playbooks that match this repo's existing skill pattern (see [openspec-propose/SKILL.md](.agents/skills/openspec-propose/SKILL.md)).

## What Changes

- Five new Claude Code skills under [.agents/skills/](.agents/skills/), each a single `SKILL.md` with YAML frontmatter and prompt body
- Each skill defines: trigger conditions, required inputs, the external tools it invokes (e.g. `nmap`, `dig`, `whois`, `openssl s_client`), and a normalized JSON output schema
- A shared `OUTPUT_SCHEMAS.md` reference so capabilities that overlap (banner grabbing, service ID, host discovery) return compatible structures
- A skill-author-facing `DEPENDENCIES.md` listing required external CLIs, install commands per-platform (macOS/Linux), and which operations need elevated privileges (e.g. `nmap -sS`)

No Python package, no class hierarchy, no unit-test harness — these are markdown skill prompts that instruct the agent how to wield existing CLI tools. The current `tasks.md` framing as a Python module library has been replaced.

## Capabilities

### New Capabilities

- `port-scanner`: TCP/UDP port scanning and basic service detection on a single host
- `network-reconnaissance`: DNS (forward/reverse/nameserver), WHOIS, and traceroute against a target with result caching guidance
- `service-enumerator`: Banner grabbing, service/version fingerprinting, and optional CVE correlation against an already-discovered port list
- `tls-analyzer`: Certificate chain retrieval, weakness detection (expiry, key size, deprecated algos), and protocol/cipher inventory
- `network-mapper`: Host discovery across a CIDR scope, device-type classification, and topology export (JSON / DOT / CSV)

### Modified Capabilities

<!-- No existing capabilities are being modified for this change. -->

## Impact

- Affects: this repo's [.agents/skills/](.agents/skills/) directory — additive, no edits to existing skills or openspec workflow
- APIs: none (skills are agent-side prompts, not a runtime API)
- Dependencies: external CLIs — `nmap`, `dig`/`drill`, `whois`, `traceroute`/`mtr`, `openssl`, `curl`. Documented in `DEPENDENCIES.md`; skills must detect missing binaries and report cleanly rather than failing mid-run.
- Privilege: `nmap -sS` SYN scans, ARP sweeps, and raw-socket traceroute need root/`sudo` or capabilities — skills must prefer unprivileged variants (`nmap -sT`, ICMP traceroute) by default and only suggest privileged variants when the operator has opted in.
- Systems: none — output is text/JSON for the agent to use downstream.
- Breaking Changes: none.

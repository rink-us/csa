Purpose
- Quick, high-signal notes for an OpenCode/Claude session working in this repo. Every line is something an agent would likely miss without help.

Where the real logic lives
- Skills are not Python modules or a runtime — they are prompt-driven Claude Code files under `.agents/skills/` (each `SKILL.md` is authoritative).
- Slash-command wrappers live under `.agents/commands/netd/`. Edit those files (not a mirrored `.claude/` copy) to change command behaviour.
- Shared schemas: `.agents/skills/network-discovery/OUTPUT_SCHEMAS.md` — use these when chaining skills.

Entry points / exact commands
- Preferred (terse): use the slash commands exposed to Claude Code. Exact names and positional args below (arguments are positional unless otherwise noted):
  - `/netd:scan <target> [port-range] [protocol] [privileged]`
  - `/netd:recon <query> [operations]`
  - `/netd:services <target> <ports> [cve]` (ports = comma list or JSON array)
  - `/netd:tls <target>[:port] [sni]`
  - `/netd:map <scope> [arp] [no-classify] [allow-wide]`
  - `/netd:audit <target> [port-range]` (recon → scan → services → TLS; in-conversation only)
  - `/netd:report <target> [port-range] [with-letter]` (runs audit + writes JSON to `./reports/`)
  - `/netd:vuln-scan <scope> [port-set] [scan-all] [no-letter]`
  - `/netd:engagement <scope> client=<name>` (full engagement workflow; writes sidecar)
  - `/netd:precheck` (verifies required CLIs installed)

Fallback / explicit invocation
- If the slash command is ambiguous or unavailable, name the skill explicitly in plain language, e.g. `Use the port-scanner skill on scanme.nmap.org, ports 1-1024.` — this skips skill-selection and goes straight to the SKILL.md.

Where outputs land
- Reports saved by `/netd:report` and engagement flows go to `./reports/`.
- Filename convention: `reports/report-<target>-<YYYY-MM-DD-HHMMZ>.json` (machine) and `.txt` (client letter). Engagement sidecar: `engagement-<client-slug>-<date>.json`.
- Reports may contain sensitive data. The repo does not gitignore `reports/` by default — handle accordingly.

Critical behavioural details an agent would miss
- These SKILL.md files assume specific external CLIs. If a required binary is missing, skills emit a `missing_binary` error; do not try to emulate or implement fallback behaviour locally.
- `service-enumerator` has `cve_lookup=false` by default (it's slow/noisy). CIRCL is the default CVE source (no key); NVD is rate-limited and requires API/key handling.
- Privileged scans: `port-scanner.privileged=true` triggers `sudo nmap -sS` (SYN scan). Always confirm operator intent and sudo availability before suggesting privileged scans — higher detection footprint.
- macOS shipped LibreSSL is not full-featured for TLS tasks. `tls-analyzer` prefers OpenSSL 1.1.1+ (install `openssl@3` via Homebrew) — otherwise CT/OCSP steps degrade gracefully but are limited.
- For many-host scans (LAN / /24), prefer the `/netd:vuln-scan` pattern: ARP discovery → classify → one nmap process per host in parallel. Do NOT pass many hosts into a single nmap process (it serializes timeouts and is very slow for silent IoT devices).
- `port_set` semantics (used by port-scanner and slash commands): `top-100` = nmap `-F`; `top-1024` = `-p 1-1024` (default); `iot-curated` = specific admin ports; `full` = `1-65535`. Use `port_range` to override precisely.
- Timing profiles: `lan` = aggressive (`-T4 --max-rtt-timeout 200ms ...`), `wan` = conservative (`-T3`), `stealth` = `-T2` (low-footprint). `--host-timeout` is always appended.
- Embedded devices: avoid heavy nmap fingerprinting (`--version-intensity 9 -sC`) — use `service-enumerator`'s focused banner grabs (`curl`, `openssl s_client`, `nc`, `dig`) which return in seconds and are more reliable.
- When probing multiple ports on the same host, launch banner grabs in parallel (short per-port timeouts) to avoid wasted wall time.
- `netd:audit` and `/netd:report` will confirm ownership/authorization before scanning internet hosts; always verify targets that look like public internet hosts. Use `scanme.nmap.org` for harmless testing only.

Editing guidance
- To change or add a slash command, edit the canonical file in `.agents/commands/netd/`. Those files drive what's exposed in Claude Code sessions.
- To change behaviour of a skill, edit its `SKILL.md` under `.agents/skills/` (the skill text is the implementation/prompt). There is no hidden runtime to update.

Where to look next (authoritative sources)
- `.agents/skills/*/SKILL.md` — definitive behaviour and inputs/outputs
- `.agents/commands/netd/*.md` — slash command wiring and step sequences (e.g. `netd:audit`)
- `.agents/skills/network-discovery/DEPENDENCIES.md` — install hints for required CLIs
- `.agents/skills/network-discovery/OUTPUT_SCHEMAS.md` — data schemas used when chaining skills
- `playbooks/` — human-reviewed procedures and engagement expectations (authorization, sidecar contents)

When to ask the user a question (examples)
- Target ownership/authorization for any non-lab/public-host scan (confirm before proceeding).
- Whether to enable privileged scans (`privileged=true`) or CVE lookup (`cve_lookup=true`) — both change risk/noise/time.

Keep it short
- If it's not in this file or the SKILL.md in `.agents/skills/`, assume it's not repo-contracted behaviour. Ask instead of guessing.

# csa (Cyber Security Assessor)

## Skills

### network-discovery bundle

Five Claude Code skills for network reconnaissance and posture review. Each `SKILL.md` is self-contained and chains via the shared schemas in [.agents/skills/network-discovery/OUTPUT_SCHEMAS.md](.agents/skills/network-discovery/OUTPUT_SCHEMAS.md).

- [port-scanner](.agents/skills/port-scanner/SKILL.md) — TCP/UDP port scanning with service detection (nmap)
- [network-reconnaissance](.agents/skills/network-reconnaissance/SKILL.md) — DNS, WHOIS, traceroute (dig, whois, traceroute)
- [service-enumerator](.agents/skills/service-enumerator/SKILL.md) — Banner grabbing, version fingerprinting, optional CVE correlation (curl, nmap)
- [tls-analyzer](.agents/skills/tls-analyzer/SKILL.md) — Certificate chain, weaknesses, protocol/cipher inventory, OCSP, CT (openssl)
- [network-mapper](.agents/skills/network-mapper/SKILL.md) — CIDR sweeps, device classification, topology export (nmap)

Plus two reporting skills that consume the bundle's output:

- [assessment-report](.agents/skills/assessment-report/SKILL.md) — merges per-skill envelopes into a single technical JSON report with summary + prioritized recommendations; writes to `./reports/`
- [client-report](.agents/skills/client-report/SKILL.md) — renders a saved JSON report into an email-ready plain-text client letter

Required external CLIs and install commands are listed in [.agents/skills/network-discovery/DEPENDENCIES.md](.agents/skills/network-discovery/DEPENDENCIES.md). Generated reports land in [reports/](reports/).

## Usage

Three ways to invoke the skills, ordered from terse to explicit.

### 1. Slash commands (terse, recommended for repeat tasks)

Six commands live in [.agents/commands/netd/](.agents/commands/netd/). They're exposed to Claude Code via a top-level `.claude → .agents` symlink, so `.claude/commands/`, `.claude/skills/`, and `.claude/settings.local.json` all resolve into `.agents/`. Edit the canonical files under `.agents/commands/netd/` — the changes show up at the `/netd:*` prompts on next session. Each command is a thin wrapper that points Claude at the corresponding `SKILL.md` and passes your arguments through.

| Command | Purpose | Example |
| --- | --- | --- |
| `/netd:scan <target> [port-range] [protocol] [privileged]` | Port scan a host | `/netd:scan scanme.nmap.org 1-1024` |
| `/netd:recon <query> [operations]` | DNS + WHOIS + traceroute | `/netd:recon example.com dns_forward,whois` |
| `/netd:services <target> <ports> [cve]` | Banner-grab + fingerprint on open ports | `/netd:services scanme.nmap.org 22,80` |
| `/netd:tls <target>[:port] [sni]` | Full TLS posture review | `/netd:tls example.com` |
| `/netd:map <scope> [arp] [no-classify] [allow-wide]` | CIDR sweep + topology | `/netd:map 10.0.0.0/24` |
| `/netd:audit <target> [port-range]` | Full chain: recon → scan → services → TLS (in-conversation only, not saved) | `/netd:audit example.com` |
| `/netd:report <target> [port-range] [with-letter]` | Run the audit chain and save a timestamped JSON report to `./reports/` | `/netd:report example.com 1-1024 with-letter` |
| `/netd:letter <json-path-or-target>` | Render a saved JSON report into an email-ready client letter | `/netd:letter example.com` |
| `/netd:precheck` | Verify required CLIs are installed and adequate | `/netd:precheck` |

Arguments after the command name are positional. Unknown flags are surfaced as a clarification request rather than silently ignored. Each command refuses to run if the required target argument is missing.

### 2. Explicit-by-name fallback (always works)

If you forget a slash command's flag syntax — or you're in a non-Claude-Code context that doesn't load slash commands — just name the skill in plain English:

```
Use the port-scanner skill on scanme.nmap.org, ports 1-1024.
Use the network-reconnaissance skill on example.com, only dns_forward and whois.
Use the service-enumerator skill on scanme.nmap.org for ports [22, 80].
Use the tls-analyzer skill on example.com:443.
Use the network-mapper skill on 192.168.1.0/24 with discovery_method=arp.
```

Naming the skill explicitly skips Claude's description-match step — it goes straight to reading the SKILL.md and following it. Useful when there's any ambiguity about which skill should fire.

### 3. Multi-skill chain in one prompt

Describe the workflow and let Claude orchestrate. The shared schemas in [OUTPUT_SCHEMAS.md](.agents/skills/network-discovery/OUTPUT_SCHEMAS.md) mean each skill's output feeds the next without reshaping, and the two reporting skills consume the merged output as their input.

**Just the assessment, returned in conversation** (transient — nothing saved):

```
For example.com:
  - Run reconnaissance (DNS + WHOIS).
  - Resolve the primary A record and port-scan that IP, top 1024 ports.
  - Enumerate services on whatever's open.
  - Run tls-analyzer on port 443.
Return one merged JSON report keyed by step.
```

This is what `/netd:audit` automates.

**Assessment + saved technical report** (writes JSON to `./reports/`):

```
For example.com:
  - Run reconnaissance, port-scan top 1024, enumerate services, and analyze TLS on 443.
  - Then run the assessment-report skill to merge everything into a single JSON document
    with a summary block and prioritized recommendations.
  - Save it to ./reports/ using the standard filename convention.
Tell me the path when done.
```

This is what `/netd:report` automates.

**Assessment + technical report + client letter** (full deliverable pair — JSON and email-ready .txt side by side in `./reports/`):

```
For example.com:
  - Run the audit chain (recon, scan, services, TLS on 443).
  - Build the technical JSON report with summary and recommendations; save to ./reports/.
  - Then render a client-facing plain-text letter from that JSON using the client-report skill;
    save the .txt alongside the JSON.
Return both paths.
```

This is what `/netd:report <target> with-letter` automates.

Use a natural-language chain when you want a non-default mix — e.g. recon-only across a domain list, TLS-only against a Subject Alternative Name set, or re-running just `/netd:letter` against a previous report after editing the recommendations by hand.

### Reports

Generated reports are saved under [reports/](reports/) with the convention:

```
reports/report-<target>-<YYYY-MM-DD-HHMMZ>.json   # technical, machine-readable
reports/report-<target>-<YYYY-MM-DD-HHMMZ>.txt    # client letter, plain text
```

`<target>` is the canonical hostname/IP lowercased, with `:` replaced by `_`. The timestamp is the `chain_finished_at` from the assessment, UTC, minutes precision, sortable. The `.txt` is generated by `/netd:letter` from the matching `.json`, so the basenames stay paired.

Reports may contain sensitive findings — `reports/` is not gitignored by default but consider doing so for client-bearing assessments.

### Notes

- All skills are unauthenticated and unprivileged by default. The `-sS` SYN scan and ARP discovery require `sudo` and are opt-in.
- `scanme.nmap.org` is the canonical public target for testing — the operators authorize scans. Don't substitute it with arbitrary internet hosts.
- macOS ships LibreSSL, not modern OpenSSL — [tls-analyzer](.agents/skills/tls-analyzer/SKILL.md) detects this and degrades gracefully (CT-log step skipped). Install `openssl@3` via Homebrew for the full output.
- If a required CLI is missing, the skill emits a `missing_binary` error with an install hint pulled from [DEPENDENCIES.md](.agents/skills/network-discovery/DEPENDENCIES.md) rather than crashing.

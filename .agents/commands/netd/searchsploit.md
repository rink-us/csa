---
name: "netd:searchsploit"
description: Search ExploitDB and Metasploit for exploits matching a keyword, vendor, or platform — no CVE required.
category: Network Discovery
tags: [network, exploits, exploitdb, searchsploit, recon]
---

Keyword-driven exploit search. Use when `vuln-correlator` returned no CVEs but you have a device model, vendor name, or platform to search against.

**Input**: `/netd:searchsploit <query> [platform] [type]`

- `<query>` (required) — free-text search. Model numbers (`un65mu630d`), vendors (`samsung`), platforms (`tizen`), or service names (`apache httpd 2.4.49`). Wrap multi-word queries in quotes.
- `[platform]` — filter by platform: `hardware`, `linux`, `windows`, `multiple`, `tizen`, `router`, etc.
- `[type]` — `exploit` (default), `shellcode`, `paper`, or `all`.

If no query is supplied, ask the user.

**Steps**

1. Parse `query`, optional `platform`, optional `type`.
2. Read [.agents/skills/exploitdb-searcher/SKILL.md](../../skills/exploitdb-searcher/SKILL.md) and run each source in order:
   - `exploitdb` → `searchsploit <query> --json` (local), or CSV `grep` fallback (online CSV fetch), or gitlab CSV download (last resort)
   - `metasploit` → `msfconsole -q -x "search <query>; exit"` (parse table output)
3. Apply post-filters for platform and type.
4. Deduplicate across sources by EDB-ID or module path.
5. Truncate to max_results (default 20) per source.
6. Return a single envelope with results array, metasploit_modules array, and summary.

**Failure handling**

- No single source failure is fatal — the skill degrades to the next available source.
- If ALL sources fail (no searchsploit, no CSV access, no msfconsole), report each failure and return empty results.
- Zero matching exploits is valid output — "no exploits found."

**Example**

```
/netd:searchsploit samsung un65mu630d
/netd:searchsploit "samsung smart tv" hardware
/netd:searchsploit "apache httpd 2.4.49"
/netd:searchsploit tizen exploit
/netd:searchsploit "cisco ios"
```

---
name: "netd:paths"
description: Discover web directories, files, and sensitive endpoints via wordlist-driven brute-force (gobuster/ffuf) plus targeted probes.
category: Network Discovery
tags: [network, web, directories, recon]
---

Brute-force web paths on a discovered HTTP/HTTPS service. Use after `/netd:scan` or `/netd:services` confirms a web server is running.

**Input**: `/netd:paths <url> [wordlist] [extensions]`

- `<url>` (required) — base URL with scheme and optional port, e.g. `https://example.com` or `http://10.0.0.5:8080`
- `[wordlist]` — `top-2000` (default), `top-10000`, `admin`, `backup`, or `api`
- `[extensions]` — comma-separated extensions to append, e.g. `php,bak,txt,old,sql,env`

If no URL is supplied, ask the user.

**Steps**

1. Parse `url`, optional `wordlist`, optional `extensions`.
2. Read [.agents/skills/web-path-finder/SKILL.md](../../skills/web-path-finder/SKILL.md) and run the probe phase first: curl well-known sensitive paths in parallel (robots.txt, .well-known/, .git/HEAD, .env, wp-content, actuator/health, etc.).
3. Run the brute-force phase using gobuster (or ffuf fallback) with the selected wordlist and extensions.
4. Combine probe + brute-force results into a unified `paths[]` array, sorted by status code (interesting codes first: 200, 401, 403, 301/302, then the rest).
5. Highlight sensitive findings (exposed admin panels, .git/HEAD presence, configuration leaks) in the summary.

**Example**

```
/netd:paths https://example.com
/netd:paths http://10.0.0.5:8080 top-10000 php,asp,aspx
/netd:paths https://example.com/wp-admin admin
```

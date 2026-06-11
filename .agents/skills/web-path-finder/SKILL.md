---
name: web-path-finder
description: Discover web paths, directories, files, and backups on a web server using wordlist-driven brute-force (gobuster or ffuf). Use after port-scanner or service-enumerator has confirmed HTTP/HTTPS is open, to identify hidden endpoints, admin panels, backup files, and configuration disclosures.
license: MIT
compatibility: Requires gobuster or ffuf. curl for response verification.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Brute-force web paths using wordlist-based enumeration. Designed to run after a port scan confirms HTTP/HTTPS is available. Reports discovered paths with HTTP status codes, response sizes, and content-type hints.

Output uses the standard envelope from [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) and adds path-specific shapes documented below.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target_url` | string | — | Base URL including scheme and port, e.g. `http://example.com` or `https://example.com:8443`. No trailing slash required. |
| `wordlist` | string | `"top-2000"` | Built-in wordlist selection or path to a custom file. See wordlist descriptions below. |
| `extensions` | string | `""` | Comma-separated file extensions to append during brute-force, e.g. `"php,asp,aspx,do,jsp,bak,txt,old,zip,tar,gz,sql,env,json,xml,conf,cfg,ini,log,save,swp"`. When set, each word is tested with and without each extension. |
| `status_codes` | string | `"200,204,301,302,307,401,403,405,500"` | Comma-separated HTTP status codes to include in results. Filters out irrelevant codes (404, etc.). |
| `follow_redirects` | bool | `false` | If `true`, follow redirects to their destination to detect redirect-chain endpoints. |
| `recursive` | bool | `false` | If `true`, recurse into discovered directories up to a depth of 2. |
| `rate_limit_ms` | int | `200` | Milliseconds between requests. Lower values are faster but more intrusive. 0 = max speed (no delay). |
| `user_agent` | string | `"Mozilla/5.0 (compatible; OpenCode Pentest; +https://opencode.ai)"` | User-Agent header sent with each request. |
| `timeout_seconds` | int | `60` | Max total runtime. |
| `probe_common_files` | bool | `true` | If `true`, also probe well-known sensitive files directly (robots.txt, sitemap.xml, .well-known/, crossdomain.xml, etc.) before the brute-force phase. |

---

## Wordlists

Built-in wordlists are stored under `.agents/skills/web-path-finder/wordlists/`:

| Name | Size (words) | Purpose |
| --- | --- | --- |
| `top-2000` | ~2,000 | Common admin paths, CMS directories, API endpoints. Covers the most frequently useful paths for a quick scan. |
| `top-10000` | ~10,000 | Comprehensive directory listing including less common paths, backup file patterns, and framework-specific endpoints. |
| `admin` | ~500 | Purely administrative paths — login pages, admin panels, control panels, management interfaces. Minimal noise. |
| `backup` | ~300 | Backup-specific patterns — `.bak`, `.old`, `.save`, `.swp`, `.sql`, `.tar.gz`, `.zip` variants of all common files. |
| `api` | ~400 | Common REST/GraphQL API endpoint patterns — `/api`, `/v1`, `/v2`, `/graphql`, `/swagger`, `/openapi.json`, `/docs`. |

---

## Probe phase (always runs first)

Before the brute-force, probe a fixed set of well-known sensitive paths. These return in seconds using `curl`:

```sh
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/robots.txt
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/sitemap.xml
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/.well-known/
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/crossdomain.xml
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/clientaccesspolicy.xml
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/.git/HEAD
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/.env
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/wp-content/
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/wp-admin/
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/README.txt
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/phpinfo.php
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/server-status
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/actuator/health
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/actuator/env
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/actuator/dump
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/console/
curl -s -o /dev/null -w "%{http_code}" --max-time 5 <target_url>/debug/
```

Each probe returns a `PathEntry`. Responses with status codes in `status_codes` are reported as findings.

---

## Brute-force invocation

### Using gobuster (preferred when available)

```sh
gobuster dir -u <target_url> -w <wordlist_path> \
  -s <status_codes> \
  -t <concurrency> \
  -r <follow_redirects> \
  -x <extensions> \
  -a <user_agent> \
  -q \
  --timeout 5s
```

Where concurrency is derived from `rate_limit_ms`: `concurrency = max(5, min(100, 1000 / rate_limit_ms))`.

### Using ffuf (fallback when gobuster unavailable)

```sh
ffuf -u <target_url>/FUZZ \
  -w <wordlist_path> \
  -fc 404 \
  -t <concurrency> \
  -H "User-Agent: <user_agent>" \
  -timeout 5
```

---

## Output — PathEntry

```json
{
  "path": "/wp-admin",
  "status_code": 301,
  "size_bytes": 232,
  "content_type": "text/html; charset=utf-8",
  "redirect_to": "https://example.com/wp-admin/",
  "source": "probe",
  "method": "GET"
}
```

Each discovered path gets one PathEntry record. The merged envelope:

```json
{
  "skill": "web-path-finder",
  "target_url": "https://example.com",
  "started_at": "2026-06-10T12:00:00Z",
  "finished_at": "2026-06-10T12:01:30Z",
  "result": {
    "probe_results": [ /* PathEntry from probe phase */ ],
    "brute_force_results": [ /* PathEntry from dirbust */ ],
    "summary": {
      "total_paths_found": 12,
      "sensitive_findings": [
        "/wp-admin/ (301 → login page)",
        "/.env (200 — possible configuration disclosure)",
        "/actuator/health (200 — Spring Boot health endpoint)"
      ],
      "wordlist_used": "top-2000",
      "words_tried": 2000,
      "elapsed_seconds": 42
    }
  },
  "errors": []
}
```

---

## Failure paths

| Condition | Behavior |
| --- | --- |
| `gobuster` AND `ffuf` both missing | Fatal `missing_binary`. Provide install hint. |
| Target unreachable (connection refused / timeout during probe) | All paths fail; return a single error entry; do not attempt brute-force against an unreachable target. |
| Target returns 4xx/5xx for every path | Still report findings (none found). This is valid output. |
| Wordlist file not found | Fatal error — cannot proceed without a wordlist. |

---

## When to use this skill

Use `web-path-finder` after `port-scanner` or `service-enumerator` confirms an HTTP/HTTPS service is available. It fills the gap left by port-scanning by finding:

- **Admin panels** — `/admin`, `/administrator`, `/wp-admin`, `/panel`, `/cpanel`
- **Backup files** — `config.php.bak`, `db.sql.old`, `.env.save`, `composer.json`
- **Configuration disclosures** — `.git/HEAD`, `.env`, `actuator/` (Spring Boot), `phpinfo.php`
- **API endpoints** — `/api`, `/v1`, `/graphql`, `/swagger`, `/openapi.json`
- **Sensitive directories** — `/backups`, `/logs`, `/tmp`, `/uploads`, `/private`

For WordPress-specific scanning use the `service-enumerator` skill's WordPress playbook.

---

## Example invocations

```
web-path-finder: target_url=https://example.com, wordlist=top-2000
web-path-finder: target_url=http://10.0.0.5:8080, wordlist=top-10000, extensions=php,bak,txt,old
web-path-finder: target_url=https://example.com/wp-admin, wordlist=admin, follow_redirects=true
web-path-finder: target_url=http://192.168.1.100, wordlist=backup, recursive=true
```

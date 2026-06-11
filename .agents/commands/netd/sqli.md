---
name: "netd:sqli"
description: Test web endpoints for SQL injection vulnerabilities using sqlmap. Automated detection, fingerprinting, and optional data extraction.
category: Network Discovery
tags: [network, web, sqli, sqlmap, injection]
---

Detect SQL injection vulnerabilities in web applications. HIGH-RISK — pre-flight authorization check is mandatory and cannot be bypassed. Use after `/netd:paths` or `/netd:services` has identified potentially injectable parameters.

**Input**: `/netd:sqli <url> [param] [method] [techniques]`

- `<url>` (required) — full URL including query string for GET, or base URL for POST
- `[param]` — specific parameter to test (default: test all parameters)
- `[method]` — `get` (default), `post`, or `custom` (uses a raw request file)
- `[techniques]` — desired injection techniques as a string: `BEUSTQ` (default = all). Omit letters to exclude: `T` skips time-based, `U` skips union queries.
- `[flags]` — append `extract` to attempt data extraction (requires 2nd confirmation), `level=N` (default 3), `risk=N` (default 2), `dbms=NAME` to force a database type, `tamper=SCRIPTS` comma-separated for WAF bypass.

If no URL is supplied, ask the user.

**Steps**

1. Parse `url`, optional `param`, `method`, `techniques`, and flags.
2. Read [.agents/skills/sql-injector/SKILL.md](../../skills/sql-injector/SKILL.md).
3. Run the pre-flight authorization check — display the target URL, parameters, techniques; require `yes` to proceed.
4. Run sqlmap with the appropriate flags, technique string, level, risk, and optional tamper scripts.
5. Parse sqlmap output into structured findings: vulnerable endpoints, detected DBMS, successful techniques, and extracted data summary.
6. If `extract` flag is set, require a SECOND confirmation before running the extraction phase.
7. Report findings sorted by severity (critical first).

**Failure handling**

- If sqlmap is missing, report `missing_binary` and suggest install command.
- If all parameters are not injectable, report "no vulnerabilities detected" with the list of tested parameters.
- If WAF blocks probes, attempt automatic detection and suggest tamper scripts.
- If the target is unreachable, abort with a fatal error.

**Example**

```
/netd:sqli https://example.com/page.php?id=1
/netd:sqli https://example.com/search q post EU level=5 risk=3
/netd:sqli https://example.com/products dbms=mssql tamper=space2comment,between,randomcase
/netd:sqli https://example.com/users extract
```

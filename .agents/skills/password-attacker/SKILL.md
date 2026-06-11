---
name: password-attacker
description: Test credential strength on discovered services using hydra-driven brute-force, password spraying, and default-credential checks. Use after port-scanner and service-enumerator have identified services that accept authentication (SSH, RDP, HTTP forms, SMTP, IMAP, FTP, SMB, Telnet, etc.).
license: MIT
compatibility: Requires hydra (thc-hydra). Optional: john, hashcat for offline cracking of captured hashes.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Test passwords against live services. This skill is for AUTHORIZED testing only — the operator MUST have explicit written permission to attempt authentication against every target. The skill enforces a pre-flight confirmation prompt for any run.

Output uses the standard envelope from [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) and adds credential-specific shapes documented below.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target` | string | — | Hostname or IP of the target. |
| `service` | string | — | Protocol or service name. See supported services table below. |
| `port` | int | service-default | Port number. Defaults to the standard port for the service if not provided. |
| `username` | string | `null` | Single username to test. Mutually exclusive with `userlist`. |
| `userlist` | string | `null` | Path to a username wordlist. Mutually exclusive with `username`. |
| `password` | string | `null` | Single password to test. Mutually exclusive with `passlist`. |
| `passlist` | string | `null` | Path to a password wordlist. Mutually exclusive with `password`. |
| `spray` | bool | `false` | If `true`, use password-spraying mode: try one password against all users, then move to the next password. Reduces account lockout risk. |
| `default_creds` | bool | `false` | If `true`, check the built-in default-credentials database for the detected service/product before running brute-force. |
| `rate_limit_ms` | int | `1000` | Minimum delay between authentication attempts (milliseconds). Slower = less likely to trigger rate limits or account lockouts. |
| `max_attempts` | int | `10000` | Maximum total authentication attempts. Safety limit to prevent runaway brute-force. |
| `protocol_options` | string | `""` | Additional protocol-specific options passed to hydra's `-m` flag (e.g. `"http-post-form"` path/body template). |
| `timeout_seconds` | int | `300` | Max total runtime. |
| `confirm_authorization` | bool | `true` | If `true`, requires operator confirmation before ANY attempt is made. The skill refuses to run without this safeguard unless explicitly overridden. |

---

## Supported services

| Service | hydra module | Default port | Notes |
| --- | --- | --- | --- |
| `ssh` | `ssh` | 22 | Password and key-based auth testing |
| `ftp` | `ftp` | 21 | Plaintext FTP credential test |
| `http-get` | `http-get` | 80/443 | HTTP Basic Auth |
| `http-post-form` | `http-post-form` | 80/443 | Form-based authentication. Requires `protocol_options` with the form string: `"/login:username=^USER^&password=^PASS^:F=incorrect"` |
| `https-post-form` | `https-post-form` | 443 | Same as http-post-form over TLS |
| `smb` | `smb` | 445 | SMB password test |
| `mysql` | `mysql` | 3306 | MySQL database credential test |
| `postgresql` | `postgresql` | 5432 | PostgreSQL credential test |
| `mssql` | `mssql` | 1433 | MSSQL credential test |
| `imap` | `imap` | 143/993 | IMAP login test |
| `pop3` | `pop3` | 110/995 | POP3 login test |
| `smtp` | `smtp` | 25/587 | SMTP AUTH test |
| `rdp` | `rdp` | 3389 | RDP credential test |
| `vnc` | `vnc` | 5900 | VNC password test |
| `telnet` | `telnet` | 23 | Telnet login test |
| `ldap2` / `ldap3` | `ldap2` / `ldap3` | 389/636 | LDAP simple bind test |
| `redis` | `redis` | 6379 | Redis auth test |
| `snmp` | `snmp` | 161 | SNMP community string test |

---

## Default credentials database

When `default_creds=true`, check the built-in mapping of `(product, version) → [(username, password), ...]` before brute-forcing. The database lives at `.agents/skills/password-attacker/default-creds.json`. If a match is found for the detected product and version, test those default credentials with hydra first.

Common defaults checked (non-exhaustive):
- Routers: `admin/admin`, `admin/password`, `root/admin`
- Printers: `admin/access`, `admin/1111`, `root/root`
- IP cameras: `admin/12345`, `admin/admin`, `root/vizxv`
- Database servers: `root/` (empty), `root/root`, `sa/<blank>` (MSSQL)
- IoT hubs: `admin/admin`, `admin/1234`

---

## Invocation

### Hydra brute-force (single user, password list)

```sh
hydra -l <username> -P <passlist> \
  -s <port> \
  -t <threads> \
  -w <rate_limit_ms> \
  -o /tmp/hydra-output-<target>.json \
  <target> <service> <protocol_options>
```

Thread count: `threads = max(1, min(16, 60000 / rate_limit_ms))` — slower rate means fewer threads.

### Hydra password spray (one password, many users)

```sh
hydra -L <userlist> -p <password> \
  -s <port> \
  -t <threads> \
  -w <rate_limit_ms> \
  -o /tmp/hydra-output-<target>.json \
  -f \
  <target> <service> <protocol_options>
```

`-f` stops after first valid login found (spray is trying one password against all users — if it works once, everyone with that password is exposed).

### HTTP form brute-force

The `protocol_options` input must contain the full hydra form string:

```
/login:username_field=^USER^&password_field=^PASS^:F=incorrect|Invalid|failed:C=SESSION_COOKIE
```

The format: `<path>:<POST body>:F=<failure string>:C=<cookie jar>`.

---

## Output — CredentialTest

```json
{
  "service": "ssh",
  "port": 22,
  "username": "root",
  "password": "admin123",
  "outcome": "success",
  "source": "brute_force",
  "target": "192.168.1.100",
  "tested_at": "2026-06-10T12:00:00Z"
}
```

Each attempt produces one `CredentialTest` record. The merged envelope:

```json
{
  "skill": "password-attacker",
  "target": "192.168.1.100",
  "started_at": "2026-06-10T12:00:00Z",
  "finished_at": "2026-06-10T12:05:30Z",
  "result": {
    "service": "ssh",
    "port": 22,
    "attempts": {
      "total": 5000,
      "successful": 2,
      "default_credential_hits": 0
    },
    "credentials_found": [
      { "service": "ssh", "username": "root", "password": "admin123", "outcome": "success", "source": "brute_force" },
      { "service": "ssh", "username": "backup", "password": "backup2024", "outcome": "success", "source": "brute_force" }
    ],
    "summary": {
      "service": "ssh",
      "default_creds_tested": false,
      "attempts_made": 5000,
      "accounts_compromised": 1,
      "highest_privilege": "root"
    }
  },
  "errors": []
}
```

---

## Pre-flight authorization check

This skill is HIGH-RISK. Before ANY attempt is made:

1. Show the operator the exact target, service, and credential list(s) being used.
2. Require explicit confirmation: `"This will attempt authentication against <target>:<port> (<service>). You have written authorization to test this target? (yes/no)"`
3. Only proceed on a literal `yes` response. Anything else aborts.

The confirmation check cannot be bypassed by any skill input. It is hardcoded into the process.

---

## Failure paths

| Condition | Behavior |
| --- | --- |
| `hydra` not found | Fatal `missing_binary`. Install hint: `brew install hydra` (macOS), `apt install hydra` (Debian/Ubuntu). |
| Target unreachable (connection refused) | Fatal — do not attempt brute-force against an offline host. |
| Service does not support the attempted auth method | Capture hydra's error output; return as non-fatal error. |
| Rate limit or account lockout detected (hydra reports repeated timeouts on valid credentials) | Abort the run; report the locked account as a finding ("account lockout policy detected — N failed attempts before lockout"). |
| Wordlist file not found | Fatal error. |

---

## Ethical guardrails

- Never run with `rate_limit_ms < 200` against production services — risk of service disruption from connection saturation.
- Never run with `max_attempts > 100000` without explicit operator override.
- The pre-flight confirmation is mandatory and cannot be skipped.
- Credentials found are returned in the output AND written to `./reports/credentials-<target>-<timestamp>.json` for evidence preservation. This file should be handled as high-sensitivity data.

---

## Example invocations

```
password-attacker: target=192.168.1.100, service=ssh, username=root, passlist=rockyou-top-1000
password-attacker: target=mail.example.com, service=imap, port=993, userlist=users.txt, password=Welcome2024, spray=true
password-attacker: target=10.0.0.5, service=http-post-form, port=80, username=admin, passlist=top-500, protocol_options="/login:user=^USER^&pass=^PASS^:F=Invalid password"
password-attacker: target=192.168.1.1, service=ftp, port=21, default_creds=true
```

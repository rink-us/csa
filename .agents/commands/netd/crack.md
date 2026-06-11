---
name: "netd:crack"
description: Test credential strength on a discovered service via hydra-driven brute-force, password spraying, or default-credential checks.
category: Network Discovery
tags: [network, credentials, brute-force, auth]
---

Attempt authentication against a discovered service. HIGH-RISK — the pre-flight authorization check is mandatory and cannot be bypassed.

**Input**: `/netd:crack <target> <service> [user] [pass]`

- `<target>` (required) — hostname or IP
- `<service>` (required) — protocol name from the supported list (ssh, ftp, http-post-form, smb, rdp, mysql, etc.)
- `[user]` — single username (default: `root` for SSH, `admin` for HTTP forms). Prefix with `@` (e.g. `@users.txt`) to use a file.
- `[pass]` — single password. Prefix with `@` (e.g. `@rockyou-top-1000.txt`) to use a wordlist. Default: `@top-500` built-in list.
- `[flags]` — append `spray` for password-spraying mode, `defaults` for default-credential check, `port=N` for custom port.

If no target is supplied, ask the user.

**Steps**

1. Parse `target`, `service`, optional `user`, optional `pass`, and any flags.
2. Read [.agents/skills/password-attacker/SKILL.md](../../skills/password-attacker/SKILL.md).
3. Run the pre-flight authorization check — display the exact target, service, and credential list(s); require `yes` to proceed.
4. If the `defaults` flag is set, check the built-in default credentials database first.
5. Run hydra with the determined service module, credentials, and rate limiting.
6. Report findings: which credentials succeeded, the total attempt count, and a summary of account access levels.

**Failure handling**

- If hydra is missing, report `missing_binary` and suggest the install command.
- If hydra finds no valid credentials but the target is reachable, report "no credentials found" — this is not an error.
- If rate-limiting or account lockout is detected, report it as a finding.

**Example**

```
/netd:crack 10.0.0.5 ssh root @top-1000
/netd:crack mail.example.com imap @users.txt Welcome2024 spray
/netd:crack 192.168.1.1 ftp defaults
/netd:crack example.com http-post-form admin @top-500 port=8080
```

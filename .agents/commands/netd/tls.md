---
name: "netd:tls"
description: Retrieve TLS chain, detect cert weaknesses, enumerate protocols/ciphers, score posture.
category: Network Discovery
tags: [network, tls, certificates, security]
---

Full TLS posture review of a single endpoint — chain, weaknesses, protocol/cipher support, CT logs, OCSP stapling, A+/F grade.

**Input**: `/netd:tls <target>[:port] [sni]`

- `<target>` (required) — hostname or IP; `:port` suffix optional (default `443`)
- `[sni]` — SNI to send (default = target)

If no argument is supplied, ask the user.

**Steps**

1. Parse `target` (split `host:port` if present) and optional `sni`.
2. Read [.agents/skills/tls-analyzer/SKILL.md](.agents/skills/tls-analyzer/SKILL.md) and run **all eight steps** in order:
   - Chain retrieval (`openssl s_client -showcerts`)
   - Per-cert field extraction (`openssl x509 -noout -text`)
   - Weakness detection (expiry, weak keys, deprecated algorithms, self-signed, hostname mismatch)
   - Protocol enumeration (TLSv1.0–1.3)
   - Cipher enumeration with strong/acceptable/weak classification
   - CT log SCT inspection
   - OCSP stapling probe
   - Security score + letter grade
3. If the local openssl is LibreSSL (no `-ext` flag), emit the documented `limited_openssl` non-fatal error and continue with reduced output (CT step skipped).
4. Return the standard envelope with `certificates`, `protocols`, `ciphers`, `ct`, `ocsp_stapling`, `security_score`, `grade`.

**Example**

```
/netd:tls example.com
/netd:tls mail.example.com:465
/netd:tls 10.0.0.5:8443 internal.lab
```

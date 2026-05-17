---
name: tls-analyzer
description: Retrieve a target's TLS certificate chain, extract certificate fields, detect weaknesses (expiry, weak keys, deprecated algorithms), and enumerate supported protocol versions and ciphers. Use when the user wants a TLS posture review of a host:port — e.g. before chasing a cert expiry, after a config change, or as part of a security audit.
license: MIT
compatibility: Requires openssl >= 1.1.1 (the modern OpenSSL, not LibreSSL — macOS bundles LibreSSL by default; use brew's openssl@3).
metadata:
  bundle: network-discovery
  version: "1.0"
---

Connect to a TLS endpoint, pull the certificate chain, decode the leaf cert and intermediates, evaluate weaknesses, and probe which protocol versions and cipher suites the server accepts. Outputs `Certificate` records per [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) plus a TLS-specific result payload.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target` | string | — | Hostname or IP |
| `port` | int | `443` | Any TLS port (e.g. 465 SMTPS, 993 IMAPS, 8443) |
| `sni` | string | `=target` | SNI value to send |
| `check_ct_logs` | bool | `true` | Inspect SCT extension in leaf cert |
| `check_ocsp_stapling` | bool | `true` | Probe for stapled OCSP response |
| `timeout_seconds` | int | `60` | Total skill cap |

If `target` missing or `port` out of range, refuse with a validation error.

## Step 1: chain retrieval

```sh
openssl s_client -connect <target>:<port> -servername <sni> -showcerts < /dev/null
```

Parse stdout: each `-----BEGIN CERTIFICATE-----`…`-----END CERTIFICATE-----` block is one cert in chain order (leaf first). Persist each to a temp file for the next step. If `openssl s_client` exits non-zero with "no peer certificate" or similar, return the error and stop.

## Step 2: per-cert field extraction

For each cert:

```sh
openssl x509 -noout -text -fingerprint -in <cert>
openssl x509 -noout -subject -issuer -dates -serial -in <cert>
```

Map onto the `Certificate` schema:

- `subject`, `issuer` — from `-subject`, `-issuer` (RFC 2253 form)
- `not_before`, `not_after` — convert `-dates` from `notBefore=...` / `notAfter=...` to ISO 8601
- `key_algorithm`, `key_size` — from the `-text` `Public Key Algorithm:` and `Public-Key:` lines
- `signature_algorithm` — from `Signature Algorithm:`
- `san_list` — from `X509v3 Subject Alternative Name:` (split on `, `)
- `serial` — from `-serial`
- `is_wildcard` — true if any SAN starts with `*.`
- `is_self_signed` — true if `subject == issuer` AND chain length is 1

## Step 3: weakness detection

Apply these rules to the leaf certificate; add each matched flag to `weakness_flags`:

| Flag | Condition |
| --- | --- |
| `expired` | now > `not_after` |
| `expiring_soon` | `not_after - now` < 30 days |
| `weak_rsa` | `key_algorithm == "RSA"` and `key_size` < 2048 |
| `weak_ecc` | `key_algorithm` is `EC`/`ECDSA` and `key_size` < 256 |
| `sha1_signature` | `signature_algorithm` contains `sha1` (case-insensitive) |
| `self_signed` | `is_self_signed == true` |
| `hostname_mismatch` | `target` doesn't match `CN` or any SAN (honor wildcards) |

## Step 4: protocol enumeration

Test each protocol explicitly:

```sh
openssl s_client -connect <target>:<port> -servername <sni> -tls1   < /dev/null
openssl s_client -connect <target>:<port> -servername <sni> -tls1_1 < /dev/null
openssl s_client -connect <target>:<port> -servername <sni> -tls1_2 < /dev/null
openssl s_client -connect <target>:<port> -servername <sni> -tls1_3 < /dev/null
```

Exit code 0 with a returned cert → protocol supported. Flag `TLSv1.0` and `TLSv1.1` as `deprecated: true`; mark `SSLv3` as `deprecated: true` if the openssl build supports `-ssl3` and the server accepts it (most modern builds drop SSLv3 entirely — that's fine).

## Step 5: cipher enumeration

List the local OpenSSL's cipher universe and test each against the server. Practical approach — use openssl's name groups instead of one-at-a-time (which is slow):

```sh
openssl ciphers -v 'ALL:eNULL'
```

For each cipher name from the list, attempt:

```sh
openssl s_client -connect <target>:<port> -servername <sni> -cipher <name> < /dev/null
```

Classify supported ciphers:

- `strong` — TLS 1.3 cipher OR AEAD (`AES*GCM`, `CHACHA20`) with FS (`ECDHE`/`DHE`)
- `weak` — uses `RC4`, `3DES`, `MD5`, `NULL`, anonymous (`aNULL`), or export (`EXP`)
- `acceptable` — everything else

If the cipher list is very large and time-bound, sample the strong/weak boundary cases first and document `ciphers_truncated: true` in output.

## Step 6: CT log check (`check_ct_logs=true`)

```sh
openssl x509 -noout -ext 1.3.6.1.4.1.11129.2.4.2 -in <leaf-cert>
```

If the SCT extension is present, report `ct.scts_present: true` and parse the per-log-ID entries the openssl text dump shows. If absent (or older OpenSSL that doesn't support `-ext`), report `ct.scts_present: false` with `ct.detection_method` so the user knows whether it's truly absent or untested.

## Step 7: OCSP stapling check (`check_ocsp_stapling=true`)

```sh
openssl s_client -connect <target>:<port> -servername <sni> -status < /dev/null
```

Look for `OCSP Response Status:` in the output. `successful` → `ocsp_stapling.enabled: true`; "no response sent" → `false`.

## Step 8: security score

Deterministic computation. Start at 100, subtract per weakness:

| Penalty | Points |
| --- | --- |
| `expired` | -50 |
| `expiring_soon` | -10 |
| `weak_rsa` | -25 |
| `weak_ecc` | -20 |
| `sha1_signature` | -25 |
| `self_signed` | -15 (skip if `hostname_mismatch` also triggers — same root cause) |
| `hostname_mismatch` | -30 |
| Each `weak` cipher accepted | -3 (cap at -15) |
| Each deprecated protocol accepted (TLSv1.0/1.1/SSLv3) | -10 |
| OCSP stapling absent | -2 |
| CT SCTs absent | -2 |

Score → grade: `A+` ≥95, `A` ≥85, `B` ≥70, `C` ≥55, `D` ≥40, `F` <40. Floor at 0.

## Result envelope

```json
{
  "skill": "tls-analyzer",
  "target": "...",
  "started_at": "...",
  "finished_at": "...",
  "result": {
    "target": "...",
    "port": 443,
    "sni": "...",
    "certificates": [ /* Certificate, leaf first */ ],
    "protocols": [
      { "version": "TLSv1.0", "supported": false, "deprecated": true },
      { "version": "TLSv1.2", "supported": true, "deprecated": false },
      ...
    ],
    "ciphers": [{ "name": "TLS_AES_256_GCM_SHA384", "strength": "strong" }, ...],
    "ct": { "scts_present": true, "log_ids": ["...", "..."] },
    "ocsp_stapling": { "enabled": true, "response_status": "successful" },
    "security_score": 78,
    "grade": "B"
  },
  "errors": []
}
```

## Failure paths

| Condition | Behavior |
| --- | --- |
| `openssl` not found | Fatal `missing_binary` |
| Target refuses TLS handshake | Fatal error `{"code":"tls_handshake_failed","detail":"..."}` |
| LibreSSL detected (no `-ext` support) | Non-fatal `{"code":"limited_openssl","message":"CT log inspection requires OpenSSL 1.1.1+; current is LibreSSL"}` — continue with reduced output |
| Per-cipher probe times out | Skip that cipher; don't abort |

## Example invocations

```
tls-analyzer: target=example.com
tls-analyzer: target=mail.example.com, port=465
tls-analyzer: target=internal.lab, port=8443, sni=internal.lab, check_ct_logs=false
```

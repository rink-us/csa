---
name: assessment-report
description: Merge the outputs of the network-discovery skills (network-reconnaissance, port-scanner, service-enumerator, tls-analyzer, optionally network-mapper) into a single technical assessment report — JSON document with summary, per-step envelopes, and a prioritized recommendations list. Saves to ./reports/. Use after running an audit chain when the user wants a persisted, structured artifact rather than a transient conversation answer.
license: MIT
compatibility: Consumes outputs from skills in the network-discovery bundle.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Assemble the merged technical report from one or more network-discovery skill outputs. Produces a single JSON file at `./reports/report-<target>-<timestamp>.json` using the shared record shapes in [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md).

This skill does NOT run the scans — it merges results that already exist (either freshly produced earlier in the conversation, or pasted in by the operator). For the "scan and report in one shot" workflow, use the `/netd:report` slash command which orchestrates an audit chain + this skill.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target` | string | — | Canonical hostname or IP the report describes |
| `steps` | object | — | Map of `{ step_name: envelope }` — one entry per source skill |
| `output_dir` | string | `./reports/` | Where to write the JSON file |
| `filename` | string | auto | If omitted, auto-name as `report-<target>-<YYYY-MM-DD-HHMMZ>.json` (UTC) |
| `recommendations` | array | auto | If omitted, derive from the step data per the rules below |

`steps` should use these keys when present (others are accepted but won't get a tailored summary):

- `step1_recon` — `network-reconnaissance` envelope
- `step2_port_scan` — `port-scanner` envelope
- `step3_services` — `service-enumerator` envelope
- `step4_tls` — `tls-analyzer` envelope
- `step5_network_map` — `network-mapper` envelope (optional)

## Output structure

```json
{
  "target": "...",
  "chain_started_at": "<earliest started_at from steps>",
  "chain_finished_at": "<latest finished_at from steps>",
  "openssl_used": "<absolute path + version, if step4_tls present>",
  "summary": { /* see below */ },
  "step1_recon":    { /* envelope */ },
  "step2_port_scan":{ /* envelope */ },
  "step3_services": { /* envelope */ },
  "step4_tls":      { /* envelope */ },
  "recommendations":[ /* see below */ ]
}
```

### `summary` derivation

Populate when the corresponding step is present:

| Field | Source |
| --- | --- |
| `fronted_by` | If `whois.registrar` matches Cloudflare/Akamai/Fastly OR A records fall in known CDN ranges (104.16.0.0/12, 151.101.0.0/16, etc.), set to that CDN name. Otherwise `null`. |
| `resolved_ipv4` / `resolved_ipv6` | `step1_recon.result.dns_forward.records` filtered by type |
| `primary_ip_scanned` | The IP passed to `step2_port_scan` |
| `open_ports` | `step2_port_scan.result.ports[].port` where `status="open"` |
| `tls_grade` / `tls_score` | `step4_tls.result.grade` / `.security_score` |
| `notable` | A short array of human-readable findings — flag CDN-fronting, TLS deprecation status, OCSP/CT status, HSTS-missing observation from headers, etc. Aim for 4–8 bullets. |

### `recommendations` derivation rules

For each rule that matches, append a recommendation object `{id, severity, title, description, remediation}`. Numbering is sequential (`rec-1`, `rec-2`, ...).

| Trigger | Severity | Title |
| --- | --- | --- |
| `step3_services` 443 banner has no `strict-transport-security` header | medium | Enable HSTS |
| `step1_recon.whois.dnssec == "unsigned"` | low | Enable DNSSEC |
| `step3_services` 443 banner has `access-control-allow-origin: *` | low | Review wide-open CORS on / |
| `step4_tls` shows TLS 1.0 or 1.1 supported | high | Disable TLS 1.0/1.1 |
| `step4_tls.certificates[0].weakness_flags` contains anything | high | Address certificate weakness |
| `step4_tls.security_score < 80` | high | Improve TLS posture (score below B) |
| `step4_tls.ocsp_stapling.enabled == false` | low | Enable OCSP stapling |
| `step4_tls.ct.scts_present == false` (not `null`) | low | Investigate missing CT SCTs |
| `step3_services` 443 banner lacks any of CSP / X-Frame-Options / frame-ancestors | informational | Add Content-Security-Policy and frame-ancestors |
| `summary.fronted_by != null` AND no `step5_network_map` referenced origin | informational | Origin server is not assessed |

Each recommendation MUST include all four narrative fields. Keep `description` to 2–4 sentences explaining what + why. Keep `remediation` to a concrete, copy-pasteable action (dashboard path, CLI command, or config snippet).

If the trigger table produces zero recommendations, emit a single `informational` entry titled "No findings — periodic re-assessment recommended" pointing the operator at re-running this in 90 days.

## Write behavior

1. Compute `filename` if not provided. Sanitize the `target`: lowercase, strip protocol prefixes, replace `:` with `_`, strip trailing `/`.
2. Ensure `output_dir` exists (`mkdir -p`).
3. Write the JSON pretty-printed (2-space indent). Validate with a quick `python3 -m json.tool` (or equivalent) before declaring success.
4. Return:
   ```json
   { "skill": "assessment-report", "path": "./reports/report-...-...Z.json", "size_bytes": N, "summary_grade": "A+", "recommendation_count": 5 }
   ```

## Failure paths

| Condition | Behavior |
| --- | --- |
| `steps` is empty / missing | Refuse with a validation error — there is nothing to merge |
| `target` is missing | Refuse — required for filename and summary |
| `output_dir` is not writable | Report the directory and the underlying error code |
| JSON validation fails after write | Report and do NOT delete the bad file (the operator may want to inspect) |

## Example invocation

```
assessment-report:
  target=montanacomputersolutions.com
  steps={
    "step1_recon":    <network-reconnaissance envelope>,
    "step2_port_scan":<port-scanner envelope>,
    "step3_services": <service-enumerator envelope>,
    "step4_tls":      <tls-analyzer envelope>
  }
```

Returns `{ "path": "./reports/report-montanacomputersolutions.com-2026-05-17-1825Z.json", ... }`.

For reference, the canonical example output of this skill is committed at [reports/report-montanacomputersolutions.com-2026-05-17-1825Z.json](../../../reports/report-montanacomputersolutions.com-2026-05-17-1825Z.json).

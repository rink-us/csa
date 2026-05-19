---
name: "netd:engagement"
description: Run the network-assessment playbook end-to-end against a client site. Wraps /netd:precheck + /netd:vuln-scan + /netd:letter with engagement-level scaffolding (client metadata, scope confirmation, sidecar JSON capturing the engagement context). Use when arriving at a new client site or starting a recurring assessment — this is the top-level entry point for a complete deliverable.
category: Network Discovery
tags: [network, engagement, playbook, workflow, client]
---

End-to-end execution of the [network-assessment playbook](../../../playbooks/network-assessment.md). One command produces a technical JSON report, a client-facing letter, and an engagement sidecar tying it all to the client context (authorization, operator, scope-of-work reference). Optimized for being run on-site or VPN-connected.

If this is your first time running an engagement, read [playbooks/network-assessment.md](../../../playbooks/network-assessment.md) first — it covers the manual review steps that happen AFTER this command finishes, which the slash command cannot automate.

---

**Input**: `/netd:engagement <scope> client=<name> [operator=<name>] [auth-ref=<ref>] [port-set=<set>] [scan-all] [no-letter]`

- `<scope>` (required) — CIDR (`192.168.1.0/24`), range, single IP, or comma-separated list. Same validation as `/netd:vuln-scan`.
- `client=<name>` (required) — Free-form client name. Used in the engagement sidecar and as part of the engagement_id slug.
- `operator=<name>` — Operator's name for the engagement record. If omitted, ask once at the start.
- `auth-ref=<ref>` — Authorization reference (SOW ID, ticket number, email-thread subject). If omitted, ask once — DO NOT skip silently. The reference is recorded in the sidecar.
- `port-set=<set>` — Passed through to `/netd:vuln-scan` (`top-100` default, `top-1024`, `iot-curated`, `full`).
- `scan-all` — Disable the classify-and-skip optimization. Passed through to `/netd:vuln-scan`.
- `no-letter` — Skip the client-facing `.txt`. Generates only the JSON technical report.

If `scope` or `client=` is missing, ask the user.

---

## Steps

### Phase A — Pre-flight (always, ~30 seconds)

1. Parse all arguments. If `client=` is missing, ask. If `operator=` and `auth-ref=` aren't supplied, prompt the user once for both:
   ```
   Operator name (for the engagement record): _____
   Authorization reference (SOW/ticket/email subject; "self" for own-infra): _____
   ```
   These go into the sidecar and CANNOT be skipped — they're the engagement's paper trail. The slash command SHALL refuse to start the engagement if either is absent after the prompt (no silent defaults).

2. Derive `engagement_id` = `<client-slug>-<YYYY-MM-DD>` UTC, where `client-slug` is the client name lowercased, ampersands replaced with `and`, spaces and remaining punctuation replaced with `-`, multiple consecutive `-` collapsed to one.

3. **Engagement-ID collision check.** Before proceeding, check whether `reports/engagement-<engagement_id>.json` already exists. If it does, refuse to overwrite — prompt the operator for a suffix (e.g. `-2`, `-followup`, `-take2`). The new sidecar then writes to `reports/engagement-<engagement_id>-<suffix>.json` and `engagement_id` is updated accordingly.

4. Run [/netd:precheck](precheck.md) and capture the per-tool status into `tools_used[]` for the sidecar. Record this verbatim — every entry's `name`, `version`, `status`, and (where present) `alternate` + `fix` fields. Capture `precheck_completed_at` as the ISO 8601 UTC timestamp when precheck finished. If any tool has `status=missing` (blocker), stop, report the gap, and do NOT write a sidecar (no engagement attempted). Warnings (`outdated`, `degraded_shadowed`) are noted in the sidecar but allow the engagement to proceed.

5. **Scope confirmation prompt** — show the user the parsed scope, the local interface they're scanning from, the default gateway they'll be scanning toward, and ask for confirmation:
   ```
   About to run engagement:
     Client:        <name>
     Engagement ID: <id>
     Scope:         <scope>
     Your IP:       <local IP>
     Gateway:       <default gateway>
     Authorization: <auth-ref>

   Proceed?
   ```
   Wait for explicit affirmative confirmation. This is the moment to catch a wrong-network mistake before any scan packets leave. If the operator declines or the prompt is not affirmatively answered, stop — no sidecar is written, no scan packets leave. Capture `scope_confirmed_at` as the ISO 8601 UTC timestamp when confirmation was received.

### Phase B — Execute (typically 3–5 minutes for a /24)

6. Capture `execution_started_at` as ISO 8601 UTC. Invoke [/netd:vuln-scan](vuln-scan.md) with the parsed scope + `port-set` + `scan-all` flag (if set) + `with-letter` (unless `no-letter` was passed). Capture:
   - The path of the technical JSON report it wrote
   - The path of the client letter (if generated) — OR the literal string `"skipped: --no-letter"` when `no-letter` was passed
   - Its summary block (devices_discovered, devices_scanned, total_cves, highest_severity)

   When done, capture `execution_finished_at` as ISO 8601 UTC.

   If `/netd:vuln-scan` fails fatally, still proceed to Phase C — write a sidecar with `summary.partial=true`, the error in `errors[]`, and the partial artifact paths in `artifacts` (so the audit record is preserved). Mark unfinished phases with `"skipped: execution failed - <reason>"`.

### Phase C — Sidecar (always, ~5 seconds)

7. Write the engagement sidecar to `reports/engagement-<engagement_id>.json` conforming to the `EngagementRecord` shape in [OUTPUT_SCHEMAS.md](../../skills/network-discovery/OUTPUT_SCHEMAS.md). All required fields per the schema:
   ```json
   {
     "engagement_id": "...",
     "client": { "name": "...", "primary_contact": null, "site_address": null },
     "operator": { "name": "...", "email": null },
     "authorization": { "reference": "...", "date": null },
     "scope": { "ranges": [ ... ], "exclusions": [] },
     "playbook": "network-assessment",
     "started_at": "<phase A start, ISO 8601 UTC>",
     "finished_at": "<phase C complete, ISO 8601 UTC>",
     "precheck_completed_at": "<from step 4>",
     "scope_confirmed_at":    "<from step 5>",
     "execution_started_at":  "<from step 6 start>",
     "execution_finished_at": "<from step 6 finish>",
     "sidecar_written_at":    "<now>",
     "scans": [
       { "scope": "...", "report_json": "reports/report-...-vuln-...Z.json", "client_letter": "reports/report-...-vuln-...Z.txt" }
     ],
     "summary": { /* from vuln-scan summary, plus "partial": false (or true if execution failed) */ },
     "tools_used": [ /* verbatim from precheck */ ],
     "artifacts": {
       "technical_report": "reports/report-...-vuln-...Z.json",
       "client_letter":    "reports/report-...-vuln-...Z.txt",
       "sidecar":          "reports/engagement-<engagement_id>.json"
     },
     "operator_notes": "",
     "errors": []
   }
   ```

   **Field rules** (per the EngagementRecord schema):
   - Every `*_at` phase timestamp MUST be either an ISO 8601 UTC string OR `"skipped: <reason>"`. Never `null`, never absent.
   - `artifacts.client_letter` MUST be `"skipped: --no-letter"` (not `null`) when the operator passed `no-letter`. Same for any other deliberately-skipped artifact.
   - `artifacts.sidecar` references this file's own path.
   - `operator_notes` starts as `""`. The operator may edit it by hand later; other fields should not be hand-edited because that breaks the audit trail.

7. Final output:
   ```
   === Engagement complete ===
   Client:           Acme Corp
   Engagement ID:    acme-corp-2026-05-17
   Operator:         Kevin Doe
   Authorization:    SOW-2026-04-Acme

   Files written:
     reports/engagement-acme-corp-2026-05-17.json       (sidecar metadata)
     reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.json  (technical report)
     reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.txt   (client letter)

   Headline findings:
     - 13 devices discovered (4 scanned, 9 skipped: phones + vendor IoT)
     - 12 CVEs across 3 hosts; highest severity: high
     - Most concerning: Apache httpd 2.4.7 on 192.168.1.164 (printer admin UI, EOL since 2017)

   Next steps (manual — see playbook Phase 3):
     1. Spot-check the technical report for misclassified hosts
     2. Review and adjust the recommendations
     3. Fill in your signature in the .txt letter
     4. Send to <client primary contact>
     5. (Optional) Edit operator_notes in the sidecar with engagement observations

   Total runtime: 4 min 12 sec
   ```

---

## Failure handling

| Condition | Behavior |
| --- | --- |
| `client=` missing and not provided when asked | Refuse — engagement metadata is required |
| Authorization reference not provided | Refuse — the paper trail is the point of the sidecar |
| `/netd:precheck` reports a blocker (`missing` status) | Stop. Tell the operator what's missing. Do not write a partial engagement record. |
| Scope confirmation prompt is not affirmatively answered | Stop. The operator should be able to abort here without consequence. |
| `/netd:vuln-scan` fails partway through | Still write the sidecar, with `summary.partial=true` and the error in `errors[]`. The partial data may still be useful. |
| `reports/` not writable | Stop before scanning. Tell the operator to fix the directory. |

---

## Example invocations

```
/netd:engagement 192.168.1.0/24 client="Acme Corp"
/netd:engagement 10.0.0.0/24 client="Globex Industries" operator="Kevin Doe" auth-ref="SOW-2026-04-Globex"
/netd:engagement 192.168.1.0/24 client="My Home Lab" auth-ref="self" scan-all
/netd:engagement 10.0.0.0/24,10.0.1.0/24 client="Initech" port-set=top-1024
```

---

## See also

- [playbooks/network-assessment.md](../../../playbooks/network-assessment.md) — the full procedure document including the manual review steps this command can't automate
- [/netd:vuln-scan](vuln-scan.md) — what this command wraps in Phase B
- [/netd:letter](letter.md) — for regenerating the client letter after editing the technical report

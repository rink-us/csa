---
name: "netd:report"
description: Run the full assessment chain against a target and save a persisted technical report (JSON) to ./reports/. Equivalent to /netd:audit + assessment-report skill in one shot. Optionally chains into /netd:letter to also generate a client-facing text report.
category: Network Discovery
tags: [network, report, workflow, audit]
---

End-to-end "scan it and write a saved report" workflow. Produces a timestamped JSON file under `./reports/` that you can keep, diff against future runs, or hand to the `/netd:letter` command for a client-ready letter.

**Input**: `/netd:report <target> [port-range] [with-letter]`

- `<target>` (required) — hostname or IP
- `[port-range]` — passed to the port scan (default `1-1024`)
- `[with-letter]` — pass `with-letter` to also generate the client-facing `.txt` letter alongside the JSON

If no target is supplied, ask for one.

**Steps**

1. Parse `target`, optional `port_range`, and the `with-letter` flag.
2. Ensure `./reports/` exists (`mkdir -p reports`).
3. Run the audit chain — same orchestration as [/netd:audit](audit.md):
   - **Recon** via [.agents/skills/network-reconnaissance/SKILL.md](../../skills/network-reconnaissance/SKILL.md) (operations: `dns_forward`, `dns_reverse`, `whois`, `traceroute`)
   - **Port-scan** via [.agents/skills/port-scanner/SKILL.md](../../skills/port-scanner/SKILL.md) against the resolved primary IPv4, port range as supplied
   - **Service-enumerate** via [.agents/skills/service-enumerator/SKILL.md](../../skills/service-enumerator/SKILL.md) on the open ports
   - **TLS-analyze** via [.agents/skills/tls-analyzer/SKILL.md](../../skills/tls-analyzer/SKILL.md) on TLS-bearing ports (skip silently if none)
   - Capture each step's envelope; do NOT lose any per-step errors.
4. Invoke [.agents/skills/assessment-report/SKILL.md](../../skills/assessment-report/SKILL.md) with:
   ```
   target=<target>
   steps={ step1_recon, step2_port_scan, step3_services, step4_tls }
   output_dir=./reports/
   ```
   The skill derives the filename `report-<target>-<YYYY-MM-DD-HHMMZ>.json`, generates the `summary` block and the `recommendations` array per its trigger rules, and writes the file.
5. **If `with-letter` was passed**: chain into [/netd:letter](letter.md) with the JSON path just written. This produces the matching `.txt` alongside.
6. Report back to the user:
   - The full path of the JSON (and the `.txt` if applicable)
   - The TLS grade and recommendation count from the summary
   - A one-line headline of the highest-severity finding (if any)

**Failure handling**

- If a step's required binary is missing, the assessment-report still gets written — but the affected step's envelope contains the `missing_binary` error. The report's `summary.notable` should call out the incomplete coverage.
- If the recon step can't resolve the target, port-scan and TLS steps are skipped; the report is still written with just the recon envelope plus a top-level `errors` entry explaining the chain was aborted.
- `./reports/` not writable: the chain still runs; the merged report is emitted into the conversation instead of saved, with a clear note that the operator can re-run after fixing the directory.

**Example**

```
/netd:report example.com
/netd:report scanme.nmap.org 1-65535
/netd:report montanacomputersolutions.com 1-1024 with-letter
```

**Output**

```
Saved: ./reports/report-example.com-2026-05-17-1830Z.json
TLS:   A+ (score 100)
Recommendations: 4 (1 medium, 2 low, 1 informational)
Highest finding: Enable HSTS (medium)

(with-letter): also wrote ./reports/report-example.com-2026-05-17-1830Z.txt
```

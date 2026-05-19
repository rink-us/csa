# Network Assessment Playbook

A repeatable end-to-end procedure for assessing a client's local network: discover all hosts, scan for open ports and vulnerabilities, produce a technical report, and deliver a client-facing letter.

**Audience:** Operator performing the assessment (you or anyone on your team).
**Each execution of this playbook is called an *engagement*.**
**Typical wall time:** 5–15 minutes for a typical /24, plus 15–30 minutes of manual review before delivering to the client.

---

## When to use this playbook

Use it for:
- A first-time security baseline at a new client site (or new portion of an existing client's network)
- A recurring quarterly/annual posture check against a network you already know
- A focused re-assessment after a configuration change (new router, network expansion, IoT device addition)

Do NOT use it for:
- Penetration tests with exploitation. This playbook is reconnaissance, identification, and CVE correlation. It does not attempt to exploit findings — that's a different engagement type with different authorization requirements.
- Wide-area scans of arbitrary internet ranges. Scope is bounded to a network the client owns and has authorized you to scan.
- Compliance audits requiring credentialed scanning (PCI DSS, HIPAA technical safeguards). Those need authenticated scans this playbook doesn't perform.

---

## Phase 0 — Pre-engagement (before arriving on-site or starting remote)

These steps happen before you connect to the client's network.

### 0.1 Authorization in writing

Confirm you have written authorization to perform the scan. At minimum:
- The CIDR ranges or specific hosts you're permitted to scan
- Any exclusions (production systems that should not be touched, third-party tenants on the same network)
- A point of contact at the client who is reachable during the engagement
- A start and end date or window

Don't proceed without this. If it's not in writing, get an email confirmation first.

### 0.2 Capture the engagement metadata

Note for the engagement sidecar file:
- Client name and primary contact
- Authorization reference (SOW number, email thread, ticket ID)
- Scope ranges + exclusions
- Your name as the operator
- Planned start/end dates

The `/netd:engagement` slash command will prompt for these and write them into a sidecar JSON. If you prefer to fill them in by hand, the file format is documented under "Sidecar engagement file" below.

### 0.3 Verify your tooling

Run `/netd:precheck` on your laptop. Fix any blockers (missing binaries) before the engagement. Warnings (outdated dig, LibreSSL on macOS) are acceptable but worth noting in the engagement sidecar's `tools_used` field.

### 0.4 Refresh CVE snapshot for air-gapped engagements

If the upcoming engagement is at a site with restrictive network egress (no outbound HTTPS allowed during the assessment window, air-gapped client environment, unreliable cellular hotspot), refresh the local CVE snapshot before traveling:

```
/netd:cve-snapshot
```

Initial ingest takes ~30 minutes and uses ~2 GB disk. Weekly re-runs are quick (only the recent + modified delta feeds, < 1 minute). The snapshot lets `vuln-correlator` work entirely offline via `cve_source=offline` — engagements can then be run with no outbound network calls to CIRCL or NVD. See the new "When to use offline CVE mode" section below for the trade-off discussion.

If you're running from a vetted golden image where dependencies are guaranteed (e.g. a pre-configured engagement laptop you maintain), you can ask `/netd:engagement` to skip the precheck phase — but the sidecar will record this explicitly as `"precheck_completed_at": "skipped: <your reason>"` rather than silently bypassing it. The same `"skipped: <reason>"` convention applies to any required phase you legitimately need to bypass; see the "Skipping a phase" subsection under "Sidecar engagement file" below for the full convention.

---

## Phase 1 — Pre-scan (on-site or VPN-connected to client network)

### 1.1 Connect to the right network

If on-site: connect to the network the scope describes. If multiple SSIDs, use the one the client identifies as in-scope. Confirm your DHCP-assigned IP falls within the authorized scope.

If remote: connect to the client's VPN or jump host. Confirm `route -n get default` (macOS) / `ip route` (Linux) shows you on the right interface.

### 1.2 Verify scope before scanning

Run `route -n get default` and `ifconfig <iface>` and confirm:
- Your local IP is inside the authorized scope
- The default gateway IP matches what the client described

If anything is unexpected, stop and confirm with the client point of contact before scanning. Scanning the wrong network is the worst-case error this playbook can produce.

### 1.3 Identify high-value targets to call attention to in the report

Ask the client (or note from prior knowledge):
- Which devices, if any, hold sensitive data (file server, database, backup appliance)
- Which devices, if any, are production / customer-facing and must not be aggressively scanned
- Which devices are owned by third parties (ISP-managed router, leased printer, MSP-managed appliance)

These get flagged specially in the post-scan review.

---

## Phase 2 — Scan (executed via `/netd:engagement`)

The bulk of the technical work is automated. From the repo root:

```
/netd:engagement <scope> client=<name> [scan-all]
```

What runs automatically:

1. **Discovery** — ARP sweep of the scope, then reverse DNS for each up host. Produces a per-host inventory with IP, MAC, vendor, hostname.
2. **Classification** — each host gets a `device_class` (gateway, router, printer, server, phone, vendor_cloud_iot, etc.) based on MAC vendor + hostname signals. Default behavior skips port-scanning phones and phone-home-only IoT devices; pass `scan-all` to override.
3. **Parallel port scans** — one nmap process per host worth scanning, all launched simultaneously. Top 100 ports + service version detection. Capped at 2 minutes per host.
4. **Parallel CVE analysis** — for each host with identifiable services, a sub-agent looks up known CVEs in CIRCL, derives a per-host findings narrative.
5. **Technical report** — merged JSON written to `reports/report-<scope>-vuln-<timestamp>.json`.
6. **Client letter** — plain-text email-ready letter written to `reports/report-<scope>-vuln-<timestamp>.txt`.
7. **Engagement sidecar** — JSON metadata file at `reports/engagement-<client-slug>-<date>.json` linking to the above plus the engagement context (client name, authorization reference, operator).

Total wall time: typically 3–5 minutes for a /24. Longer if the network has many active servers or if you passed `scan-all`.

---

## Phase 3 — Manual review (before delivering anything to the client)

This is the part the automation can't do. Plan 15–30 minutes.

### 3.1 Spot-check the technical report

Open `reports/report-*-vuln-<timestamp>.json` and verify:
- The `inventory` lists what you'd expect. If a known device is missing, note it (might mean MAC randomization, firewall rules, or that the device was off).
- The `summary.devices_skipped` block names devices the classifier chose not to scan. Sanity-check those classifications against your physical knowledge of the site. If a "server" was misclassified as a phone, re-run with `scan-all` or single-host `/netd:scan` against the misclassified IP.
- The CVE counts look plausible. A printer with 4 CVEs is normal; a router with 80 CVEs probably means the classifier mismatched a product version.

### 3.2 Review the recommendations

The technical report's `recommendations[]` array was derived deterministically from triggers in `assessment-report SKILL.md`. Add or adjust:
- Remove any recommendation that doesn't apply (e.g. "Enable HSTS" if the client doesn't run a public web server)
- Add any context-specific recommendation the automation didn't catch (physical access concerns, weak Wi-Fi password if you tested, plain-text WPA2-PSK if relevant)
- Re-prioritize if needed — sometimes a "low" technical finding is "high" for a specific client (a printer with default credentials is critical at a law firm because it stores scanned documents)

If you change anything, re-run `/netd:letter <client>` to regenerate the client-facing letter from the edited JSON.

### 3.3 Review the client letter

Open the `.txt` and read it as if you're the client:
- Does the BOTTOM LINE accurately summarize the engagement? Adjust if the auto-generated framing is too rosy or too dramatic.
- Are the recommendations sorted by what THIS client cares about?
- Do the "What we did not check" gaps match reality?
- Fill in or delete the `[Your Name] / [Your Title] / [Contact Information]` placeholder. If your mail client adds a signature, delete the placeholder.

---

## Phase 4 — Deliverable handoff

### 4.1 Send the client letter

Copy the contents of the `.txt` into a fresh email to the client primary contact. Attach the JSON report only if the client has an IT contact who'll read it. Otherwise, mention that the technical detail is available on request — most non-technical clients won't open the JSON.

### 4.2 Archive

The reports stay in `reports/` indefinitely by default. For client engagements, consider:
- Moving the engagement sidecar + the two report files into a per-client subdirectory: `reports/<client-slug>/<timestamp>/`
- Backing up offline if the client retention agreement requires it
- Adding `reports/` to `.gitignore` if you commit this repo to version control (reports may contain client-sensitive findings)

### 4.3 Schedule the followup

If the client engagement is recurring (quarterly, annual), note the next assessment date in your calendar / CRM. If it's one-time, note any high-severity findings that the client said they'd remediate — a follow-up confirmation scan in 30 days is good practice.

---

## Sidecar engagement file

`/netd:engagement` writes a JSON file to `reports/engagement-<client-slug>-<date>.json` conforming to the **`EngagementRecord` schema** documented in [.agents/skills/network-discovery/OUTPUT_SCHEMAS.md](../.agents/skills/network-discovery/OUTPUT_SCHEMAS.md#engagementrecord).

For the full field list, validation rules, and example payload, follow the OUTPUT_SCHEMAS.md link. A short summary:

- Required identification: `engagement_id`, `client.name`, `operator.name`, `authorization.reference`
- Required scope: `scope.ranges[]` (non-empty), `scope.exclusions[]` (may be empty)
- Required phase timestamps (always present, never null): `precheck_completed_at`, `scope_confirmed_at`, `execution_started_at`, `execution_finished_at`, `sidecar_written_at`
- Required artifacts block: `artifacts.technical_report`, `artifacts.client_letter`, `artifacts.sidecar` — each a relative file path OR a `"skipped: <reason>"` string
- Free-text post-engagement field: `operator_notes` (starts empty)

The sidecar is the source of truth for "what happened at this engagement."

### Skipping a phase

If you legitimately need to bypass a phase — for example, running from a vetted golden image where the dependency precheck is already known good, or re-running an engagement after fixing a single host where the scope-confirmation was just done — set the phase's `*_at` field to a string starting with `"skipped: "` followed by your reason. Example: `"skipped: golden image - deps verified at boot"`. This preserves the audit trail by making the deliberate bypass visible, rather than leaving a null field that looks like a bug.

Do not abuse this. An engagement with three "skipped" phases is barely an engagement; an auditor reading the sidecar will (rightly) treat it as suspect.

### Editing the sidecar after the engagement

The `operator_notes` field is the ONLY field intended for post-hoc operator edits. Add context the automation couldn't capture: physical security observations from the site visit, conversations with the client during the engagement, follow-up items, why a specific recommendation was prioritized differently than the default.

Do NOT edit other fields by hand. The audit trail depends on the automated fields being trustworthy — if a timestamp or a CVE count or a tool-version was wrong, the right fix is to re-run the engagement (with a `-2` or `-followup` suffix for the engagement ID), not to silently rewrite history. The exception is the rare case where the automation wrote a clear bug (e.g. a malformed JSON value); document that with a note in `operator_notes` explaining what you corrected and why.

---

## What this playbook does NOT cover

| Need | Use instead |
|---|---|
| Public-internet posture review of a domain | [/netd:report](../.agents/commands/netd/report.md) — same `assessment-report` skill, single-host chain |
| Just a TLS check of a webserver | [/netd:tls](../.agents/commands/netd/tls.md) |
| Quick port-scan of one host without the engagement scaffolding | [/netd:scan](../.agents/commands/netd/scan.md) |
| Penetration testing with exploitation | Out of scope. Different engagement, different authorization. |
| Authenticated vulnerability scanning | Out of scope for v1. Future playbook. |
| Web application security review | Out of scope. Different toolset (Burp, ZAP). |

---

## When to use offline CVE mode

`vuln-correlator` supports three CVE sources: `circl` (default), `nvd`, and `offline`. The default works for most engagements — outbound HTTPS to `cve.circl.lu` is usually fine. Switch to `offline` when any of these apply:

- **Air-gapped client environment.** No outbound internet from the assessment laptop. CIRCL and NVD are unreachable by design. Offline mode lets the engagement still produce CVE findings.
- **Restrictive client egress.** The engagement contract or client policy forbids outbound HTTPS during the assessment window — typically to keep the client's defenders from triaging your traffic as suspicious.
- **Unreliable field connectivity.** Cellular hotspots and customer guest Wi-Fi often throttle or block NVD's feed endpoints in ways that aren't visible until you're mid-engagement. Offline mode turns "engagement failed at the CVE phase" into "engagement degraded to slightly older data."
- **Engagement reproducibility matters.** Pin `snapshot_version` to a specific snapshot date and a re-run weeks later produces the same CVE findings. Useful for compliance follow-ups and incident timeline reconstruction.

### The trade-off

Offline data is as fresh as your last `/netd:cve-snapshot` run. If you ran it last week, you'll miss CVEs disclosed in the past 7 days. `vuln-correlator` emits a `snapshot_stale` warning when the snapshot is older than 30 days (configurable), but it doesn't refuse — better stale CVE data than no CVE data.

For high-stakes engagements where missing a recent CVE matters more than network purity, run online (CIRCL/NVD). For everything else, offline is comparable in fidelity and dramatically more reliable.

### How to use it during an engagement

The `/netd:engagement` command will eventually accept `cve-source=offline` as a passthrough; until then, the manual path is:

1. Ensure a fresh snapshot exists: `/netd:cve-snapshot` (skip if you ran it in the last week)
2. Run the engagement normally with `/netd:engagement <scope> client="..."` — but invoke `vuln-correlator` separately with `cve_source=offline` if you need to override the source mid-chain

The sidecar's `tools_used` block will include the snapshot version that was active, so the engagement record stays auditable.


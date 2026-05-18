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

`/netd:engagement` writes a JSON file to `reports/engagement-<client-slug>-<date>.json` with this structure:

```json
{
  "engagement_id": "acme-corp-2026-05-17",
  "client": {
    "name": "Acme Corp",
    "primary_contact": "jane@acme.example",
    "site_address": "(optional)"
  },
  "operator": {
    "name": "Kevin Doe",
    "email": "k@montanacomputersolutions.com"
  },
  "scope": {
    "ranges": ["192.168.1.0/24"],
    "exclusions": [],
    "authorization_reference": "SOW-2026-04-Acme",
    "authorization_date": "2026-04-15"
  },
  "playbook": "network-assessment",
  "playbook_version": "1.0",
  "started_at": "2026-05-17T19:42:00Z",
  "finished_at": "2026-05-17T19:48:00Z",
  "scans": [
    {
      "scope": "192.168.1.0/24",
      "report_json": "reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.json",
      "client_letter": "reports/report-192.168.1.0_24-vuln-2026-05-17-1942Z.txt"
    }
  ],
  "summary": {
    "hosts_discovered": 13,
    "hosts_scanned": 4,
    "total_cves": 12,
    "highest_severity": "high"
  },
  "tools_used": [
    { "name": "nmap", "version": "7.99", "status": "ok" },
    { "name": "openssl", "version": "LibreSSL 3.3.6", "status": "degraded_shadowed" }
  ],
  "operator_notes": "(free text — fill in any observations from the engagement that aren't captured in the automated output)"
}
```

The sidecar is the source of truth for "what happened at this engagement." Edit `operator_notes` after the fact to capture context you couldn't include in the client letter.

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

## Version

This playbook is v1.0. When the underlying skills change in a way that affects the procedure (new classification rules, new scan defaults, additional report sections), bump `playbook_version` in the sidecar and update this document.

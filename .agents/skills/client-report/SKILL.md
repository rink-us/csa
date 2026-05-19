---
name: client-report
description: Translate a technical assessment report (from the assessment-report skill) OR a pentest engagement record (from the pentest-engagement skill) into an email-ready plain-text client letter. Use when the operator needs to share findings with a non-technical business owner, manager, or stakeholder — the same data, rewritten in business language with priorities, plain-English explanations, and concrete remediation steps.
license: MIT
compatibility: Consumes the JSON output of `assessment-report` (recon variant) or `pentest-engagement` (pentest variant).
metadata:
  bundle: network-discovery
  version: "1.1"
---

Render a technical JSON report into a plain-text client letter suitable for direct email. The audience is a business owner or non-technical stakeholder, NOT a security peer. Optimize for skimmability and "what do I do about it" clarity.

Two input shapes are supported:
- **`assessment-report`** — recon/posture report from `/netd:report`. Letter rendered via `/netd:letter`.
- **`EngagementRecord` (pentest variant)** — sidecar from `/pentest:engagement`. Letter rendered via `/pentest:letter`.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `report_path` | string | — | Absolute or repo-relative path to the JSON file |
| `report_type` | enum | auto-detect | `recon` or `pentest`. Auto-detected from the JSON shape if omitted — see "Shape detection" below |
| `output_dir` | string | (same dir as `report_path`) | Where to write the `.txt` |
| `filename` | string | auto | If omitted, replace the `.json` extension with `.txt` on the input filename |
| `signature_placeholder` | string | `[Your Name]\n[Your Title]\n[Contact Information]` | What to drop in for the closing — operator fills in before sending |

If `report_path` doesn't exist, or its shape matches neither the recon nor the pentest variant, refuse with a validation error.

### Shape detection

| If JSON contains... | Treat as |
| --- | --- |
| Top-level `target` AND `recommendations[]` AND at least one `stepN_*` key | `recon` |
| Top-level `engagement_id` AND `client.name` AND `sessions[]` AND `evidence_directory` | `pentest` |
| Both / neither / ambiguous | Refuse — ask the operator to pass `report_type` explicitly |

## Output format

Plain text. No markdown, no HTML, no Unicode beyond ASCII (so it renders identically across mail clients). Wrap body paragraphs at ~70 columns. Use ASCII-only section dividers (`----` under headings, `[ ... ]` brackets for priority labels).

### Structural template

```
Subject: Security Assessment Summary - <target> (<human date>)


Below is a summary of the security assessment we completed on
<human date> against your public website at <target>. A detailed
technical report is attached separately (<json filename>).


BOTTOM LINE
-----------

<2-3 sentences. State the headline grade, whether anything is urgent,
and the count of recommendations. Lead with the most reassuring or
most alarming framing - whichever is honest. Never sugar-coat.>


WHAT WE CHECKED
---------------

<Bulleted list, 3-5 items derived from which steps appear in the report:
  - "The public DNS and registration records for your domain" (if step1_recon)
  - "All standard internet-facing ports (TCP 1 through 1024) on your
    public IP address" (if step2_port_scan)
  - "The HTTP and HTTPS service configuration" (if step3_services)
  - "Your TLS certificate, supported encryption protocols, and cipher
    selection" (if step4_tls)
  - "Active hosts within <CIDR> and their device types" (if step5_network_map)
>


WHAT WE FOUND - THE GOOD NEWS
-----------------------------

<3-6 numbered findings. For each, lead with the plain-English positive
framing, then a short factual backing. Translate jargon - "AEAD with
forward secrecy" becomes "modern encryption". Anchor in summary.notable
plus the headline schema fields.>

<Skip this section entirely if there is genuinely no good news to lead
with - move straight to RECOMMENDATIONS.>


RECOMMENDATIONS
---------------

We grouped these by priority. <If no high-severity: "Nothing here is
urgent." | If any high-severity: "Items in the High Priority section
should be addressed in the current sprint, not deferred.">

[ HIGH PRIORITY - address before deferring ]      (only if any high)

  N. <Title from recommendation.title>
     <Underline of equal length>
     What it is: <2-3 sentences from recommendation.description, in
                  business language. Strip jargon. Explain the concept
                  before naming it.>

     Why it matters: <1-2 sentences. Tie to concrete impact - "an
                      attacker on the same Wi-Fi network", "credentials
                      sent in plaintext", "search engines flag the site
                      as insecure", etc. Avoid hand-wavy "best practice".>

     How to fix: <The recommendation.remediation, translated to
                  step-by-step language a business owner can either
                  do themselves or hand to their IT contact.>

     Time required: <Rough estimate - "5 minutes", "an afternoon for
                     someone familiar with the platform", etc.>


[ MEDIUM PRIORITY - we recommend doing this soon ]

  <same per-rec structure>


[ LOW PRIORITY - nice to have ]

  <same per-rec structure>


[ INFORMATIONAL - longer-term improvement ]

  <same per-rec structure>


WHAT WE DID NOT CHECK
---------------------

<Always include. Cite specific gaps from the report:
  - If summary.fronted_by is set: explain the CDN/origin distinction and
    that this report is the edge view.
  - If only IPv4 was scanned: note that IPv6 wasn't covered.
  - If no authenticated scans were attempted: note that.
  - If network-mapper wasn't run: skip mention - it isn't a gap for a
    single-host assessment.
The point is to set scope honestly, not to upsell.>


Attachment:
  - <json filename>
    (full technical report; can be shared with your IT contact)
```

## Tone rules

These are mandatory — apply them across every section:

1. **No emoji.** No Unicode symbols. ASCII only.
2. **No second person plural that talks down.** "We recommend" is fine; "You should really..." reads as scolding.
3. **No raw acronyms on first use.** Spell out HSTS, CSP, DNSSEC, CORS, OCSP, CT, CDN, etc. once each before abbreviating. Don't spell out repeatedly — it's an email, not a glossary.
4. **No marketing.** Don't refer to "industry best practices" or "advanced threats." State the concrete behavior in plain terms.
5. **No emergency framing unless an emergency actually exists.** A medium-severity HSTS gap is not "exposed to attack" — it is "vulnerable in the specific case of a hostile network during a visitor's first connection of the day."
6. **No closing salutation ("Best regards," "Sincerely," etc.) and no opening greeting ("Hello," "Hi Name,").** The operator usually pastes the body into a mail client that adds those — and our prior convention here is to omit them. The body ends at the `Attachment:` block. (Override if the operator explicitly asks for a salutation.)
7. **Honest framing of bottom line.** If the report is genuinely clean (no high-severity, no medium-severity), say so unambiguously in BOTTOM LINE. Don't manufacture concern to justify the engagement.

## Severity mapping

The `assessment-report` JSON's `recommendations[*].severity` values map directly:

| JSON severity | Section header | Order |
| --- | --- | --- |
| `high` | `[ HIGH PRIORITY - address before deferring ]` | 1 |
| `medium` | `[ MEDIUM PRIORITY - we recommend doing this soon ]` | 2 |
| `low` | `[ LOW PRIORITY - nice to have ]` | 3 |
| `informational` | `[ INFORMATIONAL - longer-term improvement ]` | 4 |

Omit a section header if the report has zero recommendations at that severity.

## Pentest variant

When `report_type=pentest` (or auto-detected from a pentest `EngagementRecord` JSON), the same tone rules and overall template apply, with the following field-mapping differences. The output filename convention is unchanged (input `.json` → output `.txt` next to it).

### Source-to-section mapping

| Letter section | Source in pentest JSON |
| --- | --- |
| `Subject:` line | `client.name` + scope + sidecar date (`started_at`) |
| `BOTTOM LINE` headline framing | Derived from `outcomes` counts (see "Headline framing" below) |
| `SCOPE CLARIFICATION` block (conditional) | Present if `scope_decisions[]` is non-empty OR if `outcomes.refused_out_of_scope / total_attempts >= 0.25` |
| `WHAT WE CHECKED` | Aggregated `candidate_summary` from each evidence record, grouped by attack surface |
| `WHAT WE FOUND - THE GOOD NEWS` | Evidence records where `outcome=failure` (attack tried + failed = defender win) or `outcome=success` AND `finding_severity=informational_positive` (control verified working) |
| `RECOMMENDATIONS` | Evidence records' `finding_for_customer` text, severity-sorted by `finding_severity` (see severity mapping below) |
| `WHAT WE DID NOT CHECK` | Translated `scope_decisions[]`, plus standard scope caveats (authenticated surface, IPv6, application-class attacks if not exercised) |
| `Attachment:` block | `artifacts.sidecar`, `artifacts.narrative` (if present), evidence-dir reminder |

### Headline framing

| Engagement outcome | BOTTOM LINE opener |
| --- | --- |
| 0 `success` AND 0 `partial` for executed exploits, 0 critical/high findings | "Your defenses held. We tested it as an outside attacker would and could not get in." |
| Any `success` outcomes | "We were able to <briefly describe initial access from the highest-severity success record>. The remainder of this letter explains how, what it implies, and how to close the gap." |
| `partial` outcomes only (probe-effective but no shell) | "We identified specific areas where your defenses functioned as intended and areas where they performed under expected, but not above." |
| `refused_out_of_scope` >= 25% of total attempts | Lead with a SCOPE CLARIFICATION block before BOTTOM LINE; then frame BOTTOM LINE around the in-scope subset only |

### Pentest finding-severity mapping

Evidence records use `finding_severity` rather than the recon-letter's `severity`. Map as follows:

| `finding_severity` value | Letter section |
| --- | --- |
| `critical`, `high` | `[ HIGH PRIORITY ]` |
| `medium` | `[ MEDIUM PRIORITY ]` |
| `low` | `[ LOW PRIORITY ]` |
| `informational` | `[ INFORMATIONAL ]` |
| `informational_positive` | Surface in `WHAT WE FOUND - THE GOOD NEWS`, NOT in `RECOMMENDATIONS` |

A record with `finding_severity=informational_positive` describes a hardening control that was verified working. It belongs in the GOOD NEWS section, never in RECOMMENDATIONS — surfacing such an item as "you should do this" reads as scolding the customer for a control they already have. A record without a `finding_for_customer` field at all is internal-only and omitted from the letter.

### Tool-honesty constraint

The pentest variant has one extra mandatory tone rule beyond the recon-letter rules:

> **The customer-facing wording must never imply tooling the operator didn't actually use.** Cross-reference `tools_used[]` from the sidecar. If a tool's `status` is `missing` and the engagement fell back to a manual probe, the letter MUST phrase coverage in terms of what was actually tested — e.g. "we sent five intentionally-invalid login attempts and observed the response" rather than "we ran wpscan against your plugins." Honest scoping is more important than tool-name-dropping.

## Write behavior

1. Read and parse `report_path`. Auto-detect shape using "Shape detection" above (or honor an explicit `report_type` input). Refuse if neither shape matches.
2. For pentest shape: also read each `rec-*.json` under `evidence_directory` to aggregate findings. Refuse with `evidence_dir_missing` if the directory doesn't exist on disk.
3. Derive default `filename` from the input — same basename, swap `.json` → `.txt`.
4. Write the file to `output_dir` (default: same directory as input).
5. Return:
   ```json
   { "skill": "client-report", "report_type": "recon" | "pentest", "path": "./reports/...-....txt", "size_bytes": N, "lines": L }
   ```

## Example invocations

```
# Recon (assessment-report) variant
client-report: report_path=./reports/report-montanacomputersolutions.com-2026-05-17-1825Z.json

# Pentest (EngagementRecord) variant
client-report: report_path=./reports/engagement-c-sharps-arms-2026-05-18.json
```

The recon variant writes `./reports/report-montanacomputersolutions.com-2026-05-17-1825Z.txt`; canonical example at [reports/report-montanacomputersolutions.com-2026-05-17-1825Z.txt](../../../reports/report-montanacomputersolutions.com-2026-05-17-1825Z.txt). The pentest variant writes `./reports/engagement-c-sharps-arms-2026-05-18.txt`; canonical example at [reports/engagement-c-sharps-arms-2026-05-18.txt](../../../reports/engagement-c-sharps-arms-2026-05-18.txt).

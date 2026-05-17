---
name: client-report
description: Translate a technical assessment report (produced by the assessment-report skill) into an email-ready plain-text client letter. Use when the operator needs to share findings with a non-technical business owner, manager, or stakeholder — the same data, rewritten in business language with priorities, plain-English explanations, and concrete remediation steps.
license: MIT
compatibility: Consumes the JSON output of the assessment-report skill.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Render a technical `assessment-report` JSON into a plain-text client letter suitable for direct email. The audience is a business owner or non-technical stakeholder, NOT a security peer. Optimize for skimmability and "what do I do about it" clarity.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `report_path` | string | — | Absolute or repo-relative path to the JSON produced by `assessment-report` |
| `output_dir` | string | (same dir as `report_path`) | Where to write the `.txt` |
| `filename` | string | auto | If omitted, replace the `.json` extension with `.txt` on the input filename |
| `signature_placeholder` | string | `[Your Name]\n[Your Title]\n[Contact Information]` | What to drop in for the closing — operator fills in before sending |

If `report_path` doesn't exist or isn't a JSON object with the expected `assessment-report` shape, refuse with a validation error.

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

## Write behavior

1. Read and parse `report_path`. If parse fails or `recommendations` is missing, refuse.
2. Derive default `filename` from the input — same basename, swap `.json` → `.txt`.
3. Write the file to `output_dir` (default: same directory as input).
4. Return:
   ```json
   { "skill": "client-report", "path": "./reports/...-....txt", "size_bytes": N, "lines": L }
   ```

## Example invocation

```
client-report: report_path=./reports/report-montanacomputersolutions.com-2026-05-17-1825Z.json
```

Writes `./reports/report-montanacomputersolutions.com-2026-05-17-1825Z.txt`. For reference, the canonical example output is at [reports/report-montanacomputersolutions.com-2026-05-17-1825Z.txt](../../../reports/report-montanacomputersolutions.com-2026-05-17-1825Z.txt).

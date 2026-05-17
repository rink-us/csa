---
name: "netd:letter"
description: Render an existing technical assessment report (JSON) into a client-facing plain-text letter suitable for email. Operates on an existing JSON file produced by /netd:report — does not re-run any scans.
category: Network Discovery
tags: [network, report, client, email]
---

Translate a saved JSON assessment report into the email-ready text format. The output goes next to the JSON in `./reports/` with the same basename and a `.txt` extension.

**Input**: `/netd:letter <json-path-or-target>`

- Pass either:
  - A path to a saved JSON report (`./reports/report-example.com-2026-05-17-1830Z.json`), OR
  - Just a target name (`example.com`) — the command will find the most recent `report-<target>-*.json` under `./reports/` and use that

If neither resolves, ask the user for an explicit path.

**Steps**

1. Resolve the input argument:
   - If it's a path, verify it exists and parses as JSON.
   - If it's a bare name, glob `./reports/report-<target>-*.json`, sort lexically (timestamps are sortable), and pick the last entry. If the glob has multiple matches, list them and ask which to use unless one is obviously the most recent.
2. Verify the JSON has the expected `assessment-report` shape — at minimum the keys `target`, `recommendations`, and at least one `stepN_*` entry. If not, refuse with a clear error.
3. Invoke [.agents/skills/client-report/SKILL.md](../../skills/client-report/SKILL.md) with `report_path=<resolved path>`. The skill writes the `.txt` next to the JSON, applies all tone rules, sorts recommendations by severity, and uses the signature placeholder convention (no opening salutation, no closing salutation — body ends at `Attachment:`).
4. Report back to the user:
   - The written `.txt` path
   - Line count and approximate word count (so they can gauge whether it fits a one-screen email)
   - A reminder that `[Your Name] / [Your Title] / [Contact Information]` placeholders need filling in before sending — IF the skill chose to include a signature block. (Current default: no signature block; the operator pastes into their mail client which adds the signature automatically.)

**Failure handling**

- If the input JSON is malformed or missing `recommendations`, refuse — don't try to render half a letter.
- If the report has zero recommendations, the letter still renders with the BOTTOM LINE expressing the clean result and the WHAT WE DID NOT CHECK section preserving scope honesty. Don't fabricate findings.
- If `./reports/` is read-only or the .txt path collides with an existing file, ask the operator whether to overwrite, append `-v2`, or abort.

**Example**

```
/netd:letter ./reports/report-example.com-2026-05-17-1830Z.json
/netd:letter example.com
/netd:letter montanacomputersolutions.com
```

**Output**

```
Wrote: ./reports/report-example.com-2026-05-17-1830Z.txt
Length: 168 lines, ~870 words (fits one comfortably-sized email)
```

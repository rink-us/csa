## ADDED Requirements

### Requirement: Client report consumes a saved technical JSON
The agent skill SHALL accept a `report_path` pointing at a JSON file produced by the `assessment-report` skill, validate that the JSON contains at minimum `target`, `recommendations`, and at least one `stepN_*` entry, and refuse if the input is malformed.

#### Scenario: Valid report path
- **WHEN** `report_path` points at a file matching the assessment-report shape
- **THEN** the skill proceeds to render the letter

#### Scenario: Missing recommendations field
- **WHEN** the JSON parses but has no `recommendations` array
- **THEN** the skill refuses with a validation error explaining the missing field

### Requirement: Client letter is ASCII-only plain text
The agent skill SHALL produce output that is plain text, contains no markdown formatting, no HTML, no Unicode characters beyond ASCII, and uses ASCII section dividers (`----` underlines, `[ ... ]` brackets for priority labels) so it renders identically across mail clients.

#### Scenario: Output is ASCII-only
- **WHEN** the skill writes the .txt file
- **THEN** every byte is a printable ASCII character or whitespace (no emoji, no smart quotes, no em-dashes)

#### Scenario: No markdown syntax
- **WHEN** the skill writes the .txt file
- **THEN** the output contains no `#` headers, no `*` or `_` for emphasis, no `[link](url)` syntax

### Requirement: Client letter follows the structural template
The agent skill SHALL produce a letter with these sections in order: Subject line, BOTTOM LINE, WHAT WE CHECKED, WHAT WE FOUND - THE GOOD NEWS (omittable when honestly no good news), RECOMMENDATIONS (with severity-grouped subsections), WHAT WE DID NOT CHECK, Attachment.

#### Scenario: Standard section order
- **WHEN** a letter is rendered from a report with mixed-severity recommendations
- **THEN** the sections appear in the order listed above, separated by blank lines and ASCII dividers

#### Scenario: No-findings letter still includes scope-honesty section
- **WHEN** a report has zero recommendations
- **THEN** the BOTTOM LINE explicitly says so, the WHAT WE FOUND section may be omitted or trimmed, and the WHAT WE DID NOT CHECK section is still present

### Requirement: Client letter sorts recommendations by severity
The agent skill SHALL group the recommendations under `[ HIGH PRIORITY - ... ]`, `[ MEDIUM PRIORITY - ... ]`, `[ LOW PRIORITY - ... ]`, `[ INFORMATIONAL - ... ]` subsections in that order, with empty subsections omitted entirely.

#### Scenario: Recommendations grouped by severity header
- **WHEN** the source report contains 1 high, 2 medium, and 0 low recommendations
- **THEN** the letter contains the HIGH PRIORITY and MEDIUM PRIORITY subsections; LOW PRIORITY is omitted (no empty header)

### Requirement: Each recommendation in the letter is structured for the non-technical reader
The agent skill SHALL render each recommendation with sub-blocks: "What it is" (2–3 sentences in plain language with jargon spelled out on first use), "Why it matters" (1–2 sentences tied to concrete impact), "How to fix" (step-by-step language), and "Time required" (rough estimate).

#### Scenario: Acronym spelled out on first use
- **WHEN** a recommendation references HSTS
- **THEN** the first mention in the letter spells it out as "HSTS (HTTP Strict Transport Security)" or equivalent; subsequent mentions in the same letter may use the abbreviation

#### Scenario: Plain-language impact tied to concrete attacker behavior
- **WHEN** a recommendation describes "Why it matters"
- **THEN** the text refers to specific attacker actions ("an attacker on the same Wi-Fi network", "credentials sent in plaintext", etc.), not generic phrases like "best practice"

### Requirement: Client letter omits opening and closing salutations
The agent skill SHALL NOT include opening greetings ("Hello,", "Hi <name>,") or closing salutations ("Best regards,", "Sincerely,") in the body by default. The operator's mail client typically adds those.

#### Scenario: No greeting line
- **WHEN** the letter is rendered
- **THEN** the body begins directly with the substantive content ("Below is a summary...") after the Subject line

#### Scenario: Body ends at Attachment block
- **WHEN** the letter is rendered
- **THEN** the last non-blank lines reference the attachment, with no closing salutation appended

### Requirement: Client letter is saved next to the source JSON
The agent skill SHALL write the .txt file alongside the source JSON with the same basename and a `.txt` extension by default, overridable via `output_dir`/`filename`.

#### Scenario: Default filename
- **WHEN** the input is `./reports/report-example.com-2026-05-17-1825Z.json`
- **THEN** the output is `./reports/report-example.com-2026-05-17-1825Z.txt`

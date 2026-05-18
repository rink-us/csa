## Context

The original `agents-skills-network-discovery` change designed five capabilities at a single level of abstraction: scan, recon, enumerate, analyze, map. Running them in a real end-to-end engagement surfaced two architectural needs the original design didn't anticipate:

1. **A reporting layer above the capabilities.** Operators don't ship raw envelope outputs to clients. They want a merged technical report and an email-ready letter. Doing this ad-hoc in the agent loop works once; doing it consistently across engagements wants dedicated skills with documented schemas and tone rules.

2. **Real-world embedded devices break the original assumptions.** Two specific failures emerged:
   - Consumer IoT (Echo, Owlet, smart-home microcontrollers, phones with random MACs) responds to ARP but silently drops layer-3 probes. A naive port-scan-everyone approach grinds for 30+ minutes against devices that have no scan-target value.
   - Embedded admin UIs (Verizon CPE, Canon printer, similar) respond to probes so slowly that aggressive nmap fingerprinting (`-sV --version-intensity 9 -sC`) exhausts the host-timeout, returning empty port data. Direct banner grabs (`curl -sI`, `openssl s_client`, `dig CHAOS TXT`) get the same useful information in seconds.

The implementation has already evolved to address both — this change brings the specs into agreement.

## Goals / Non-Goals

**Goals:**
- Add specs for `assessment-report`, `client-report`, `vuln-correlator` that match what already exists in `.agents/skills/`
- Update `port-scanner` / `network-mapper` / `service-enumerator` specs to require the performance and classification behaviors that the SKILL.md files already document
- Capture the embedded-device anti-pattern in spec form so future implementers don't re-discover it
- Keep all new requirements behaviorally testable — each new requirement comes with at least one Scenario

**Non-Goals:**
- No new external dependencies beyond optionally `snmpget`. Specifically not adding nmap NSE vuln scripts (`--script vuln`) — that's a deliberate scope cut for a future change.
- No spec for the orchestration layer (slash commands `/netd:vuln-scan`, `/netd:engagement`, etc.). Those are operator-facing automations, not capabilities. Their specs would be a separate concern (workflow specs) that this change doesn't tackle.
- No spec for the `playbooks/` directory. Playbooks are narrative procedure documents for humans; spec'ing them would conflate two artifact types.
- No retroactive testing requirement — the original `agents-skills-network-discovery` change skipped the live-network verification phase for the new skills because each is invoked under live conditions ad-hoc rather than via a fixture suite. This change preserves that posture.

## Decisions

**1. The three new skills are first-class capabilities, not "utilities."**
- **Decision**: `assessment-report`, `client-report`, `vuln-correlator` each get their own spec under `openspec/specs/<name>/` like the original five.
- **Rationale**: They have well-defined inputs, outputs, and behaviors that other agents will need to invoke. They're not internal helpers. Treating them as utilities would orphan them from the spec library and make their requirements impossible to verify.
- **Alternative considered**: bundle reporting + correlation as sections inside the existing specs. Rejected — different lifecycle, different audience.

**2. The embedded-device anti-pattern is encoded as a `port-scanner` requirement, not a separate guideline.**
- **Decision**: The `port-scanner` spec gains an explicit requirement: "The skill SHALL NOT use `-sV --version-intensity 9` or `-sC` as defaults against hosts whose device_class is `gateway`, `router_or_ap`, `printer`, `iot_camera`, `vendor_cloud_iot`, or `network_device`."
- **Rationale**: A guideline that lives only in prose gets ignored by the next implementer. A required behavior with a Scenario is testable: "Given device_class=printer and intensity=default, when invocation is built, the flags must not include `--version-intensity 9` or `-sC`."
- **Alternative considered**: leave it as guidance in the SKILL.md only. Rejected — already proven that the guideline alone isn't sticky; tonight's session re-discovered the timeout four times before encoding it.

**3. `vuln-correlator` is designed for sub-agent invocation, but the spec doesn't mandate sub-agents.**
- **Decision**: The spec describes the per-host input/output contract. It notes the sub-agent invocation pattern as a recommendation. Single-process invocation is equally valid for low-host-count cases.
- **Rationale**: Sub-agents are an orchestration choice (which the calling slash command makes), not a capability requirement. The capability is "analyze one host"; how the caller scales out is its own concern.

**4. The embedded-device signal capture in `service-enumerator` is a normalized output, not a free-form field.**
- **Decision**: The `service-enumerator` spec adds named fields to the output: `tls.subject_org`, `tls.validity_window_years`, `headers.security_headers_present[]`, `headers.security_headers_missing[]`, `behavior.redirect_to_https`, `headers.last_modified_year`. Each is required to be populated when the corresponding signal is observable.
- **Rationale**: `vuln-correlator`'s embedded-device fallback path keys off these specific fields. Free-form text would force every consumer to re-parse. Defining them in the spec makes the producer/consumer contract explicit.
- **Alternative considered**: put all observations into a generic `observations: []` array of `{key, value}`. Rejected — the field names matter for the downstream fallback rules.

**5. The 10-class device taxonomy in `network-mapper` is named and frozen, but extensible.**
- **Decision**: The spec lists the 10 classes (`gateway`, `router_or_ap`, `printer`, `network_device`, `server`, `phone`, `vendor_cloud_iot`, `apple_device`, `iot_camera`, `unknown`) and requires that the skill emit one per host. The signal-to-class mapping table is informative — implementations may refine it as IoT product names evolve.
- **Rationale**: Downstream consumers (the `skip_classes` input, the report's classify-and-skip language, the client letter's IoT-vs-server framing) all branch on these specific names. Renaming or removing one would silently break the chain. Adding new classes is allowed via a future change.

**6. Reports go to `./reports/` by convention; the path is not absolute.**
- **Decision**: The `assessment-report` spec requires the skill to default to `./reports/` (relative to the caller's cwd) but accept an explicit `output_dir` override.
- **Rationale**: Repository-relative paths keep the convention portable across operators' working directories. Absolute paths would tie the output to one user's filesystem.

## Risks / Trade-offs

**Risk: Spec adds requirements the implementation doesn't perfectly match.**
- **Mitigation**: Each new requirement was derived from re-reading the current SKILL.md files. The spec is meant to formalize what's there, not invent new behavior. If a Scenario fails against the current implementation, that's a bug in the SKILL.md that we fix in tasks.md as part of this change rather than relaxing the spec.

**Risk: The `service-enumerator` named output fields become brittle as new signals are discovered.**
- **Mitigation**: The field set is intentionally additive — new signals (e.g. a future `hsts_includeSubDomains: bool`) can be added in a follow-up change without breaking existing consumers. The current 6 fields are the minimum needed for `vuln-correlator`'s fallback path to work.

**Risk: The anti-pattern requirement in `port-scanner` could miss new device classes.**
- **Mitigation**: The requirement targets specific named classes. When a new class is added to `network-mapper`'s taxonomy in a future change, the anti-pattern list must be updated in the same change. Documenting this expectation in the spec text reduces the chance of drift.

**Risk: `vuln-correlator`'s reliance on CIRCL and NVD APIs creates an external-availability dependency the spec doesn't address.**
- **Mitigation**: The spec requires graceful degradation when sources are unreachable (non-fatal per-service error, empty cves array, narrative continues). Adding a third source or an offline CVE database is out of scope for this change.

**Risk: The reporting skills create artifacts that may contain client-sensitive data, but the spec doesn't address retention or sharing.**
- **Mitigation**: The `assessment-report` spec notes that output may contain sensitive findings and recommends gitignoring `./reports/`. Retention/sharing is an operator-policy concern outside the capability's scope.

## Migration Plan

1. **Phase 1**: Write the three new specs (`assessment-report`, `client-report`, `vuln-correlator`) as `ADDED Requirements` deltas.
2. **Phase 2**: Write the three modification deltas (`port-scanner`, `network-mapper`, `service-enumerator`) using `MODIFIED Requirements` blocks for changed requirements and `ADDED Requirements` for net-new ones.
3. **Phase 3**: Run the tasks (verify the spec scenarios pass against the current implementation; fix any drift by updating the SKILL.md, NOT by relaxing the spec).
4. **Phase 4**: Archive with `/opsx:archive` to promote all six specs (3 new + 3 modified) to the main library.

No code changes are needed in `.agents/skills/` for the new-capability specs — those skills already exist. The modification specs may surface small drifts that need SKILL.md touch-ups during the tasks phase.

## Resolved Questions

- **`snmpget` dependency status** — promoted to **required** in this change. Rationale: SNMP probing is a documented part of the service-enumerator playbook; making it optional risks silently dropping SNMP findings when the binary happens not to be present. The install commands across macOS, Debian/Ubuntu, and Fedora/RHEL include it inline.
- **Engagement workflow formal spec** — decision: open a **follow-up change to formalize `/netd:engagement`** before treating the workflow as stable. The follow-up will define the engagement sidecar JSON shape as a first-class capability, the required precheck-then-scope-then-execute phase order, and the artifact set every engagement must produce. This change does NOT spec the engagement — it stays at the orchestration layer in `.agents/commands/netd/engagement.md` until the follow-up lands.
- **Offline CVE database for `vuln-correlator`** — decision: open a **follow-up change to add offline-CVE-lookup support**. The follow-up will package an NVD snapshot ingestion path and add an `offline` value to `cve_source`. Useful for engagements where outbound HTTPS to CIRCL/NVD is not permitted (air-gapped client environments, restrictive corporate firewalls). Out of scope for this change.

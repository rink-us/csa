## Context

This repo's existing automation lives entirely in [.agents/skills/](.agents/skills/) as Claude Code markdown skills — single `SKILL.md` files with YAML frontmatter, a description that drives skill selection, and a prompt body that tells the agent how to perform the work. There is no Python runtime, no class library, no test harness. The network-discovery proposal must match this pattern: each capability is a prompt that instructs the agent how to invoke existing CLI tools (`nmap`, `dig`, `whois`, `openssl`, `traceroute`) and normalize their output.

## Goals / Non-Goals

**Goals:**
- Five `SKILL.md` files under [.agents/skills/](.agents/skills/) matching the existing skill format
- Each skill self-contained: trigger conditions, inputs, tool invocations, output schema
- Two shared docs (`OUTPUT_SCHEMAS.md`, `DEPENDENCIES.md`) so the skills stay consistent with each other
- Skills behave correctly when external CLIs are missing (detect, report, don't crash the agent)

**Non-Goals:**
- No Python package, runtime library, async wrapper, or unit-test suite — these are prompts, not code
- No reimplementation of `nmap`/`dig`/`openssl` logic — skills shell out to the canonical tool
- No SIEM/SOAR integration; the skill output is text/JSON for the calling agent to use
- No exploitation primitives — reconnaissance, enumeration, and posture analysis only

## Decisions

**1. Markdown skills, not a Python library**
- **Decision**: Each capability is a `SKILL.md` file under [.agents/skills/](.agents/skills/) following the existing repo pattern
- **Rationale**: Matches every other skill in this repo; zero new runtime; the agent already knows how to invoke CLI tools — it just needs the right playbook
- **Alternative considered**: Python module library with base classes and async wrappers — rejected as inconsistent with the rest of the repo and over-engineered for what is essentially "tell the agent how to run nmap and parse the output"

**2. Shell out to canonical CLIs**
- **Decision**: Skills instruct the agent to invoke `nmap`, `dig`, `whois`, `openssl s_client`, `traceroute`/`mtr`, `curl` — not custom socket code
- **Rationale**: These tools are battle-tested, ubiquitous, and already in operators' environments. The skill's job is composition, not reimplementation.
- **Alternative considered**: Recommend Python libraries (`python-nmap`, `dnspython`, `cryptography`) — rejected; adds a hidden runtime dependency for what is fundamentally CLI orchestration.

**3. Normalize output to a shared schema**
- **Decision**: Each skill's prompt body specifies a JSON output schema; overlapping shapes (a "port + service + banner" record, a "host" record, a "cve" record) are defined once in `OUTPUT_SCHEMAS.md` and referenced by name
- **Rationale**: Lets the agent chain skills (e.g. `network-mapper` → `service-enumerator` → `tls-analyzer`) without re-parsing each tool's native format
- **Alternative considered**: Raw tool output passthrough — rejected; pushes the parsing burden onto every downstream caller

**4. Unprivileged defaults; privileged variants opt-in**
- **Decision**: Default invocations use unprivileged variants (`nmap -sT` TCP-connect, ICMP traceroute, no ARP scan). Privileged variants (`-sS` SYN scan, ARP sweep) require the operator to opt in and confirm they have root/sudo
- **Rationale**: Most agent runs won't have sudo; failing on a privileged default would surprise users; privileged scans also have a stronger detection footprint and deserve an explicit opt-in

**5. Skill-level caching guidance, not a cache implementation**
- **Decision**: The reconnaissance skill prompt advises the agent to reuse recent DNS/WHOIS results within the same conversation rather than re-querying; no on-disk cache is built
- **Rationale**: Consistent with markdown-skill simplicity. Persistent caching would require runtime code we're explicitly not building.

**6. Tool-missing handling is reactive, not proactive**
- **Decision**: Skills don't pre-check binary availability; they invoke the tool and, on `command not found`, report what's missing and link to `DEPENDENCIES.md` install instructions
- **Rationale**: Faster path on the happy case; clear remediation on the unhappy case; no platform-detection logic baked into prompts

## Risks / Trade-offs

**Risk: Privileged variants suggested in environments without sudo**
- **Mitigation**: Default to unprivileged; flag privileged variants explicitly; skill prompt instructs agent to confirm sudo availability before suggesting them.

**Risk: Stale/inaccurate CVE correlation**
- **Mitigation**: Service-enumerator's CVE step is optional and clearly labelled as a hint, not a verdict; output flags the data source and freshness.

**Risk: External CLI version skew across platforms**
- **Mitigation**: `DEPENDENCIES.md` pins minimum versions; skills use flags supported by those minimums; fall back to portable alternatives (`drill` for `dig`, `mtr` for `traceroute`) where helpful.

**Risk: Skill scope creep — overlap between port-scanner, service-enumerator, and network-mapper**
- **Mitigation**: Each skill's prompt explicitly states what it does and what it defers to the others; `OUTPUT_SCHEMAS.md` shares the record shapes so chained invocations work cleanly.

## Migration Plan

1. **Phase 1**: Land the two shared docs first (`OUTPUT_SCHEMAS.md`, `DEPENDENCIES.md`) so subsequent skills can reference them
2. **Phase 2**: Author `port-scanner` and `network-reconnaissance` (the foundations — no dependencies on other skills)
3. **Phase 3**: Author `service-enumerator` (depends on port lists from Phase 2) and `tls-analyzer` (independent)
4. **Phase 4**: Author `network-mapper` (composes the others)
5. **Phase 5**: End-to-end exercise — drive one skill from another in a single agent run to verify the shared schemas chain cleanly

Deployment is additive — new files under [.agents/skills/](.agents/skills/). No edits to existing skills or `openspec` workflow.

## Resolved Questions

- **Authenticated/credentialed scanning** — out of scope for v1. The five skills are unauthenticated only. A future change can add `authenticated-service-check` etc.
- **Acceptable timeout** — per-skill, expressed in the prompt as suggested defaults (port-scan: 5 min; reconnaissance ops: 30 s each; TLS analysis: 60 s; network-mapper: 15 min for a /24). Operator can override.
- **Historical trending** — out of scope. Skills emit point-in-time snapshots. Operators who want trending feed snapshots into their own store.

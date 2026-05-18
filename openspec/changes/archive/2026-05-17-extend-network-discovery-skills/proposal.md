## Why

The `agents-skills-network-discovery` change (archived 2026-05-17) shipped five capability skills for network discovery, reconnaissance, and TLS analysis. In practice — running them against a real home/SOHO network during a live engagement — three drift points emerged that the synced main specs don't yet reflect:

1. **Three skills exist in `.agents/skills/` that have no spec at all.** `assessment-report`, `client-report`, and `vuln-correlator` were added to the implementation to support the engagement reporting workflow but were never proposed as spec'd capabilities.
2. **Three existing specs are behind the implementation.** `port-scanner`, `network-mapper`, and `service-enumerator` SKILL.md files grew new inputs and behavior (port-set presets, timing profiles, device classification, embedded-device banner-grab playbook) based on live-engagement lessons; the specs still describe the v1 surface.
3. **A real performance failure mode was discovered and encoded into the SKILL.md files** — aggressive nmap fingerprinting (`--version-intensity 9 -sC`) reliably exhausts `--host-timeout` against embedded admin UIs (consumer routers, printers, IoT controllers). The fix (direct banner grabs via `curl`/`openssl`/`dig`/`nc`) is now documented as the preferred path, but the spec doesn't require it.

This change brings the specs back in line with the implementation so that future agents reading the main spec library see what the system actually does, and so that the next archived change starts from a coherent baseline.

## What Changes

- Add specs for the three new capabilities: `assessment-report`, `client-report`, `vuln-correlator`
- Modify the `port-scanner` spec to require the new performance inputs (`port_set`, `timing_profile`, `discovery_mode`, `host_timeout_seconds`) and document the embedded-device anti-pattern
- Modify the `network-mapper` spec to require the device-class taxonomy, the `classify_first`/`skip_classes` inputs, and the conservatism rule (when uncertain, classify as `unknown` so the caller still scans)
- Modify the `service-enumerator` spec to require the expanded per-protocol banner-grab playbook (TLS cert probe, DNS CHAOS, IPP/LPR/SNMP) and the embedded admin UI signal-capture set (security headers checklist, TLS vendor identification, asset-age inference)
- No new external dependencies — the implementation already uses standard CLIs (`curl`, `openssl`, `dig`, `nc`, `snmpget`) that the existing `DEPENDENCIES.md` covers (with `snmpget` as a new optional addition)

## Capabilities

### New Capabilities

- `assessment-report`: Merge per-skill envelopes from the network-discovery bundle into a single technical JSON report with summary block and prioritized recommendations; write to `./reports/`
- `client-report`: Render a saved technical JSON report into an email-ready plain-text client letter following a documented structural template and tone rules
- `vuln-correlator`: Per-host CVE lookup and findings synthesis. Sub-agent-shaped (one host per invocation); supports both CIRCL/NVD CVE correlation AND an embedded-device fallback path when no product+version is available

### Modified Capabilities

- `port-scanner`: Add `port_set`, `timing_profile`, `discovery_mode`, `host_timeout_seconds` inputs. Add anti-pattern requirement: skill MUST NOT default to `--version-intensity 9` or `-sC` against embedded devices.
- `network-mapper`: Add `classify_first` and `skip_classes` inputs. Add 10-class device taxonomy as a required behavior. Add conservatism rule: when device class is uncertain, classify as `unknown` (so the host still gets scanned).
- `service-enumerator`: Expand per-protocol banner-grab playbook with TLS cert, DNS CHAOS, IPP, LPR, SNMP entries. Add embedded-device signal-capture requirements (security headers checklist, TLS vendor identification, asset-age inference).

## Impact

- Affects: this repo's [.agents/skills/](.agents/skills/) directory (already updated — this change retroactively spec's what's there) plus the main spec library [openspec/specs/](openspec/specs/) (will gain 3 new specs and 3 modified specs on archive)
- APIs: none (skills are agent-side prompts, not a runtime API)
- Dependencies: `snmpget` becomes an optional CLI for the SNMP banner-grab path; documented as optional in `DEPENDENCIES.md`. No other new tools.
- Systems: none
- Breaking changes: none — the modifications to existing specs are additive (new inputs have defaults; existing invocations still work)

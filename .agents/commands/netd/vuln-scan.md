---
name: "netd:vuln-scan"
description: Fast end-to-end vulnerability scan of a local network. Discovers hosts via ARP, classifies device types to skip phone-home-only IoT, runs parallel port scans on the rest, then spawns parallel sub-agents for per-host CVE analysis. Optimized for the typical home/SOHO case where most devices respond to ARP but drop layer-3 probes. Targets ~3-5 minute end-to-end runtime for a /24 instead of 30+ minutes of naive sequential scanning.
category: Network Discovery
tags: [network, vuln, parallel, sub-agents, workflow]
---

End-to-end vulnerability discovery against a local network. The performance win over running `/netd:scan` per host serially is large because this command does three things differently:

1. **Classifies devices BEFORE scanning** — skips port-scanning known phone-home IoT classes (Amazon Echo, Google Home, smart-home microcontrollers, phones with random MACs) where there's nothing to scan.
2. **Launches one nmap process per remaining host in parallel** — wall time becomes `max(slowest_host)` instead of `sum(all_hosts)`. Critical when targets silently drop probes.
3. **Spawns parallel sub-agents for the CVE analysis phase** — each agent owns one host, looks up CVEs concurrently, returns a findings block. Main loop merges them.

Result on a typical /24 with ~12-15 devices: end-to-end in 3-5 minutes vs. 30+ minutes for naive serial scanning.

---

**Input**: `/netd:vuln-scan <scope> [port-set] [scan-all] [no-letter]`

- `<scope>` (required) — CIDR (`192.168.1.0/24`), range, or comma-separated host list. Same validation as `/netd:map`.
- `[port-set]` — `top-100` (default, fastest), `top-1024`, `iot-curated`, or `full`. See [port-scanner SKILL.md](../../skills/port-scanner/SKILL.md) for definitions.
- `[scan-all]` — pass `scan-all` to disable the classify-and-skip optimization. Every up host gets port-scanned. Slower; only useful when you suspect a phone-home-classed device may actually have unexpected services.
- `[no-letter]` — by default a client-facing `.txt` is generated alongside the JSON report. Pass `no-letter` to skip.

If no scope is supplied, ask the user.

---

## Steps

### Phase 1: Discovery (~10 seconds)

1. Parse `scope` and flags.
2. Confirm scope passes the network-mapper validation (reject `/0`, refuse anything wider than `/16` without `allow-wide`).
3. Run ARP host discovery per [network-mapper SKILL.md](../../skills/network-mapper/SKILL.md) (`discovery_method=arp`). Requires sudo — if not available, fall back to `discovery_method=ping` and warn that the inventory will be incomplete.
4. Reverse-DNS resolve each up host (parallel — use `getent hosts` or `dig -x` per IP; cap at 2s per lookup).

### Phase 2: Classification (~5 seconds)

5. For each up host, derive `device_class` per the table in [network-mapper SKILL.md](../../skills/network-mapper/SKILL.md) using the PTR + MAC vendor + hostname signals.
6. Build two sets:
   - `to_scan`: hosts whose `device_class` is in {`gateway`, `router_or_ap`, `printer`, `network_device`, `server`, `apple_device`, `iot_camera`, `unknown`}
   - `to_skip`: hosts whose `device_class` is in {`phone`, `vendor_cloud_iot`}
7. If `scan-all` was passed, move everything to `to_scan`.
8. Report the split to the user before launching scans, e.g.:
   ```
   Will scan 4 hosts: 192.168.1.1 (gateway), 192.168.1.100 (router_or_ap),
                     192.168.1.164 (printer), 192.168.1.163 (iot_camera)
   Will skip 9 hosts (phone or vendor_cloud_iot): 192.168.1.157 (phone:Pixel-8-Pro),
     192.168.1.158 (vendor_cloud_iot:tuya), 192.168.1.160 (vendor_cloud_iot:amazon-Echo), ...
   ```

### Phase 3: Parallel port scans (~30 seconds to ~3 minutes)

9. For each host in `to_scan`, launch a **separate background nmap process** with the documented unprivileged invocation from port-scanner SKILL.md, using:
   - `port_set` from the user's argument (default `top-100`)
   - `discovery_mode=force` (skip nmap's own host discovery — we already know they're up via ARP)
   - `timing_profile=lan` (aggressive local-network defaults)
   - `host_timeout_seconds=120` (cap each host at 2 minutes)
   - `--version-intensity 5` (the documented default — do NOT use 9 or `-sC` against embedded admin UIs; see [port-scanner SKILL.md "Anti-pattern: aggressive fingerprinting against embedded devices"](../../skills/port-scanner/SKILL.md))
   - XML output to `/tmp/vuln-scan-<run-id>/scan-<ip>.xml`
10. Use bash backgrounding (`&` + `wait`) to launch all N scans simultaneously. Total wall time is `max(per-host scan time)` — typically 30s for hosts with services, capped at 120s by `host-timeout`.

   **IMPORTANT — zsh word-splitting:** if you store nmap flags in a shell variable and expand it on the command line, use `${=NMAP_FLAGS}` in zsh to force splitting. Default zsh passes the entire variable as a single argument, which nmap rejects with "Scantype not supported." Safer alternative: inline the flags directly per nmap invocation instead of using a variable.

11. After `wait`, parse each XML and build a `Port[]` list per host. Hosts with zero open ports get recorded but do not trigger Phase 4 work for themselves.

### Phase 3.5: Direct banner grabs (always, ~10 seconds)

For each host in `to_scan` that has at least one open port, **launch parallel banner grabs** per the [service-enumerator SKILL.md per-protocol playbook](../../skills/service-enumerator/SKILL.md) — `curl -sI` for HTTP/HTTPS, `openssl s_client` for TLS cert details, `dig CHAOS TXT version.bind` for DNS, `nc -w 3` for raw protocols (LPR, SMTP, FTP, SSH).

Why this phase exists: embedded admin UIs (routers, printers, IoT controllers) routinely don't return `Server:` headers in nmap's `-sV` output, so the port-scan phase produces empty product+version data. Direct banner grabs reliably capture security headers, TLS cert subject/issuer (which often reveals vendor), and behavior signals (HTTP-to-HTTPS redirects, asset Last-Modified dates) that the embedded-device fallback in `vuln-correlator` then uses for finding generation.

Real-world observation: a 5-minute aggressive nmap fingerprinting attempt against 3 embedded devices (Verizon CR1000B + Verizon CE1000A + Canon printer) returned zero useful service data. The same 3 hosts probed in parallel via banner grabs completed in under 10 seconds and revealed the full TLS posture, security header inventory, and a Canon firmware-age signal (Last-Modified: 2007). **Always run Phase 3.5 alongside Phase 3 for embedded device classes** (`gateway`, `router_or_ap`, `printer`, `iot_camera`, `network_device`).

### Phase 4: Parallel CVE analysis via sub-agents (~30 seconds)

12. Filter the scanned hosts to those with at least one open port. Include hosts whether or not their services have `product`+`version` data — the `vuln-correlator` skill's embedded-device fallback path will use the Phase 3.5 banner data for hosts without version info.
13. For each such host, **spawn one Agent (`subagent_type=general-purpose`)** with the prompt:
    ```
    You are analyzing one host from a network vulnerability scan. Read the
    vuln-correlator skill at .agents/skills/vuln-correlator/SKILL.md and follow
    its instructions exactly against this host's data, including the embedded-
    device fallback path when service product/version data is missing:

    <host JSON with ports + service details + banner_grab_data from Phase 3.5>

    Use cve_source=circl. Do NOT scan or probe — only do the CVE lookups via
    curl and the per-host narrative. Return a single JSON object matching the
    skill's output structure. Under 800 words of analysis.
    ```
14. Send all Agent invocations in a single message (multiple tool uses in one block) so they run concurrently — DO NOT serialize them.
15. When all sub-agents return, collect their output blocks.

### Phase 5: Report assembly (~5 seconds)

16. Build the merged result:
    - `summary`: scope, devices_total, devices_scanned, devices_skipped, hosts_with_services, total_cves_found, highest_severity
    - `inventory[]`: every up host with its device_class, hostname, MAC, vendor, and whether it was scanned
    - `vuln_findings[]`: the sub-agent outputs concatenated, sorted by severity descending
    - `errors[]`: any per-host scan failures or sub-agent failures
17. Call [assessment-report skill](../../skills/assessment-report/SKILL.md) with this merged data to write `./reports/report-<sanitized-scope>-vuln-<timestamp>.json`.
18. Unless `no-letter` was passed, chain into [/netd:letter](letter.md) for the client-facing `.txt`.

---

## Output

```
=== /netd:vuln-scan summary ===
Scope:                192.168.1.0/24
Devices discovered:   13 (via ARP)
Devices scanned:      4 (after classify-and-skip)
Devices skipped:      9 (phone or vendor_cloud_iot)
Hosts with services:  3
Total CVEs found:     12 (2 critical, 5 high, 5 medium)

Highest finding: Apache httpd 2.4.7 (EOL since 2017) on 192.168.1.164 — 3 high CVEs

Saved:
  reports/report-192.168.1.0_24-vuln-2026-05-17-2030Z.json
  reports/report-192.168.1.0_24-vuln-2026-05-17-2030Z.txt  (client letter)

Wall time: 3 min 42 sec
```

## Failure handling

| Condition | Behavior |
| --- | --- |
| ARP discovery requires sudo and none available | Fall back to ping discovery; warn that inventory is incomplete; continue. |
| `nmap` missing | Fatal `missing_binary` — abort. |
| One per-host scan fails or times out | That host's section in the report records the error; other hosts continue. |
| One sub-agent fails | That host's `vuln_findings` entry records the error; other hosts' findings merge normally. |
| `./reports/` not writable | Emit the merged JSON into the conversation as fallback. |

## Tuning knobs (for very specific situations)

If the defaults don't fit:

- **Very large networks (/16, /22)** — pass `port-set=top-100` and reduce parallelism manually (set `--max-parallelism` in the nmap invocations) so you don't overwhelm a switch.
- **Slow / lossy links** — use `timing_profile=wan` (not currently exposed as a command flag; invoke `port-scanner` directly for that)
- **Privileged scans needed** (SYN scan, OS detection) — invoke `port-scanner` per host with `privileged=true` instead of using this command. This command stays unprivileged for portability.

## Example

```
/netd:vuln-scan 192.168.1.0/24
/netd:vuln-scan 192.168.1.0/24 iot-curated
/netd:vuln-scan 10.0.0.0/24 top-100 scan-all
/netd:vuln-scan 192.168.1.1,192.168.1.100,192.168.1.164 top-1024 no-letter
```

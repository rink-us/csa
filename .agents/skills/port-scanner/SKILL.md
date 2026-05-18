---
name: port-scanner
description: Scan TCP/UDP ports on a single host and identify the services listening on open ports. Use when the user wants to know which ports are open on a host, what's running on them, or as the first step before service enumeration or TLS analysis. Wraps nmap.
license: MIT
compatibility: Requires nmap >= 7.80. Privileged variants (-sS SYN scan) require root/sudo.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Scan TCP/UDP ports on a single host and return a normalized list of `Port` records (per [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md)).

This skill is the foundation for chained workflows: its `Port` output feeds directly into `service-enumerator` and `tls-analyzer`. Defers deep banner-grabbing and CVE correlation to `service-enumerator`; defers multi-host discovery to `network-mapper`.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target` | string | — | Single host: IPv4, IPv6, or hostname |
| `port_set` | enum | `"top-1024"` | `"top-100"` (nmap `-F`, fastest), `"top-1024"` (default), `"iot-curated"` (see below), `"full"` (1-65535) |
| `port_range` | string | derived from `port_set` | Overrides `port_set`. Free-form, e.g. `"1-1024"`, `"80,443,8080"`, `"1-65535"` |
| `protocol` | enum | `"tcp"` | `"tcp"`, `"udp"`, or `"both"` |
| `privileged` | bool | `false` | If `true`, use `-sS` SYN scan (requires sudo) |
| `discovery_mode` | enum | `"default"` | `"default"` lets nmap do host discovery first; `"force"` adds `-Pn` to skip discovery — use when the host was already confirmed up via ARP/another means and TCP/ICMP probes would just timeout |
| `timing_profile` | enum | `"lan"` | `"lan"` (aggressive, local-network defaults below), `"wan"` (conservative, retain defaults), `"stealth"` (`-T2`, paced) |
| `host_timeout_seconds` | int | `120` | Per-host wall-time cap. Bounds the worst case when a host silently drops everything. |
| `timeout_seconds` | int | `300` | Max total runtime cap |

If `target` is missing or `port_range`/`port_set` is malformed, refuse the request with a validation error.

### Port set definitions

| `port_set` | Resolves to |
| --- | --- |
| `top-100` | `nmap -F` (no `-p` needed) — top 100 by frequency. ~10× faster than top-1024. |
| `top-1024` | `-p 1-1024` |
| `iot-curated` | `-p 22,23,53,80,81,443,554,1900,5000,5353,7001,8000,8008,8080,8081,8443,8888,9100,32400` — admin UIs, RTSP cameras, SSDP/UPnP, IPP/printer, Plex |
| `full` | `-p 1-65535` |

### Timing profiles

| `timing_profile` | Extra flags appended to nmap invocation |
| --- | --- |
| `lan` (default) | `-T4 --max-rtt-timeout 200ms --min-parallelism 64 --max-retries 1` |
| `wan` | `-T3` (nmap default; safe for arbitrary internet targets) |
| `stealth` | `-T2 --max-retries 1` (paced; useful for pentest mode where detection footprint matters) |

`--host-timeout` is appended from `host_timeout_seconds` regardless of profile.

## Invocation

**Unprivileged default (works without sudo):**

```sh
nmap -sT -sV --version-intensity 5 <port flags from port_set/range> \
     <timing flags from timing_profile> \
     --host-timeout <host_timeout_seconds>s \
     <-Pn if discovery_mode=force> \
     -oX - <target>
```

UDP adds `-sU`; both-protocol adds `-sT -sU`.

**Privileged variant (opt-in via `privileged: true`):**

```sh
sudo nmap -sS -sV --version-intensity 5 <port flags> <timing flags> \
     --host-timeout <host_timeout_seconds>s -oX - <target>
```

Before recommending the privileged variant, confirm the operator has sudo and accepts the higher detection footprint.

## Performance pattern: parallel nmap per host

This skill is single-target by design. When scanning many hosts on a network where most silently drop probes (typical of consumer IoT), **callers should launch one nmap process per host in parallel** rather than passing all hosts to one nmap invocation. Reason: nmap's internal congestion control serializes timeouts across hosts in a batch; separate processes parallelize the timeout-waiting and reduce wall time from `N × per-host-worst-case` to `max(per-host-worst-case)`.

The `/netd:vuln-scan` slash command implements this pattern.

## Anti-pattern: aggressive fingerprinting against embedded devices

Combining `--version-intensity 9` (which tries every probe in nmap-service-probes against every open port) with `-sC` (default NSE scripts including http-enum, ssl-cert, etc.) **frequently exhausts `--host-timeout` against embedded admin UIs** (consumer routers, printers, IoT controllers, ISP-managed CPE). These devices respond to probes very slowly — often hundreds of ms per request — and intensity 9 sends dozens of probes per open port. Even on just a handful of known-open ports, the total can exceed a 4-minute timeout.

Observed real-world failure: deep-probing 3 embedded hosts in parallel (Verizon router + Verizon mesh AP + Canon printer) with `--version-intensity 9 -sC --host-timeout 240s -p <known-open ports>` hit the timeout on all three; nmap returned no port data.

**Better approach for embedded admin UIs:** skip nmap's heavy fingerprinting and use direct banner grabs via the [service-enumerator skill](../service-enumerator/SKILL.md). Specifically:

| Goal | Use instead of nmap |
| --- | --- |
| HTTP Server header + security headers | `curl -sI --max-time 5 http://<host>:<port>/` |
| TLS certificate + cipher | `openssl s_client -connect <host>:<port> -showcerts < /dev/null` |
| DNS server version | `dig +short +time=3 chaos txt version.bind @<host>` |
| IPP server presence | `curl -sI --max-time 5 http://<host>:631/` (405 confirms IPP) |
| Raw protocol greeting (SSH, SMTP, FTP, LPR) | `printf '' \| nc -w 3 <host> <port>` |

These typically complete in seconds with reliable output. Reserve `nmap -sV --version-intensity 9` for known-responsive servers (full Linux/Windows hosts with mature SSH/HTTP daemons) where the deeper fingerprinting actually pays off.

If you must run nmap on embedded devices, use `--version-intensity 5` (the documented default) and skip `-sC`.

## Output normalization (nmap XML → schema)

From the XML `<host>` block, map:

- `<address addr="…" addrtype="ipv4|ipv6">` → `Host.ip`
- `<hostname name="…">` → `Host.hostname`
- `<status state="up|down">` → `Host.status`
- `<osmatch name="…">` (if `-O` was run) → `Host.os_guess`

From each `<port>` block, map:

- `portid` → `Port.port`
- `protocol` → `Port.protocol`
- `<state state="…">` → `Port.status`
- `<service name product version conf>` → `Port.service.name`, `.product`, `.version`, `.confidence` (divide nmap's 0–10 by 10)
- nmap's `-sV` banner output → `Port.service.banner.raw`, `.method = "nmap_sv"`

Wrap in the standard envelope:

```json
{
  "skill": "port-scanner",
  "target": "...",
  "started_at": "...",
  "finished_at": "...",
  "result": {
    "host": { /* Host */ },
    "ports": [ /* Port, ... */ ],
    "summary": { "open": N, "closed": N, "filtered": N }
  },
  "errors": []
}
```

## Failure paths

- **`nmap` not found** → emit a fatal `missing_binary` error per the envelope in [DEPENDENCIES.md](../network-discovery/DEPENDENCIES.md). Do not fall back to a hand-rolled socket scan.
- **Target unreachable / all ports filtered** → return `Host.status = "unreachable"` with an empty `ports` array. This is not an error.
- **Timeout reached** → return whatever ports were enumerated before the timeout with a non-fatal `error: { "code": "timeout", "message": "scan exceeded N seconds" }`.

## Example invocations

```
port-scanner: target=scanme.nmap.org, port_range=1-1024
port-scanner: target=10.0.0.5, port_range=22,80,443,3306, protocol=tcp
port-scanner: target=fd00::1, port_range=1-100, protocol=both, privileged=true
```

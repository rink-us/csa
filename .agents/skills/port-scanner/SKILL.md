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
| `port_range` | string | `"1-1024"` | e.g. `"1-1024"`, `"80,443,8080"`, `"1-65535"` |
| `protocol` | enum | `"tcp"` | `"tcp"`, `"udp"`, or `"both"` |
| `privileged` | bool | `false` | If `true`, use `-sS` SYN scan (requires sudo) |
| `timeout_seconds` | int | `300` | Max total runtime |

If `target` is missing or `port_range` is malformed, refuse the request with a validation error in the result envelope.

## Invocation

**Unprivileged default (works without sudo):**

```sh
nmap -sT -sV --version-intensity 5 -oX - <target> -p <port_range>
```

UDP adds `-sU`; both-protocol adds `-sT -sU`.

**Privileged variant (opt-in via `privileged: true`):**

```sh
sudo nmap -sS -sV --version-intensity 5 -oX - <target> -p <port_range>
```

Before recommending the privileged variant, confirm the operator has sudo and accepts the higher detection footprint.

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

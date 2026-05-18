---
name: network-mapper
description: Discover active hosts across a network range (CIDR or list), classify them by likely device type, and build a topology graph. Use when the user wants an inventory of a subnet — e.g. "what's on 10.0.0.0/24" — or wants to merge port/service/TLS results from the other skills into a single connected map.
license: MIT
compatibility: Requires nmap >= 7.80. ARP discovery (-PR) requires root and same broadcast domain as target. OS classification benefits from sudo but isn't required.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Sweep a network scope, identify live hosts, classify each by device type using port/banner heuristics, and emit a topology of `TopologyNode` + `TopologyEdge` records (per [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md)).

This is the composer skill: when fed the outputs of `service-enumerator` and `tls-analyzer` via `correlate_with`, it merges them into the per-node records so the topology carries full service/TLS context.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `scope` | string\|array | — | CIDR (`10.0.0.0/24`), range (`10.0.0.1-10.0.0.50`), single IP, or array of any of these |
| `discovery_method` | enum | `"ping"` | `"ping"` (ICMP/`-PE`, unprivileged) or `"arp"` (`-PR`, privileged, same-subnet only) |
| `device_classification` | bool | `true` | Run light port-fingerprint probe on each up host |
| `classify_first` | bool | `true` | Run device-class identification (PTR + MAC vendor + hostname pattern) BEFORE port-fingerprinting. Lets the caller skip port-scanning known phone-home device classes entirely. |
| `skip_classes` | array | `["phone","vendor_cloud_iot"]` | Device classes whose port-scan phase should be skipped because they don't expose local services. Set to `[]` to scan everything. |
| `correlate_with` | object | `null` | Prior outputs from other skills — see "Correlation" |
| `allow_wide_scope` | bool | `false` | Required to scan wider than /16 |
| `timeout_seconds` | int | `900` | 15 min cap for a /24 |

If `scope` is missing or malformed, refuse with a validation error.

## Device classification taxonomy

`classify_first` produces a `device_class` per host using three signals: PTR hostname (from reverse DNS), MAC OUI vendor (from ARP discovery), and open-port pattern (from fingerprint probe if `device_classification=true`).

| `device_class` | Identifying signals (any one matches) | Default scan behavior |
| --- | --- | --- |
| `gateway` | IP equals detected default gateway | Always scan (treat as router admin surface) |
| `router_or_ap` | MAC vendor in {Arcadyan, Netgear, ASUS, TP-Link, Linksys, Cisco-Linksys, Ubiquiti, Eero, Aruba} | Always scan |
| `printer` | MAC vendor in {Canon, HP, Brother, Epson, Xerox, Ricoh, Konica}; OR open port 9100/515/631 | Always scan (printer admin UIs commonly weak) |
| `network_device` | MAC vendor in {Cisco, Juniper, Mikrotik}; OR open SNMP 161 + 22/443 | Always scan |
| `server` | PTR hostname matches `*.local` AND has 22/80/443; OR explicit hostname containing `srv`, `server`, `nas`, `prox`, `vmware`, `synology`, `qnap` | Always scan |
| `phone` | PTR matches `Pixel-*`, `*-iPhone`, `iPhone-*`, `Galaxy-*`, `*-Android`; OR MAC is locally-administered AND no open ports in fingerprint probe | **Skipped** by default (no local services worth port-scanning) |
| `vendor_cloud_iot` | PTR matches `amazon-*`, `echo-*`, `Nest-*`, `*Chromecast*`, `OwletCam-*`, `lwip*`, `tuya-*`, `*-meross-*`, `Shelly-*`; OR MAC vendor in {Amazon Technologies, Google, Tuya Smart, Espressif, Shenzhen Bilian, Sonos, Ring, Sengled, LIFX} | **Skipped** by default (phone-home-only architecture) |
| `apple_device` | MAC vendor is Apple AND PTR doesn't match phone patterns | Scan (could be a Mac with services) |
| `iot_camera` | PTR matches `*-cam-*`, `*Cam*`, `Hikvision*`, `Dahua*`; OR open 554 (RTSP) | Always scan (cameras frequently exposed) |
| `unknown` | None of the above match | Scan (better to scan unnecessarily than miss a real service) |

The classification table is heuristic and updates incrementally — when new vendor or hostname patterns become common, append to the relevant row. Conservatism rule: **when uncertain, classify as `unknown` so the caller still scans the host**. Skipping should require high-confidence identification, not absence of contradicting evidence.

The classification is emitted in `TopologyNode.device_class` (in addition to the existing `device_type` field) so the caller can filter the node list before launching port scans.

## Scope validation

Before invoking nmap:

- Reject `/0` outright.
- Reject IPv4 prefixes shorter than `/16` (i.e. `/15`, `/14`, …, `/1`) UNLESS `allow_wide_scope=true`. Surface a `scope_too_wide` error explaining the override.
- For IPv6, reject prefixes shorter than `/112` unless `allow_wide_scope=true` (a /112 is 65k addresses — already very large).
- Multi-element `scope` arrays: validate each independently; reject the whole request if any element fails.

## Step 1: host discovery

**Unprivileged (default):**

```sh
nmap -sn -PE -n --max-retries 1 -oX - <scope>
```

`-sn` no port scan, `-PE` ICMP echo, `-n` no DNS, `--max-retries 1` keep it brisk.

**Privileged ARP (opt-in via `discovery_method=arp`):**

```sh
sudo nmap -sn -PR -n -oX - <scope>
```

ARP is faster and more accurate but only works on the local broadcast domain. Confirm the operator has sudo and is on-subnet before recommending.

Parse the XML `<host>` blocks for `<status state="up">` and emit one `Host` record per. Down hosts are not included in the topology.

## Step 2: device classification (`device_classification=true`)

For each up host, run a fast targeted scan to gather classification signals:

```sh
nmap -sT -F --version-light -oX - <host>
```

(`-F` = top-100 ports; light version detection.)

Heuristics — first match wins; `device_confidence` reflects how clean the match is:

| Pattern | `device_type` | `confidence` |
| --- | --- | --- |
| 22 open AND (80 or 443) open AND OS hint matches Linux | `linux_server` | 0.85 |
| 445 AND 3389 open | `windows_workstation` | 0.85 |
| 445 AND 88 open | `windows_dc` | 0.90 |
| 161 (SNMP) open AND (22 or 443) | `network_device` | 0.75 |
| 80 only, banner contains `cam`/`hikvision`/`axis` | `iot_camera` | 0.70 |
| 9100 open | `printer` | 0.85 |
| 53 open (TCP+UDP if scanned) AND no 22 | `dns_appliance` | 0.65 |
| Anything else | `unknown` | 0.30 |

`role` defaults to `"host"`. The gateway-detection step below upgrades some to `"gateway"`/`"router"`.

## Step 3: topology construction

Build edges:

- Detect the local gateway with `ip route get 1.1.1.1 2>/dev/null \| awk '/via/ {print $3}'` (Linux) or `route -n get default 2>/dev/null \| awk '/gateway/ {print $2}'` (macOS).
- For each discovered host in the same /24 as the gateway, emit a `TopologyEdge { source: host.ip, target: gateway.ip, type: "default_route" }`.
- For hosts in the same subnet as each other (subnet derived from the scope CIDR), emit `same_subnet` edges only between distinct hosts that are NOT the gateway, and cap to a representative sample (don't emit N² edges for a /24).
- Mark the detected gateway's `TopologyNode.role = "gateway"`.

If `ip`/`route` aren't available or `correlate_with.traceroute` data is present, prefer the latter — emit `observed_hop` edges from each traceroute's adjacent hop pairs.

## Step 4: correlation with other skills

`correlate_with` is an object that may contain results from any of the other skills, keyed by skill name:

```json
{
  "correlate_with": {
    "network-reconnaissance": { /* envelope from that skill */ },
    "port-scanner": [ /* one envelope per host */ ],
    "service-enumerator": [ /* one envelope per host */ ],
    "tls-analyzer": [ /* one envelope per host:port */ ]
  }
}
```

Merge rules — for each `TopologyNode` matched by IP:

- `port-scanner` → set `node.ports = envelope.result.ports`
- `service-enumerator` → for each service entry, find the matching `node.ports[]` and set `port.service = entry.service`
- `tls-analyzer` → append the leaf cert to `node.certificates = []`
- `network-reconnaissance.dns_reverse` → fill in `node.host.hostname` if missing
- `network-reconnaissance.traceroute` → upgrade hop IPs to `role: "router"` if they appear as intermediate hops

When merging, prefer the correlated data over `network-mapper`'s own classification probe (skip Step 2 for nodes already covered by a correlated `port-scanner` result).

## Step 5: incremental output

Don't buffer the entire topology to the end. Emit `TopologyNode` records to the agent as each host's discovery + classification completes, with the final envelope summarizing edges and totals.

Practically: emit a NDJSON-style stream of partial results during the run, then a single envelope at the end. For SKILL-level invocation where the agent reads one final JSON object, accumulate but log progress lines to stderr so the agent can show the user incremental status.

## Step 6: export formats

The default result envelope (`format=json` or omitted) uses the shared schema. Two additional formats are derived from it:

- `format=dot` — Graphviz DOT. Nodes labeled `<ip>\n<device_type>`; edges colored by `type` (`default_route` = grey, `same_subnet` = lightblue, `observed_hop` = orange). Roles render as shapes: gateway/router = diamond, host = box.
- `format=csv` — One row per node with columns `ip,hostname,device_type,role,open_ports,certificates,confidence`. `open_ports` is `;`-joined `port/proto`.

For DOT/CSV, the JSON envelope still includes the rendered string under `result.export.<format>` so the agent can hand it to the user verbatim.

## Result envelope (JSON)

```json
{
  "skill": "network-mapper",
  "target": "10.0.0.0/24",
  "started_at": "...",
  "finished_at": "...",
  "result": {
    "scope": "10.0.0.0/24",
    "nodes": [ /* TopologyNode, ... */ ],
    "edges": [ /* TopologyEdge, ... */ ],
    "statistics": {
      "hosts_up": 12,
      "hosts_total_in_scope": 256,
      "device_types": { "linux_server": 4, "printer": 1, "unknown": 7 }
    },
    "export": { "dot": "...", "csv": "..." }
  },
  "errors": []
}
```

## Failure paths

| Condition | Behavior |
| --- | --- |
| `nmap` not found | Fatal `missing_binary` |
| Scope too wide and `allow_wide_scope=false` | Fatal `scope_too_wide` |
| ARP requested without sudo | Fatal `requires_sudo` |
| Timeout reached | Return whatever was discovered + non-fatal `{"code":"timeout"}` |
| Gateway detection fails | Skip default-route edges; non-fatal `{"code":"gateway_unknown"}` |
| Per-host classification probe fails | Node still included with `device_type: "unknown"`; non-fatal error |

## Example invocations

```
network-mapper: scope=10.0.0.0/24
network-mapper: scope=192.168.1.0/24, discovery_method=arp
network-mapper: scope=10.0.0.0/24, correlate_with={"port-scanner":[<envelope>,...], "tls-analyzer":[<envelope>,...]}
network-mapper: scope=10.0.0.0/16, allow_wide_scope=true, device_classification=false
```

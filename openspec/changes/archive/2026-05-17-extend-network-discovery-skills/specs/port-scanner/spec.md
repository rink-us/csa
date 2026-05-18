## ADDED Requirements

### Requirement: Port scanner accepts a port_set preset input
The agent skill SHALL accept a `port_set` input with the values `top-100`, `top-1024`, `iot-curated`, or `full`. When provided, `port_set` resolves to nmap port flags as documented in the SKILL.md. A free-form `port_range` input continues to be accepted and overrides `port_set` if both are provided.

#### Scenario: top-100 preset resolves to nmap -F
- **WHEN** the operator invokes the skill with `port_set="top-100"`
- **THEN** the underlying nmap invocation includes `-F` (and no explicit `-p` flag)

#### Scenario: iot-curated preset uses the documented port list
- **WHEN** the operator invokes the skill with `port_set="iot-curated"`
- **THEN** the underlying nmap invocation uses `-p 22,23,53,80,81,443,554,1900,5000,5353,7001,8000,8008,8080,8081,8443,8888,9100,32400`

#### Scenario: port_range overrides port_set
- **WHEN** the operator provides both `port_set="top-100"` and `port_range="1-65535"`
- **THEN** the scan uses the explicit `port_range`

### Requirement: Port scanner accepts a timing_profile input for network-context-aware defaults
The agent skill SHALL accept a `timing_profile` input with the values `lan` (default), `wan`, or `stealth`. Each profile maps to a documented set of nmap timing flags (e.g. `lan` → `-T4 --max-rtt-timeout 200ms --min-parallelism 64 --max-retries 1`).

#### Scenario: LAN profile is the default
- **WHEN** the operator invokes the skill without specifying `timing_profile`
- **THEN** the nmap invocation includes `-T4`, `--max-rtt-timeout 200ms`, and `--min-parallelism 64`

#### Scenario: WAN profile retains nmap defaults
- **WHEN** the operator invokes the skill with `timing_profile="wan"`
- **THEN** the nmap invocation uses `-T3` (no aggressive overrides)

### Requirement: Port scanner accepts a discovery_mode input
The agent skill SHALL accept a `discovery_mode` input with the values `default` (nmap performs host discovery first) or `force` (passes `-Pn` to skip discovery — for hosts already confirmed up via ARP or another means).

#### Scenario: Force mode passes -Pn
- **WHEN** the operator invokes the skill with `discovery_mode="force"`
- **THEN** the nmap invocation includes `-Pn`

### Requirement: Port scanner accepts and enforces host_timeout_seconds
The agent skill SHALL accept a `host_timeout_seconds` integer input (default 120) and pass it to nmap as `--host-timeout <N>s`. The skill SHALL return whatever data was collected before the timeout fired and record a non-fatal `{"code":"host_timeout"}` error if it triggered.

#### Scenario: Default 120-second cap
- **WHEN** the operator invokes the skill without specifying `host_timeout_seconds`
- **THEN** the nmap invocation includes `--host-timeout 120s`

#### Scenario: Timeout returns partial results, not an empty XML
- **WHEN** a scan exceeds the host-timeout
- **THEN** the result envelope contains whatever ports were enumerated before the timeout AND a non-fatal `host_timeout` error entry

### Requirement: Port scanner does not default to aggressive fingerprinting against embedded device classes
The agent skill SHALL NOT use `--version-intensity 9` or `-sC` as defaults when the target host's `device_class` (if known from prior network-mapper input) is one of `gateway`, `router_or_ap`, `printer`, `iot_camera`, `vendor_cloud_iot`, or `network_device`. These devices respond too slowly to brute-force fingerprinting and will exhaust the host-timeout. The default version intensity for these classes SHALL be 5.

#### Scenario: Printer device class uses intensity 5
- **WHEN** the operator invokes the skill against a target known to be `device_class="printer"`
- **THEN** the underlying nmap invocation uses `--version-intensity 5` (not 9) and does NOT include `-sC`

#### Scenario: Unknown device class allows operator override
- **WHEN** the operator explicitly requests `version_intensity=9` against an embedded device class
- **THEN** the skill honors the override but emits a warning in the result noting the anti-pattern

#### Scenario: Server class permits intensity 9
- **WHEN** the target is not in the listed embedded classes (i.e. `device_class="server"` or unknown)
- **THEN** the operator may use intensity 9 without warning

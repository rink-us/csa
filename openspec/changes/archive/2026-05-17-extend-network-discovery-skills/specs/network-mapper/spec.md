## ADDED Requirements

### Requirement: Network mapper accepts a classify_first input
The agent skill SHALL accept a `classify_first` boolean input (default `true`) that, when true, runs device-class identification (using PTR hostname, MAC OUI vendor, and open-port fingerprint signals) BEFORE the heavy port-fingerprint phase, so the caller can skip phone-home-only device classes entirely.

#### Scenario: Classify first is the default
- **WHEN** the operator invokes the skill without specifying `classify_first`
- **THEN** device classification runs before any port-scan phase, and the skill emits a `device_class` for each discovered host before fingerprinting

#### Scenario: Classification disabled
- **WHEN** the operator passes `classify_first=false`
- **THEN** the skill skips the classification step and proceeds with the legacy device_type fingerprint flow

### Requirement: Network mapper accepts a skip_classes input
The agent skill SHALL accept a `skip_classes` array (default `["phone", "vendor_cloud_iot"]`) of device-class names whose port-scan phase SHALL be skipped (because those classes don't expose local services). Setting `skip_classes=[]` scans every host.

#### Scenario: Default skip excludes phones and IoT
- **WHEN** the operator invokes the skill with default `skip_classes`
- **THEN** hosts classified as `phone` or `vendor_cloud_iot` are inventoried but not port-scanned in the device-fingerprint phase

#### Scenario: Empty skip_classes scans everything
- **WHEN** the operator passes `skip_classes=[]`
- **THEN** every up host is port-scanned regardless of class

### Requirement: Network mapper emits a device_class per host using a named taxonomy
The agent skill SHALL emit a `device_class` field for every discovered host. The value SHALL be one of: `gateway`, `router_or_ap`, `printer`, `network_device`, `server`, `phone`, `vendor_cloud_iot`, `apple_device`, `iot_camera`, `unknown`. The classification SHALL use PTR hostname patterns, MAC OUI vendor lookups, and open-port signals — any single matching signal is sufficient.

#### Scenario: Gateway IP is classified as gateway
- **WHEN** a discovered host's IP matches the operator's default-gateway address
- **THEN** `device_class="gateway"`

#### Scenario: Pixel hostname classifies as phone
- **WHEN** the PTR hostname matches `Pixel-*`, `iPhone-*`, `*-iPhone`, `Galaxy-*`, or `*-Android`
- **THEN** `device_class="phone"`

#### Scenario: Amazon-vendor MAC classifies as vendor_cloud_iot
- **WHEN** the MAC OUI vendor is `Amazon Technologies`, `Google`, `Tuya Smart`, `Espressif`, or other listed vendor-cloud-IoT OUIs
- **THEN** `device_class="vendor_cloud_iot"`

#### Scenario: Canon-vendor MAC classifies as printer
- **WHEN** the MAC OUI vendor is `Canon`, `HP`, `Brother`, `Epson`, `Xerox`, `Ricoh`, or `Konica`, OR port 9100/515/631 is open
- **THEN** `device_class="printer"`

### Requirement: Network mapper applies the conservatism rule for uncertain classification
The agent skill SHALL classify a host as `unknown` (not as the closest plausible class) when no signal matches a class with high confidence. Reason: `unknown` hosts are scanned by default, so misclassification cannot silently hide a real service.

#### Scenario: Apple-vendor MAC alone does not force apple_device
- **WHEN** the MAC vendor is Apple and the PTR hostname doesn't match phone patterns AND the host has no other classifying signals
- **THEN** the skill MAY assign `apple_device` (Mac with services possible) but MUST NOT assign `phone` based on vendor alone

#### Scenario: No matching signals defaults to unknown
- **WHEN** no signal matches any class
- **THEN** `device_class="unknown"` (so the host still gets scanned)

### Requirement: Network mapper preserves device_class on subsequent invocations via correlate_with
The agent skill, when invoked with `correlate_with` containing a prior network-mapper output, SHALL preserve the `device_class` of each correlated host rather than reclassifying from scratch. Re-classification SHALL only occur when a host has new signals that would change its class.

#### Scenario: Correlated class is preserved
- **WHEN** the input `correlate_with.network-mapper` contains a host classified as `printer` and the current run has the same host with no new signals
- **THEN** the host's `device_class` remains `printer` in the new output

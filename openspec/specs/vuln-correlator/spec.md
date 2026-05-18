## ADDED Requirements

### Requirement: Vuln-correlator analyzes one host per invocation
The agent skill SHALL accept a single `host` record (with `ports[]` populated) and produce a per-host findings block. The skill SHALL NOT scan or probe the network — its input is the output of a prior port-scanner/service-enumerator run.

#### Scenario: Single host as input
- **WHEN** the operator invokes the skill with one `Host` record containing 4 services
- **THEN** the result is a single per-host envelope referencing only that host

#### Scenario: Empty ports array returns empty findings block
- **WHEN** the input `host.ports` is empty or missing
- **THEN** the skill returns an empty findings block (not an error — silent hosts are valid input)

### Requirement: Vuln-correlator queries CVE sources for services with concrete product+version
The agent skill SHALL filter services to those with both `service.product` AND `service.version` populated, then query a CVE source (CIRCL by default, NVD optional) for each (product, version) tuple. Lookups SHALL be parallelized — the skill SHALL NOT serialize them.

#### Scenario: CIRCL lookup for identified service
- **WHEN** a service has `product="Apache httpd"` and `version="2.4.7"`
- **THEN** the skill issues a `curl -s https://cve.circl.lu/api/search/Apache%20httpd/2.4.7` request and parses the response

#### Scenario: Source unreachable degrades gracefully
- **WHEN** the CVE source is unreachable
- **THEN** the skill emits a non-fatal `{"code":"cve_source_unreachable","service":"..."}` to the host's errors array and continues with other services

#### Scenario: CVE results are sorted and truncated
- **WHEN** a service returns 25 CVEs from CIRCL and `max_cves_per_service=10`
- **THEN** the output for that service has at most 10 CVE records, sorted by CVSS descending

### Requirement: Vuln-correlator normalizes CVE results into the shared Cve schema
The agent skill SHALL produce `Cve` records with `id`, `severity` (derived from CVSS: `<4` low, `<7` medium, `<9` high, `≥9` critical), `cvss`, `source`, `fetched_at`, and `confidence`.

#### Scenario: Severity derived from CVSS
- **WHEN** a CVE has CVSS base score 8.1
- **THEN** the emitted record has `severity="high"`

### Requirement: Vuln-correlator applies the embedded-device fallback when no version is available
For each service without both `product` AND `version`, the agent skill SHALL still analyze using the embedded-device signal set captured by `service-enumerator` (TLS vendor identification, security headers present/missing, asset-age, presence of legacy unauthenticated protocols, default SNMP community). Findings from this path go into the `findings[]` array, NOT the `cves[]` array.

#### Scenario: TLS issuer identifies vendor when HTTP hides it
- **WHEN** a host's HTTPS port returns no Server header but its TLS cert subject contains `O=Verizon`
- **THEN** the host's `findings[]` includes a finding identifying the device vendor and class

#### Scenario: Asset-age signal triggers EOL-firmware finding
- **WHEN** an HTTP banner reveals `Last-Modified: 2007` and the current year is more than 5 years later
- **THEN** the host's `findings[]` includes a finding about likely-unpatched firmware

#### Scenario: Open LPR port (515) triggers unauthenticated-protocol finding
- **WHEN** a host has port 515 open
- **THEN** the host's `findings[]` includes a finding about LPR being a legacy unauthenticated protocol

#### Scenario: SNMP default community works
- **WHEN** the input data indicates `snmp_community_public_works == true`
- **THEN** the host's `findings[]` includes a high-severity finding about default community access

### Requirement: Vuln-correlator output is structured for downstream merge
The agent skill SHALL emit a single JSON object per host containing `skill`, `host_ip`, `host_hostname`, `device_class`, `scanned_at`, `services_analyzed`, `cves_found`, `services[]` (each with `port`, `protocol`, `product`, `version`, `cves[]`, `advisories[]`), `findings[]` (narrative bullets), and `errors[]`.

#### Scenario: Standard envelope shape
- **WHEN** the skill completes for any host
- **THEN** the top-level keys present match the documented set; missing data is null or empty arrays, never absent keys

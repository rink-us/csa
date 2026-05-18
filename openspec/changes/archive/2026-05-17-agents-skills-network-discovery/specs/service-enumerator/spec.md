## ADDED Requirements

### Requirement: Service enumerator accepts target and port specification
The agent skill SHALL accept a target host and port list, and execute service enumeration to identify running services, versions, and potential vulnerabilities.

#### Scenario: Enumerate services on specified ports
- **WHEN** an agent specifies target="example.com" and ports=[22, 80, 443]
- **THEN** the skill probes each port and identifies the service

#### Scenario: Service enumeration with version detection
- **WHEN** service enumeration completes on an HTTP port
- **THEN** the result includes service_name, version, and detection confidence

### Requirement: Service enumerator performs banner grabbing
The agent skill SHALL retrieve service banners (HTTP headers, SMTP greetings, SSH banners, etc.) from open ports to identify service types and versions.

#### Scenario: HTTP banner retrieval
- **WHEN** the skill connects to port 80 on a target
- **THEN** it retrieves HTTP response headers and identifies the web server and version

#### Scenario: SSH banner retrieval
- **WHEN** the skill connects to port 22
- **THEN** it retrieves the SSH banner containing protocol version and server identifier

### Requirement: Service enumerator identifies known vulnerabilities
The agent skill SHALL cross-reference detected services and versions against vulnerability databases to identify known CVEs and potential weaknesses.

#### Scenario: CVE identification for known vulnerable version
- **WHEN** service enumeration identifies an outdated OpenSSL version
- **THEN** the result includes associated CVEs with severity levels

#### Scenario: No known vulnerabilities
- **WHEN** a detected service has no known vulnerabilities in the database
- **THEN** the result indicates clean status

### Requirement: Service enumeration supports custom probes
The agent skill SHALL allow agents to specify custom protocol payloads or probe scripts for service identification when standard detection fails.

#### Scenario: Custom payload probe
- **WHEN** an agent specifies a custom probe payload for a non-standard service
- **THEN** the skill sends the payload and parses the response

#### Scenario: Probe timeout handling
- **WHEN** a custom probe receives no response
- **THEN** the skill times out gracefully and reports the service as unresponsive

### Requirement: Service enumeration results are structured
The agent skill SHALL return service enumeration results in a normalized format with service details, detected versions, CVEs, and confidence metrics.

#### Scenario: Normalized enumeration output
- **WHEN** enumeration completes
- **THEN** results include: {target, ports: [{port, service, version, banner, cves: [{id, severity}], confidence}], timestamp}


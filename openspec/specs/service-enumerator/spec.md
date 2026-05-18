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


### Requirement: Service enumerator prefers direct banner grabs over nmap fingerprinting for embedded admin UIs
The agent skill SHALL use direct banner grabs (`curl -sI`, `openssl s_client`, `dig CHAOS TXT`, `nc -w 3`) — not `nmap -sV --version-intensity 9` — as the primary path for ports whose target host has `device_class` in `gateway`, `router_or_ap`, `printer`, `iot_camera`, `vendor_cloud_iot`, or `network_device`.

#### Scenario: Printer admin UI uses curl, not aggressive nmap
- **WHEN** the operator invokes the skill against a `printer` device_class host with port 80 open
- **THEN** the skill issues `curl -sI --max-time 5 http://<host>/` rather than `nmap -sV --version-intensity 9 -p 80`

#### Scenario: Server class still uses nmap fallback
- **WHEN** the target is a `server` device_class (full Linux/Windows host) and the per-protocol curl/nc playbook returns empty
- **THEN** the skill falls back to `nmap -sV --version-intensity 7` as documented

### Requirement: Service enumerator includes TLS certificate probing for TLS-bearing ports
The agent skill SHALL run an `openssl s_client -connect <host>:<port> -showcerts < /dev/null` probe for any port whose service is `https`, `imaps`, `smtps`, or otherwise TLS-wrapped. The output SHALL include the leaf certificate's subject and issuer organizations, validity dates, and key algorithm/size.

#### Scenario: TLS cert captured for HTTPS port
- **WHEN** the skill probes port 443 on a target with a valid certificate
- **THEN** the result includes `tls.subject_org`, `tls.issuer_org`, `tls.not_before`, `tls.not_after`, `tls.key_algorithm`, `tls.key_size`

#### Scenario: TLS handshake failure recorded
- **WHEN** the TLS probe cannot complete the handshake (e.g. broken HTTPS on an embedded device)
- **THEN** the result records `tls.handshake_result="failed"` with an interpretation note

### Requirement: Service enumerator probes DNS servers for version disclosure via CHAOS-class queries
The agent skill SHALL, for ports identified as `domain` (53), issue `dig +short +time=3 chaos txt version.bind @<target>` and `dig +short +time=3 chaos txt hostname.bind @<target>`. An empty response SHALL be recorded as `version_disclosure="hardened"` (a positive finding, not an error).

#### Scenario: Version disclosed
- **WHEN** the DNS server responds to `chaos txt version.bind`
- **THEN** the response is captured in the service banner and recorded as `version_disclosure="exposed"`

#### Scenario: No response is recorded as hardened
- **WHEN** the DNS server returns no response to the CHAOS query
- **THEN** the result records `version_disclosure="hardened"` rather than treating the silence as an error

### Requirement: Service enumerator captures security-header inventory for HTTP/HTTPS responses
The agent skill SHALL, for HTTP and HTTPS banner grabs, capture the presence or absence of the following security headers and emit them as `headers.security_headers_present` (array of present header names) and `headers.security_headers_missing` (array of absent ones): `strict-transport-security`, `content-security-policy`, `x-frame-options`, `x-content-type-options`, `referrer-policy`.

#### Scenario: Hardened admin UI shows all headers
- **WHEN** a port-443 banner includes HSTS, CSP, X-Frame-Options, X-Content-Type-Options, and Referrer-Policy
- **THEN** all five names appear in `security_headers_present` and `security_headers_missing` is empty

#### Scenario: Bare admin UI shows the gap
- **WHEN** a port-80 banner from an old printer has no security headers at all
- **THEN** all five names appear in `security_headers_missing`

### Requirement: Service enumerator captures the asset-age signal from Last-Modified headers
The agent skill SHALL, when the HTTP response includes a `Last-Modified` header, parse the year from the value and emit `headers.last_modified_year` as an integer. Downstream consumers may use this as a firmware-age signal.

#### Scenario: Old asset signals old firmware
- **WHEN** a banner includes `Last-Modified: Mon, 15 Jan 2007 07:45:41 GMT`
- **THEN** `headers.last_modified_year=2007`

### Requirement: Service enumerator captures the HTTP-to-HTTPS redirect behavior
The agent skill SHALL, for HTTP port 80 banners, record `behavior.redirect_to_https=true` when the response is a 301 or 302 with a `Location:` header beginning with `https://`. When the response serves content directly without redirecting, the field SHALL be `false`.

#### Scenario: Modern admin UI redirects
- **WHEN** port 80 returns `HTTP/1.1 301 Moved Permanently` with `Location: https://<host>/`
- **THEN** `behavior.redirect_to_https=true`

#### Scenario: Legacy admin UI serves content on HTTP
- **WHEN** port 80 returns `HTTP/1.1 200 OK` with HTML content
- **THEN** `behavior.redirect_to_https=false`

### Requirement: Service enumerator probes IPP and LPR with documented expected responses
The agent skill SHALL probe port 631 with `curl -sI --max-time 5 http://<target>:631/` (expecting a 405 Method Not Allowed, which confirms an IPP server is listening) and port 515 with `printf '' | nc -w 3 <target> 515` (expecting no banner, but presence on this port is itself the finding — legacy unauthenticated protocol).

#### Scenario: IPP confirms via 405
- **WHEN** port 631 is open and the curl HEAD probe returns 405
- **THEN** the service record indicates `name="ipp"` and `confidence=1.0`

#### Scenario: LPR open recorded even without banner
- **WHEN** port 515 is open and the raw TCP read returns nothing
- **THEN** the service record indicates `name="printer"` with a note about LPR being a legacy unauthenticated protocol

### Requirement: Service enumerator runs per-port banner grabs in parallel
The agent skill SHALL launch banner-grab probes for multiple ports on the same host in parallel (not serially). Each probe SHALL have an independent timeout (default 5 seconds for HTTP/TLS, 3 seconds for raw TCP reads).

#### Scenario: Parallel probes for multi-port host
- **WHEN** the operator invokes the skill against a host with 4 open ports
- **THEN** all 4 banner grabs run concurrently and the total skill runtime is `max(per-probe)` plus a small orchestration overhead, not `sum(per-probe)`

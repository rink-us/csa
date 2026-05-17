## ADDED Requirements

### Requirement: Port scanner skill accepts target configuration
The agent skill SHALL accept a target host address, port range, and protocol (TCP/UDP) as input parameters and validate that they are well-formed before execution.

#### Scenario: Valid port range specification
- **WHEN** an agent invokes the port-scanner skill with target="192.168.1.100" and port_range="1-1024"
- **THEN** the skill accepts the configuration and proceeds with scanning

#### Scenario: Invalid port range rejected
- **WHEN** an agent specifies port_range="1-99999" (invalid range)
- **THEN** the skill rejects the request with a validation error

### Requirement: Port scanner executes network scan
The agent skill SHALL execute a TCP/UDP port scan against the specified target and return results showing open, closed, and filtered ports.

#### Scenario: Successful port scan completes
- **WHEN** the port scanner runs against a reachable host with open ports
- **THEN** it returns a JSON array of discovered ports with status and service names

#### Scenario: Unreachable host is handled gracefully
- **WHEN** the target host is unreachable or does not respond
- **THEN** the skill returns a result indicating the host is unreachable without crashing

### Requirement: Port scan results include service identification
The agent skill SHALL attempt to identify services running on open ports and include service names, probable versions, and confidence levels in the output.

#### Scenario: Service identification on standard port
- **WHEN** a port scan finds port 22 open
- **THEN** the result includes service_name="SSH" with confidence level

#### Scenario: Service identification on non-standard port
- **WHEN** a port scan finds port 8080 open with HTTP service
- **THEN** the result indicates the discovered service with confidence metadata

### Requirement: Port scan execution is async and time-bounded
The agent skill SHALL support asynchronous execution for long-running scans and enforce a configurable timeout to prevent indefinite execution.

#### Scenario: Async scan initiated
- **WHEN** an agent initiates a port scan with async=true
- **THEN** the skill returns immediately with a scan_id for later result retrieval

#### Scenario: Scan timeout enforced
- **WHEN** a scan exceeds the configured timeout
- **THEN** the skill terminates the scan and returns partial results with a timeout indication

### Requirement: Port scan output is normalized to standard schema
The agent skill SHALL return results in a consistent JSON format regardless of underlying scanning tool, including target, timestamp, port list, and metadata.

#### Scenario: Output schema consistency
- **WHEN** the skill completes a scan
- **THEN** the output follows the schema: {target, scan_timestamp, ports: [{port, protocol, status, service_name, confidence}], summary: {open_count, closed_count, filtered_count}}


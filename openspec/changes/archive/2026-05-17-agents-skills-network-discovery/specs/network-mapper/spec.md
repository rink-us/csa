## ADDED Requirements

### Requirement: Network mapper accepts network scope configuration
The agent skill SHALL accept network CIDR ranges, host lists, or discovery parameters and validate the scope before beginning mapping operations.

#### Scenario: CIDR range specification
- **WHEN** an agent specifies scope="192.168.0.0/24"
- **THEN** the skill accepts the configuration and prepares to map the network range

#### Scenario: Invalid CIDR rejected
- **WHEN** an agent specifies an invalid CIDR like "999.999.999.999/24"
- **THEN** the skill rejects the configuration with validation error

### Requirement: Network mapper discovers active hosts
The agent skill SHALL perform host discovery using methods like ping sweeps, ARP scanning, or ICMP echo to identify active hosts within the network scope.

#### Scenario: Host discovery completes
- **WHEN** network mapping begins on a specified range
- **THEN** the skill identifies all active hosts and returns a list with IP addresses and hostnames (if resolvable)

#### Scenario: Discovery timeout
- **WHEN** discovery exceeds the configured timeout
- **THEN** the skill returns the hosts discovered so far with a partial completion indicator

### Requirement: Network mapper builds topology map
The agent skill SHALL correlate discovered hosts, routing information, and device types to construct a network topology showing connectivity and relationships.

#### Scenario: Topology map construction
- **WHEN** host discovery and reconnaissance completes
- **THEN** the skill builds a topology showing nodes (hosts), edges (connections), and device metadata

#### Scenario: Gateway and router identification
- **WHEN** mapping the network
- **THEN** the skill identifies network gateways, routers, and their relationships

### Requirement: Network mapper identifies device types
The agent skill SHALL classify discovered devices by type (server, router, switch, workstation, IoT) based on network behavior, banners, and fingerprinting.

#### Scenario: Device type classification
- **WHEN** analyzing a discovered host
- **THEN** the result includes device_type with confidence level (e.g., device_type="Linux Server", confidence=0.95)

#### Scenario: Unknown device handling
- **WHEN** a device cannot be classified
- **THEN** the result includes generic device_type and available fingerprinting data

### Requirement: Network mapper output formats support visualization
The agent skill SHALL export network topology in multiple formats (JSON, DOT/Graphviz, CSV) suitable for visualization and integration with network management tools.

#### Scenario: JSON topology export
- **WHEN** mapping completes
- **THEN** the skill exports topology as structured JSON with nodes and edges

#### Scenario: Graphviz format export
- **WHEN** exporting for visualization
- **THEN** the skill can output DOT format compatible with Graphviz rendering

### Requirement: Network mapping is incremental and async
The agent skill SHALL support incremental mapping where results are returned as hosts are discovered, and support async execution for large networks.

#### Scenario: Incremental results streaming
- **WHEN** mapping a large network
- **THEN** the skill returns discovered hosts incrementally as they are found

#### Scenario: Async mapping with callback
- **WHEN** an agent initiates async mapping
- **THEN** the skill returns a mapping_id and provides callback mechanism for result retrieval

### Requirement: Network mapper results include security context
The agent skill SHALL correlate discovered hosts with known vulnerabilities, exposed services, and other security indicators from previous scans.

#### Scenario: Vulnerability correlation
- **WHEN** topology includes a host with known vulnerabilities
- **THEN** the result marks the host with associated CVE/vulnerability information

#### Scenario: Service risk assessment
- **WHEN** identifying services on devices
- **THEN** the topology includes security risk indicators based on service versions

### Requirement: Network map output is normalized
The agent skill SHALL return topology and device information in a consistent schema suitable for downstream processing and integration.

#### Scenario: Normalized topology output
- **WHEN** mapping completes
- **THEN** results follow schema: {network_range, discovered_timestamp, nodes: [{ip, hostname, device_type, services, vulnerabilities}], edges: [{source, target, type}], statistics: {host_count, device_types}}


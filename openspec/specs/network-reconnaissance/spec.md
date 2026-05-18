## ADDED Requirements

### Requirement: Network reconnaissance skill supports DNS resolution
The agent skill SHALL resolve domain names to IP addresses and perform reverse DNS lookups, returning results with record types, TTL values, and nameservers.

#### Scenario: Forward DNS resolution
- **WHEN** an agent queries DNS for "example.com"
- **THEN** the skill returns A/AAAA records with resolved IP addresses and TTL values

#### Scenario: Reverse DNS lookup
- **WHEN** an agent performs reverse DNS on "192.168.1.1"
- **THEN** the skill returns the associated domain name or indicates no reverse record exists

### Requirement: Network reconnaissance skill performs WHOIS lookups
The agent skill SHALL query WHOIS information for IP addresses and domain names, returning registrar, registration date, administrative contact, and nameserver information.

#### Scenario: WHOIS lookup for domain
- **WHEN** an agent requests WHOIS data for "example.com"
- **THEN** the skill returns registration information including registrar, registrant, nameservers, and expiration date

#### Scenario: WHOIS lookup for IP
- **WHEN** an agent requests WHOIS for "192.0.2.1"
- **THEN** the skill returns ASNI/RIR registration information and organization details

### Requirement: Network reconnaissance skill executes traceroute
The agent skill SHALL trace the network path to a target host, returning hop-by-hop information including IP addresses, latency, and AS numbers.

#### Scenario: Traceroute to reachable host
- **WHEN** an agent traces the path to "example.com"
- **THEN** the skill returns an ordered list of hops with IP, hostname (if resolvable), and latency in milliseconds

#### Scenario: Traceroute to unreachable host
- **WHEN** the destination is unreachable or blocked
- **THEN** the skill returns the hops reached before timeout and indicates incomplete trace

### Requirement: Network reconnaissance results are cached to reduce external queries
The agent skill SHALL cache DNS, WHOIS, and traceroute results with configurable TTL to minimize redundant external queries and reduce latency for repeated lookups.

#### Scenario: Cache hit on repeated query
- **WHEN** an agent queries the same domain twice within cache TTL
- **THEN** the second query returns cached results immediately without external lookup

#### Scenario: Cache expiration
- **WHEN** cached data expires (TTL elapsed)
- **THEN** the next query performs a fresh external lookup

### Requirement: Network reconnaissance output is normalized
The agent skill SHALL return results in a consistent JSON schema including source query, timestamp, result type, and nested data structures.

#### Scenario: Normalized output
- **WHEN** the skill completes a reconnaissance operation
- **THEN** the output follows the schema: {query_type, query, timestamp, result, cache_hit, ttl_seconds}


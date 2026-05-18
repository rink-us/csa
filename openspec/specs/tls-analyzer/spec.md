## ADDED Requirements

### Requirement: TLS analyzer connects to target and retrieves certificate
The agent skill SHALL connect to a target host on a specified port, retrieve the presented TLS certificate chain, and validate the chain structure.

#### Scenario: Certificate retrieval from HTTPS port
- **WHEN** an agent specifies target="example.com" and port=443
- **THEN** the skill connects, retrieves the certificate chain, and parses the certificates

#### Scenario: Invalid certificate chain
- **WHEN** the target presents an invalid or incomplete certificate chain
- **THEN** the skill reports the chain defects

### Requirement: TLS analyzer extracts certificate details
The agent skill SHALL parse certificate fields including subject, issuer, validity dates, public key algorithm, key size, and SAN (Subject Alternative Names).

#### Scenario: Certificate field extraction
- **WHEN** certificate parsing completes
- **THEN** the result includes: {subject, issuer, not_before, not_after, key_algorithm, key_size, san_list, serial}

#### Scenario: Wildcard certificate detection
- **WHEN** a certificate includes wildcard subjects
- **THEN** the result flags wildcard usage and lists affected domains

### Requirement: TLS analyzer identifies certificate weaknesses
The agent skill SHALL detect security issues including expired certificates, weak key sizes, deprecated algorithms, self-signed certificates, and missing hostname validation.

#### Scenario: Expired certificate detection
- **WHEN** analyzing an expired certificate
- **THEN** the result flags expiration status and days overdue

#### Scenario: Weak key size detection
- **WHEN** a certificate uses RSA-1024
- **THEN** the result flags the weak key size as a security concern

### Requirement: TLS analyzer tests protocol and cipher configuration
The agent skill SHALL test the target's TLS/SSL protocol versions and cipher suites to identify weak, deprecated, or misconfigured protocols.

#### Scenario: Protocol version detection
- **WHEN** the skill tests TLS protocols on a target
- **THEN** it identifies supported versions (SSLv3, TLSv1.0, TLSv1.2, TLSv1.3) and flags deprecated versions

#### Scenario: Cipher suite analysis
- **WHEN** analyzing supported ciphers
- **THEN** the result categorizes ciphers as strong, weak, or deprecated with justification

### Requirement: TLS analyzer computes certificate transparency status
The agent skill SHALL check if the certificate is logged in Certificate Transparency logs and verify OCSP stapling support.

#### Scenario: CT log verification
- **WHEN** analyzing a certificate
- **THEN** the result indicates CT log status and SCT (Signed Certificate Timestamp) presence

#### Scenario: OCSP stapling check
- **WHEN** testing a target
- **THEN** the result indicates whether OCSP stapling is enabled and response validity

### Requirement: TLS analysis output is structured
The agent skill SHALL return TLS analysis in a normalized format including certificate details, detected weaknesses, protocol status, and security score.

#### Scenario: Normalized TLS output
- **WHEN** analysis completes
- **THEN** results include: {target, port, certificates: [{subject, issuer, validity, weakness_flags}], protocols: [{version, supported, deprecated}], ciphers: [{name, strength}], security_score, timestamp}


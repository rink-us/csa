## 1. Shared Foundations (build these first — every skill references them)

- [x] 1.1 Create `.agents/skills/network-discovery/` parent directory (groups the five skills and the shared docs)
- [x] 1.2 Write `.agents/skills/network-discovery/OUTPUT_SCHEMAS.md` defining the shared record shapes: `Host`, `Port`, `Service`, `Banner`, `Certificate`, `Cve`, `TopologyNode`, `TopologyEdge` (so skills chain cleanly)
- [x] 1.3 Write `.agents/skills/network-discovery/DEPENDENCIES.md` listing required CLIs (`nmap`, `dig`/`drill`, `whois`, `traceroute`/`mtr`, `openssl`, `curl`), minimum versions, install commands for macOS (brew) and Linux (apt/dnf), and which operations require root

## 2. port-scanner skill

- [x] 2.1 Create `.agents/skills/port-scanner/SKILL.md` with YAML frontmatter (`name`, `description`, `license`, `compatibility` listing nmap)
- [x] 2.2 Write trigger description in frontmatter — match how the existing repo skills phrase their description (see [openspec-propose/SKILL.md](.agents/skills/openspec-propose/SKILL.md) for the pattern)
- [x] 2.3 Document inputs: `target` (single host, IPv4/IPv6/hostname), `port_range` (default `1-1024`), `protocol` (`tcp`/`udp`/`both`), optional `privileged` opt-in
- [x] 2.4 Document the unprivileged-default invocation (`nmap -sT -sV --version-intensity 5 -oX - <target> -p <range>`) and the privileged variant (`-sS`)
- [x] 2.5 Document XML→JSON normalization mapping nmap output onto the shared `Host`+`Port`+`Service` schema from `OUTPUT_SCHEMAS.md`
- [x] 2.6 Document the missing-binary path: if `nmap` not found, surface the install line from `DEPENDENCIES.md` and stop
- [x] 2.7 Document the unreachable-host path: return a `Host` record with `status: "unreachable"`, do not error the skill

## 3. network-reconnaissance skill

- [x] 3.1 Create `.agents/skills/network-reconnaissance/SKILL.md` with frontmatter and trigger description
- [x] 3.2 Document inputs: `query` (domain or IP), `operations` (subset of `["dns_forward", "dns_reverse", "nameservers", "whois", "traceroute"]`)
- [x] 3.3 Document `dig +short A`, `dig +short AAAA`, `dig -x` (reverse), `dig NS` invocations and mapping to schema
- [x] 3.4 Document `whois <target>` invocation and the parse rules for the registration/admin/nameserver fields
- [x] 3.5 Document `traceroute -n -w 2 -q 1 <target>` (Linux/macOS portable flags) and hop-record normalization
- [x] 3.6 Document the in-conversation reuse guidance: if the agent has queried this name within the current conversation, reuse the result instead of re-querying
- [x] 3.7 Document the failure paths: NXDOMAIN, WHOIS rate-limit, traceroute timeout — each maps to a typed result, not an exception

## 4. service-enumerator skill

- [x] 4.1 Create `.agents/skills/service-enumerator/SKILL.md` with frontmatter and trigger description
- [x] 4.2 Document inputs: `target`, `ports` (list — typically the output of `port-scanner`), `cve_lookup` (bool, default off)
- [x] 4.3 Document the per-protocol banner-grab playbook: HTTP (`curl -sI`), HTTPS (`curl -skI`), SSH (raw TCP read of first 256 bytes), SMTP (read greeting + `EHLO`), FTP (read greeting)
- [x] 4.4 Document the `nmap -sV --version-intensity 7 -p <ports> <target>` fallback for protocols not covered by the direct playbook
- [x] 4.5 Document the optional CVE-correlation step: lookup against an explicitly named source (e.g. NVD/CIRCL API), label confidence, emit `Cve` records per the shared schema; default off because it's slow and noisy
- [x] 4.6 Document custom-probe support: the operator may pass a `custom_probes` block (port + payload + match-regex); skill sends raw payload and reports response
- [x] 4.7 Document timeout/no-response handling: `service: "unresponsive"` record, not an error

## 5. tls-analyzer skill

- [x] 5.1 Create `.agents/skills/tls-analyzer/SKILL.md` with frontmatter and trigger description
- [x] 5.2 Document inputs: `target`, `port` (default 443), `sni` (default = target)
- [x] 5.3 Document the `openssl s_client -connect <host>:<port> -servername <sni> -showcerts < /dev/null` chain-retrieval invocation and how to split the PEM chain
- [x] 5.4 Document field extraction via `openssl x509 -noout -text -in <cert>` and the mapping onto the shared `Certificate` schema
- [x] 5.5 Document weakness detection rules: expired/expiring-soon (< 30 days), RSA < 2048, ECC < 256, SHA-1 signatures, self-signed, hostname mismatch
- [x] 5.6 Document protocol/cipher enumeration via `openssl s_client -tls1`, `-tls1_1`, `-tls1_2`, `-tls1_3` and cipher listing via `openssl ciphers -v`
- [x] 5.7 Document CT-log check via `openssl x509 -noout -ext 1.3.6.1.4.1.11129.2.4.2` and SCT presence reporting
- [x] 5.8 Document the OCSP-stapling test via `openssl s_client -status` and how to read the staple block
- [x] 5.9 Document the security-score formula (deterministic — start at 100, subtract per weakness with documented weights) and the score-→letter-grade mapping

## 6. network-mapper skill

- [x] 6.1 Create `.agents/skills/network-mapper/SKILL.md` with frontmatter and trigger description
- [x] 6.2 Document inputs: `scope` (CIDR/range/host-list), `discovery_method` (`ping` default, `arp` privileged), `device_classification` (bool), `correlate_with` (optional prior scan results)
- [x] 6.3 Document CIDR validation guidance — reject `/0`, refuse anything wider than `/16` without explicit override
- [x] 6.4 Document host-discovery invocations: `nmap -sn -PE <scope>` (unprivileged ICMP), `nmap -PR <scope>` (ARP, requires root, same broadcast domain)
- [x] 6.5 Document the device-classification heuristics: OS-family signals from banners + open-port fingerprints (e.g. 22+80 → Linux server; 445+3389 → Windows; 161+443 → likely network device)
- [x] 6.6 Document the topology-construction rules: connect each host to its detected gateway (from `ip route get`); flag gateways/routers as distinct node types
- [x] 6.7 Document the chain to `service-enumerator` and `tls-analyzer` — when `correlate_with` includes their output, merge `Service` and `Certificate` records into the `TopologyNode`
- [x] 6.8 Document the incremental-output guidance: emit `TopologyNode` records as hosts are discovered, not only at the end (so the agent can stream results)
- [x] 6.9 Document export formats: JSON (default — the shared schema), DOT/Graphviz (translation rules), CSV (one row per node)

## 7. Cross-Skill Verification

- [x] 7.1 Pick a single safe target (lab range or `scanme.nmap.org`) and dry-run each skill's prompt through the agent once to confirm it produces well-formed output — verified live on scanme.nmap.org (port-scanner, service-enumerator, network-mapper) and example.com:443 (tls-analyzer); all documented invocations produce schema-mappable output
- [x] 7.2 Run the full chain `network-reconnaissance` → `port-scanner` → `service-enumerator` → `tls-analyzer` and verify the output schemas connect (records from one feed cleanly into the next) — verified: recon resolved scanme to 45.33.32.156; port-scanner found 22/ssh + 80/http; service-enumerator banner-grabbed both; tls-analyzer ran against example.com:443 (scanme has no HTTPS); the field paths required for handoff (`Port.port`, `Service.product/version`, `Certificate.*`) are all populated by upstream skills
- [~] 7.3 Run `network-mapper` with `correlate_with` set to the chain output above and verify merge into `TopologyNode` — partial: 1-host topology demo run on scanme; no multi-host lab range was provided, so the full sweep + correlation isn't end-to-end exercised. The merge logic is JSON-shape manipulation and the upstream schemas are confirmed to provide the required fields. Re-run with a real lab range when one is available.
- [-] 7.4 Verify missing-binary behavior: temporarily rename `nmap` and confirm the skill reports cleanly with the `DEPENDENCIES.md` install hint — **skipped**: per choice during apply, "rename nmap" is a destructive system mutation. The skill's reactive missing-binary path is documented in each SKILL.md and in [DEPENDENCIES.md](../../../.agents/skills/network-discovery/DEPENDENCIES.md); run in a throwaway shell with `PATH=/tmp` if desired.

## 8. Documentation

- [x] 8.1 Add a section to the top-level [README.md](README.md) listing the new skill bundle and pointing at `.agents/skills/network-discovery/`
- [x] 8.2 Each `SKILL.md` includes a short "Example invocations" block
- [x] 8.3 `DEPENDENCIES.md` includes a one-line `brew install`/`apt install` cheatsheet covering all five skills

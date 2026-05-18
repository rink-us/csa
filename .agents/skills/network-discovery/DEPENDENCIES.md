# External CLI Dependencies

The network-discovery skills shell out to standard CLI tools. The agent SHOULD detect missing binaries reactively (run the command, catch `command not found`) and surface the install line below instead of failing silently.

## Required CLIs

| Tool | Min version | Used by | Privileged ops? |
| --- | --- | --- | --- |
| `nmap` | 7.80 | port-scanner, service-enumerator, network-mapper | `-sS`, `-PR`, `-O` need root |
| `dig` *(or `drill`)* | bind 9.16 / ldns 1.7 | network-reconnaissance | no |
| `whois` | 5.5 | network-reconnaissance | no |
| `traceroute` *(or `mtr`)* | inetutils 2.0 / mtr 0.95 | network-reconnaissance, network-mapper | raw-socket modes need root; ICMP default does not |
| `openssl` | 1.1.1 | tls-analyzer | no |
| `curl` | 7.79 | service-enumerator | no |
| `snmpget` (from `net-snmp`) | 5.7 | service-enumerator | no |

## One-line install cheatsheet

**macOS (Homebrew)**

```sh
brew install nmap bind whois mtr openssl@3 curl net-snmp
```

(`bind` ships `dig`; `mtr` covers traceroute; `openssl@3` is the modern build — Apple's bundled openssl is LibreSSL and lacks some flags this skill uses; `net-snmp` ships `snmpget`.)

**Debian / Ubuntu**

```sh
sudo apt install -y nmap dnsutils whois traceroute openssl curl snmp
```

**Fedora / RHEL**

```sh
sudo dnf install -y nmap bind-utils whois traceroute openssl curl net-snmp-utils
```

## Privileged invocations

Default skill behavior is **unprivileged**. The following variants require root/`sudo` (or POSIX capabilities `cap_net_raw`/`cap_net_admin` granted to the binary) and skills MUST only suggest them when the operator has opted in:

- `nmap -sS` (SYN scan) — port-scanner privileged variant
- `nmap -PR` (ARP host discovery) — network-mapper, same-broadcast-domain only
- `nmap -O` (OS detection) — fallback for device classification
- `traceroute` raw-socket modes (`-I`, `-T`) — default ICMP probe works unprivileged on macOS but not on most Linux distros; UDP probe is the portable unprivileged default

The skill prompt SHOULD instruct the agent to ask the operator to confirm sudo availability before recommending a privileged variant.

## Detection-fallback chain

If a tool is missing, the skill outputs (per the result envelope in `OUTPUT_SCHEMAS.md`):

```json
{
  "errors": [{
    "code": "missing_binary",
    "message": "nmap not found on PATH",
    "context": { "install_hint": "brew install nmap  # or: sudo apt install nmap" }
  }],
  "result": {}
}
```

The agent should report this directly to the operator instead of attempting a different scan method.

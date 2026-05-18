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
| `python3` | 3.8 | cve-snapshot-manager (NVD JSON parsing + indexing) | no |
| `gzip` | any (POSIX) | cve-snapshot-manager (decompressing NVD feeds) | no |

## Optional CLIs

| Tool | Used by | Notes |
| --- | --- | --- |
| `gpg` | cve-snapshot-manager (signing), vuln-correlator (signature verification) | Only consumed when the operator opts into snapshot signing (`sign-with=<key>` on `/netd:cve-snapshot`) or signature verification (`verify_signature=true` on `vuln-correlator`). Required for team-shared snapshots; not needed for single-operator use. Install via `brew install gnupg` (macOS), `apt install gnupg` (Debian/Ubuntu), or `dnf install gnupg2` (Fedora/RHEL). |

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

## Local CVE snapshot storage

The `cve-snapshot-manager` skill writes NVD CVE snapshots under `./cve-snapshots/` (repo-relative default; overridable via `CVE_SNAPSHOT_DIR` env var or skill input). Expect:

- ~500 MB compressed download per ingest run
- ~2 GB on disk after expansion + indexing
- Each ingest creates a dated subdirectory `v<YYYY-MM-DD>/`; older snapshots are not auto-pruned

**Recommendation: gitignore the `./cve-snapshots/` directory.** Snapshots are operator-managed data, large, and don't belong in version control. Add to `.gitignore`:

```
cve-snapshots/
```

`python3` and `gzip` are typically pre-installed on macOS (Python 3 since macOS 12) and on most Linux distros (Debian, Ubuntu, Fedora, RHEL ship both). No special install action is normally required; the brew/apt/dnf install lines above don't include them for that reason.

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

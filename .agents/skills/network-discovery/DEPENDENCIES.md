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

## Extended-assessment CLIs

The following CLIs are required by specific extended-assessment skills. Not needed for every engagement — install per skill need:

| Tool | Min version | Used by | Notes |
| --- | --- | --- | --- |
| `gobuster` (or `ffuf`) | 3.0+ | web-path-finder | Directory/file brute-force. Install via `brew install gobuster` (macOS), `apt install gobuster` (Debian/Ubuntu), `dnf install gobuster` (Fedora/RHEL). Pre-installed on Kali Linux. ffuf: `brew install ffuf` or `apt install ffuf`. |
| `hydra` (thc-hydra) | 9.0+ | password-attacker | Fast network logon cracker. `brew install hydra` (macOS), `apt install hydra` (Debian/Ubuntu), pre-installed on Kali Linux. |
| `sqlmap` | 1.7+ | sql-injector | SQL injection automation. `brew install sqlmap` (macOS), `apt install sqlmap` (Debian/Ubuntu), `pip3 install sqlmap`, pre-installed on Kali Linux. |
| `enum4linux-ng` | 1.0+ | windows-enumerator | SMB/RPC enumeration for Windows targets. `brew install enum4linux-ng` (macOS), `apt install enum4linux-ng` (Debian/Ubuntu), pre-installed on Kali Linux. Fallback: `impacket-smbclient` and `smbclient`. |
| `smbclient` | 4.0+ | windows-enumerator | SMB client for share listing. Part of `samba` package. `brew install smbclient` (macOS), `apt install smbclient` (Debian/Ubuntu). |
| `crackmapexec` (NetExec) | 6.0+ | windows-enumerator | Post-exploitation AD tool. `brew install crackmapexec` (macOS) or `pipx install crackmapexec`. Pre-installed on Kali Linux. |
| `ldapsearch` | 2.4+ | windows-enumerator | LDAP directory query tool. Part of `openldap` package. Pre-installed on most distros. |
| `subfinder` (optional) | 2.6+ | subdomain-enumerator | Fast passive subdomain enumeration. `brew install subfinder` (macOS), `apt install subfinder` (Debian/Ubuntu). Falls back to dig-only mode if unavailable. |
| `amass` (optional) | 4.0+ | subdomain-enumerator | In-depth DNS enumeration and network mapping. `brew install amass` (macOS), `apt install amass` (Debian/Ubuntu). Falls back to dig-only mode if unavailable. |

## Optional CLIs

| Tool | Used by | Notes |
| --- | --- | --- |
| `gpg` | cve-snapshot-manager (signing), vuln-correlator (signature verification) | Only consumed when the operator opts into snapshot signing (`sign-with=<key>` on `/netd:cve-snapshot`) or signature verification (`verify_signature=true` on `vuln-correlator`). Required for team-shared snapshots; not needed for single-operator use. Install via `brew install gnupg` (macOS), `apt install gnupg` (Debian/Ubuntu), or `dnf install gnupg2` (Fedora/RHEL). |

## Pentest tooling

The `/pentest:engagement` workflow requires additional dependencies beyond the network-discovery defaults. These are NOT installed automatically when you install the rest of the bundle — they're needed only when running pentest engagements.

| Tool | Required? | Used by | Notes |
| --- | --- | --- | --- |
| `msfconsole` (Metasploit Framework) | required for pentest | exploit-correlator | The standard exploitation framework. Install via `brew install metasploit` (macOS), `apt install metasploit-framework` (Debian/Ubuntu), or use the upstream installer on Fedora/RHEL. **Canonical environment: Kali Linux**, which ships Metasploit pre-installed. |
| `impacket` (Python toolkit) | required for pentest | exploit-correlator (Windows-targeting modules) | A collection of Python classes for working with network protocols. Install via `brew install impacket` (macOS) or `apt install python3-impacket` (Debian/Ubuntu). Pre-installed on Kali Linux. |
| `searchsploit` (from `exploitdb` package) | optional for pentest | exploit-correlator (ExploitDB lookup) | CLI front-end to the ExploitDB index. Install via `apt install exploitdb` (Debian/Ubuntu). On macOS, install via `brew install exploitdb`. Pre-installed on Kali Linux. Without `searchsploit`, the ExploitDB candidate-lookup step is skipped. |
| `wpscan` (Ruby gem) | optional for pentest | exploit-correlator (WordPress targets) | WordPress security scanner. **Most common gap on macOS** — not packaged by Homebrew. Install via `gem install wpscan`, which requires Ruby 2.7+ (macOS's system Ruby is 2.6; run `brew install ruby` first and prepend the brew Ruby to PATH). On Debian/Ubuntu: `apt install wpscan`. Pre-installed on Kali Linux. Alternative: `docker pull wpscanteam/wpscan`. Needs a free WPScan API token (https://wpscan.com) for vulnerability-database lookups. Without `wpscan`, WordPress plugin/theme/core CVE enumeration falls back to manual `curl` probes (slower, narrower coverage). |
| `nikto` | optional for pentest | exploit-correlator (web-server CGI / misconfig scan) | Long-running web server scanner that detects outdated software, dangerous files, misconfigurations. Install via `brew install nikto` (macOS), `apt install nikto` (Debian/Ubuntu), `dnf install nikto` (Fedora/RHEL). Pre-installed on Kali Linux. |
| `nuclei` | optional for pentest | exploit-correlator (template-based web vuln scan) | Fast, template-based vulnerability scanner from ProjectDiscovery. Templates cover CVEs, misconfigurations, exposures. Install via `brew install nuclei` (macOS), `apt install nuclei` on newer Debian/Ubuntu, or `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest`. Pre-installed on Kali Linux. Run `nuclei -update-templates` after install. |
| SecLists (common credentials/wordlists) | optional for pentest | exploit-correlator (when exploits need credential lists) | Available at `https://github.com/danielmiessler/SecLists`. Clone to a fixed location and reference. Pre-installed at `/usr/share/seclists/` on Kali Linux. |

### Web-application pentest tooling — when each is needed

The three optional web-app tools above (`wpscan`, `nikto`, `nuclei`) cover overlapping but distinct surfaces. Don't install all three reflexively — pick based on the engagement scope:

| Scope | Recommended | Why |
| --- | --- | --- |
| WordPress-heavy target (any e-commerce/marketing site is likely WP) | `wpscan` + `nuclei` | wpscan owns WP-specific plugin/theme/core CVE lookups; nuclei catches generic web issues |
| Generic web property (Rails, Node, custom backend) | `nuclei` + `nikto` | nuclei for current CVE templates; nikto for legacy CGI/misconfig coverage |
| Pure service-CVE engagement (no web app surface) | none of the three | msfconsole + searchsploit cover this surface |

### Recommended pentest environment

**Use a dedicated engagement machine or VM** rather than your everyday laptop. Reasons:
- Metasploit + Impacket presence on a general-purpose machine is itself a meaningful signal (an incident responder finding either on an unexpected machine investigates further)
- Pentest tooling often requires non-standard kernel modules, raw socket access, etc. — kept off your daily-driver kernel
- Engagement evidence (captured credentials, exfiltrated data samples) is high-sensitivity material that should be isolated from personal/work data
- Tooling failures (Metasploit crashes, Python conflicts) shouldn't take down your laptop

The canonical pentest environment is **Kali Linux** (free, maintained, ships every tool the playbook references). For occasional engagements, a Kali VM is sufficient; for full-time pentest work, a dedicated physical machine is common.

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

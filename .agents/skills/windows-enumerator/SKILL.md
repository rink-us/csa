---
name: windows-enumerator
description: Enumerate Windows/Active Directory hosts by probing SMB, RPC, LDAP, and Kerberos for shares, users, sessions, OS information, null session access, and common misconfigurations. Use after port-scanner identifies Windows-typical ports (135, 139, 445, 389, 636, 88, 464, 3389, 5985, 5986).
license: MIT
compatibility: Requires enum4linux-ng, smbclient, and python3-impacket for full coverage. crackmapexec (NetExec fork) optional for enhanced session enumeration. nmap for RPC endpoint mapper queries.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Enumerate Windows domain-joined hosts and Active Directory services. Designed for internal network assessments where SMB/RPC/LDAP access is available. All operations send network probes to the target.

Output uses the standard envelope from [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) and adds Windows-specific shapes documented below.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `target` | string | — | Hostname or IP of the Windows target. |
| `username` | string | `null` | Domain username for authenticated queries (domain/user format). When null, only anonymous/null-session checks run. |
| `password` | string | `null` | Password for the given username. When null and username is set, attempt NTLM hash pass-the-hash if hash format detected. |
| `domain` | string | resolved from target | Domain name for authentication. Defaults to the target's own domain if resolvable. |
| `checks` | array | `["smb_shares","null_session","os_info","rid_bruteforce","users","sessions","ldap_anonymous","rpc_endpoints","kerberos"]` | Which enumeration checks to run. Subset from the table below. |
| `timeout_seconds` | int | `60` | Per-check timeout. |
| `rid_range_start` | int | `500` | Starting RID for user brute-force (when `rid_bruteforce` is enabled). |
| `rid_range_end` | int | `2000` | Ending RID for user brute-force. |

---

## Enumeration checks

| Check | Tool | What it does |
| --- | --- | --- |
| `smb_shares` | `smbclient` + `crackmapexec` | List SMB shares and their access level (read/write/deny) |
| `null_session` | `enum4linux-ng` | Test for anonymous/null session access via SMB and RPC |
| `os_info` | `crackmapexec` / `nmap` | Extract OS version, build number, domain, hostname, signing status |
| `rid_bruteforce` | `enum4linux-ng` / `impacket-lookupsid` | Brute-force RID range to discover local/domain users |
| `users` | `enum4linux-ng` / `rpcclient` | List logged-in users, domain users, and local groups |
| `sessions` | `crackmapexec` / `impacket-netsession` | List active SMB sessions |
| `ldap_anonymous` | `ldapsearch` / `nmap` | Test for anonymous LDAP bind and dump domain info |
| `rpc_endpoints` | `nmap` | Query RPC Endpoint Mapper for running services |
| `kerberos` | `impacket-getnfp` / `nmap` | Test for Kerberos pre-auth and enumerate users via AS-REP roasting |

---

## Invocations

### `smb_shares`

```sh
smbclient -N -L //<target> -W <domain> 2>/dev/null
```

Parse the share listing. Report each share with its access level.

If `crackmapexec` is available:
```sh
crackmapexec smb <target> -u '<username>' -p '<password>' --shares 2>/dev/null
```

```json
{
  "type": "smb_shares",
  "shares": [
    { "name": "ADMIN$", "type": "DISK", "access": "denied", "remark": "Remote Admin" },
    { "name": "C$", "type": "DISK", "access": "denied", "remark": "Default share" },
    { "name": "IPC$", "type": "IPC", "access": "read", "remark": "Remote IPC" },
    { "name": "Documents", "type": "DISK", "access": "read", "remark": "Shared Documents" },
    { "name": "Backup", "type": "DISK", "access": "read_write", "remark": "Weekly backup dump" }
  ],
  "writable_shares": ["Backup"],
  "null_session_access": true
}
```

### `null_session`

```sh
enum4linux-ng -A <target> -oR /tmp/enum4linux-<target>/ 2>/dev/null
```

Parse the output for signs of successful null session:
- "Got OS information via SMB" (if present, null session works)
- "SMB signing is not enforced"
- Any user enumeration from RPC without credentials

```json
{
  "type": "null_session",
  "null_session_available": true,
  "details": "Anonymous logon successful via SMB/RPC. Can enumerate users and groups without authentication.",
  "risk": "critical"
}
```

### `os_info`

```sh
nmap -sV -p 445 --script smb-os-discovery <target>
```

Or with crackmapexec:
```sh
crackmapexec smb <target> 2>/dev/null
```

```json
{
  "type": "os_info",
  "hostname": "DC-01",
  "domain": "ACME",
  "os": "Windows Server 2022 Standard",
  "build": "20348",
  "signing": false,
  "smb1": false
}
```

### `rid_bruteforce`

```sh
impacket-lookupsid <domain>/'<username>':'<password>'@<target> <rid_range_start> <rid_range_end>
```

Or via enum4linux-ng (same data, different parser):
```sh
enum4linux-ng -U <target> -oR /tmp/enum4linux-<target>/ 2>/dev/null
```

```json
{
  "type": "rid_bruteforce",
  "range_tested": "500-2000",
  "users_found": [
    { "rid": 500, "name": "Administrator", "type": "user" },
    { "rid": 501, "name": "Guest", "type": "user" },
    { "rid": 502, "name": "krbtgt", "type": "user" },
    { "rid": 1103, "name": "svc-backup", "type": "user" },
    { "rid": 1105, "name": "john.doe", "type": "user" }
  ],
  "total_users": 5
}
```

### `users`

Check for currently logged-in sessions:
```sh
crackmapexec smb <target> -u '<username>' -p '<password>' --sessions 2>/dev/null
impacket-netsession <domain>/'<username>':'<password>'@<target>
```

```json
{
  "type": "users",
  "active_sessions": [
    { "user": "john.doe", "source_ip": "10.0.0.50", "session_type": "SMB" },
    { "user": "svc-backup", "source_ip": "10.0.0.51", "session_type": "Network" }
  ],
  "domain_admins": ["Administrator", "jane.admin"],
  "local_admins": ["ACME\Domain Admins", "ACME\svc-backup"]
}
```

### `ldap_anonymous`

```sh
nmap -p 389 --script ldap-rootdse <target>
```

Or with ldapsearch:
```sh
ldapsearch -H ldap://<target>:389 -x -b "" -s base "(objectClass=*)" 2>/dev/null
```

```json
{
  "type": "ldap_anonymous",
  "anonymous_bind_available": true,
  "root_dse": {
    "defaultNamingContext": "DC=acme,DC=local",
    "dnsHostName": "DC-01.acme.local",
    "domainFunctionality": "2016",
    "forestFunctionality": "2016"
  },
  "information_disclosed": true
}
```

### `rpc_endpoints`

```sh
nmap -sV -p 135 --script msrpc-enum <target>
```

```json
{
  "type": "rpc_endpoints",
  "endpoints": [
    { "uuid": "12345678-1234-abcd-ef00-01234567cffb", "name": "LSARPC", "protocol": "ncacn_ip_tcp", "endpoint": "10.0.0.10:49664" },
    { "uuid": "12345678-1234-abcd-ef00-01234567cffb", "name": "SAMR", "protocol": "ncacn_np", "endpoint": @"\PIPE\samr" }
  ]
}
```

### `kerberos`

Check AS-REP roastable accounts (Kerberos pre-auth disabled):

```sh
impacket-getnfp <domain>/'<user>':'<password>'@<target>
```

```json
{
  "type": "kerberos",
  "as_rep_roastable_users": [],
  "krb5_asrep_key_set": false,
  "pre_auth_disabled_users": []
}
```

---

## Output structure (merged envelope)

```json
{
  "skill": "windows-enumerator",
  "target": "10.0.0.10",
  "started_at": "2026-06-10T12:00:00Z",
  "finished_at": "2026-06-10T12:03:30Z",
  "result": {
    "os_info": { /* os_info block */ },
    "null_session": { /* null_session block */ },
    "shares": { /* smb_shares block */ },
    "users": { /* users block */ },
    "rid_enumeration": { /* rid_bruteforce block */ },
    "ldap": { /* ldap_anonymous block */ },
    "rpc": { /* rpc_endpoints block */ },
    "kerberos": { /* kerberos block */ },
    "summary": {
      "checks_run": ["smb_shares", "null_session", "os_info", "rid_bruteforce", "users", "ldap_anonymous", "rpc_endpoints", "kerberos"],
      "critical_findings": [
        "Null session available — unauthenticated access to user/group enumeration",
        "SMB signing not enforced — relay attacks possible",
        "Anonymous LDAP bind available — domain info disclosure"
      ],
      "os": "Windows Server 2022 Standard (build 20348)",
      "domain": "ACME",
      "smb_signing": false,
      "smb1_enabled": false
    }
  },
  "errors": []
}
```

---

## Failure paths

| Condition | Behavior |
| --- | --- |
| `enum4linux-ng`, `smbclient`, AND `impacket` all missing | Fatal `missing_binary` |
| Target does not have SMB ports open | Refuse to run — this host is not a Windows target (or SMB/445 is firewalled). Emit error, abort. |
| All checks return empty (Windows host but locked down) | Report that the host is hardened (no null session, no anonymous binds). This is valid output. |
| Authentication failure (bad credentials) | Report failed auth; offer to retry without credentials (null-session-only mode). |
| One check times out | That check's result entry records the timeout; continue to the next check. |

---

## Chaining with other skills

`windows-enumerator` output feeds naturally into:
- `password-attacker` — discovered usernames become spray targets
- `exploit-correlator` — discovered OS version + patch level → CVE lookup
- `vuln-correlator` — SMB signing, null session, anonymous LDAP all generate findings

---

## Example invocations

```
windows-enumerator: target=10.0.0.10
windows-enumerator: target=DC-01.acme.local, username=ACME/john.doe, password=P@ssw0rd
windows-enumerator: target=192.168.1.50, checks=["smb_shares","null_session","os_info"]
windows-enumerator: target=10.0.0.20, rid_range_start=500, rid_range_end=5000
```

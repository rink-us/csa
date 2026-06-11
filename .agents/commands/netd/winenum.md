---
name: "netd:winenum"
description: Enumerate a Windows host — SMB shares, null sessions, OS info, RID brute-force, users, LDAP, RPC endpoints, and Kerberos.
category: Network Discovery
tags: [network, windows, active-directory, smb, enum]
---

Comprehensive Windows/AD enumeration against a target with SMB/RPC/LDAP ports open. Use after `/netd:scan` reveals Windows-typical ports (135, 139, 445, 389).

**Input**: `/netd:winenum <target> [user] [pass]`

- `<target>` (required) — hostname or IP of the Windows host
- `[user]` — domain\username for authenticated checks (e.g. `ACME\john.doe`). If omitted, only anonymous/null-session checks run.
- `[pass]` — password or NTLM hash for the user. If omitted and user is set, the skill asks for one.
- `[flags]` — `checks=X,Y,Z` to run a subset (default: all). Options: `smb_shares,null_session,os_info,rid_bruteforce,users,ldap_anonymous,rpc_endpoints,kerberos`.

If no target is supplied, ask the user.

**Steps**

1. Parse `target`, optional `user`/`pass`, optional `checks` subset.
2. Read [.agents/skills/windows-enumerator/SKILL.md](../../skills/windows-enumerator/SKILL.md) and run each requested check in order:
   - `null_session` — enum4linux-ng for null-session access
   - `os_info` — nmap smb-os-discovery or crackmapexec
   - `smb_shares` — smbclient -L, crackmapexec --shares
   - `rid_bruteforce` — impacket-lookupsid or enum4linux-ng -U
   - `users` — crackmapexec --sessions, impacket-netsession
   - `ldap_anonymous` — nmap ldap-rootdse or ldapsearch
   - `rpc_endpoints` — nmap msrpc-enum
   - `kerberos` — impacket-getnfp
3. Collate all findings into a single envelope with a summary highlighting critical issues (null session, SMB signing disabled, anonymous LDAP bind).
4. If the target does not have port 445 open, refuse with a clear error — this is not a reachable Windows host.

**Example**

```
/netd:winenum 10.0.0.10
/netd:winenum DC-01.acme.local ACME\john.doe P@ssw0rd
/netd:winenum 192.168.1.50 checks=smb_shares,null_session,os_info
```

---
name: "netd:precheck"
description: Check that all CLIs required by the network-discovery skill bundle are installed, meet minimum versions, and aren't being shadowed by an inferior alternative (e.g. macOS's bundled LibreSSL hiding a brew-installed openssl@3). Surfaces install hints for any gap.
category: Network Discovery
tags: [network, precheck, dependencies, setup]
---

Audit the local system against the required external CLIs listed in [.agents/skills/network-discovery/DEPENDENCIES.md](.agents/skills/network-discovery/DEPENDENCIES.md). Report per-tool status and exact install/fix commands for anything missing or degraded.

**Input**: none. `/netd:precheck` takes no arguments.

**Steps**

1. **Identify platform**. Run `uname -s`; if `Darwin`, also capture `sw_vers`. If Linux, identify the package manager (`which apt`, `which dnf`). Use this to pick the right install snippet from [DEPENDENCIES.md](.agents/skills/network-discovery/DEPENDENCIES.md).

2. **Probe each required tool in parallel** (one Bash invocation, all checks in one script for speed):

   | Tool | Min version | Probe |
   | --- | --- | --- |
   | `nmap` | 7.80 | `nmap --version \| head -1` |
   | `dig` | bind 9.16 | `dig -v` (alternative: `drill -v`) |
   | `whois` | 5.5 | `whois -V` (BSD variant has no version flag — treat presence as OK) |
   | `traceroute` | inetutils 2.0 | `traceroute --version` (alternative: `mtr --version`) |
   | `openssl` | OpenSSL 1.1.1 | `openssl version` — **LibreSSL must be flagged as inadequate** (it lacks `-ext` for CT, breaking tls-analyzer step 6) |
   | `curl` | 7.79 | `curl --version \| head -1` |
   | `nc` | — | `which nc` (needed by service-enumerator raw-TCP reads) |

3. **Handle the openssl-shadowing case specifically**. On macOS, `openssl` on PATH is almost always Apple's bundled LibreSSL even when `openssl@3` is installed via brew. After running `openssl version`, also probe:

   ```sh
   brew --prefix openssl@3 2>/dev/null
   ```

   If brew has openssl@3 installed but `openssl version` returns `LibreSSL`, emit a `degraded_shadowed` finding pointing at the brew path and offering both fix options (PATH prepend OR have skills invoke the brew binary by absolute path).

4. **Classify each tool's status**:
   - `ok` — installed, meets minimum version
   - `outdated` — installed but below minimum (still functional, surface as warning)
   - `degraded_shadowed` — adequate version is installed but not first on PATH
   - `missing` — not found

5. **Compose the report** as a JSON envelope plus a human-readable summary table. Structure:

   ```json
   {
     "skill": "netd:precheck",
     "platform": { "os": "Darwin", "version": "26.4.1", "package_manager": "brew" },
     "checked_at": "2026-05-17T...",
     "tools": [
       { "name": "nmap", "status": "ok", "version": "7.99", "required": "7.80", "path": "/opt/homebrew/bin/nmap" },
       { "name": "openssl", "status": "degraded_shadowed", "version": "LibreSSL 3.3.6", "alternate": { "path": "/opt/homebrew/opt/openssl@3/bin/openssl", "version": "OpenSSL 3.6.2" }, "fix": "export PATH=\"/opt/homebrew/opt/openssl@3/bin:$PATH\"" },
       ...
     ],
     "ready": false,
     "blockers": [ /* tools with status=missing or critical degradation */ ],
     "warnings": [ /* outdated or shadowed */ ]
   }
   ```

6. **Print the install-hint cheatsheet** for the detected platform from [DEPENDENCIES.md](.agents/skills/network-discovery/DEPENDENCIES.md), but only the line that matches (brew on macOS, apt on Debian/Ubuntu, dnf on Fedora/RHEL). Don't dump all three.

7. **Set the bottom-line readiness verdict**:
   - `ready: true` — everything `ok`
   - `ready: true` with warnings — `outdated` or `degraded_shadowed` present but no `missing`
   - `ready: false` — at least one `missing` blocker

   Surface the verdict as the last line so the user can decide whether to fix before running a real scan.

**Guardrails**

- Don't install anything. This command only diagnoses; the user runs the fix command themselves.
- Don't call `sudo`. All probes are version queries.
- Don't read network state (no DNS, no HTTPS) — purely local binary checks.
- If `whois -V` fails with "illegal option", that's the BSD variant — treat the binary's presence as `ok`, not as a probe failure.

**Example**

```
/netd:precheck
```

Sample output (truncated):

```
Platform: macOS 26.4.1 (Darwin)

Tool         Status              Version            Notes
-----------  ------------------  -----------------  ----------------------------------------
nmap         ok                  7.99               /opt/homebrew/bin/nmap
dig          outdated            9.10.6             req bind 9.16+ — still functional
whois        ok                  (BSD)              /usr/bin/whois
traceroute   ok                  Darwin 1.4a12      /usr/sbin/traceroute
openssl      degraded_shadowed   LibreSSL 3.3.6     openssl@3 is installed at
                                                    /opt/homebrew/opt/openssl@3/bin/openssl
                                                    but PATH resolves to LibreSSL first
curl         ok                  8.7.1              /usr/bin/curl
nc           ok                  (BSD)              /usr/bin/nc

Fix (one line):
  export PATH="/opt/homebrew/opt/openssl@3/bin:$PATH"

Verdict: ready (1 warning, 0 blockers)
```

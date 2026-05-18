---
name: vuln-correlator
description: Given a single host's discovered services (output from port-scanner or service-enumerator), look up known CVEs for each detected (product, version) tuple, optionally query the vendor's recent advisories, and produce a per-host findings block. Designed to be invoked once per host — ideally in parallel as sub-agents — after a port/service scan has completed. The main loop launches N copies and merges the results.
license: MIT
compatibility: Requires curl. CVE source defaults to CIRCL (no key); NVD optional. Outbound HTTPS to cve.circl.lu or services.nvd.nist.gov.
metadata:
  bundle: network-discovery
  version: "1.0"
---

Translate a host's open-port/service findings into a vulnerability-oriented analysis block. Per-host, independent, parallelizable. Produces `Cve` records per [OUTPUT_SCHEMAS.md](../network-discovery/OUTPUT_SCHEMAS.md) plus a short narrative `findings` array for the human reader.

This skill assumes you already have the scan data. It does NOT run nmap. It runs CVE lookups (network-bound but cheap) and synthesizes the result.

---

## Inputs

| Name | Type | Default | Notes |
| --- | --- | --- | --- |
| `host` | object | — | A single `Host` record (or `TopologyNode`) with `ports[]` populated |
| `cve_source` | enum | `"circl"` | `"circl"` (cve.circl.lu, requires outbound HTTPS), `"nvd"` (services.nvd.nist.gov, requires outbound HTTPS), or `"offline"` (local NVD snapshot, no network calls) |
| `vendor_advisory_check` | bool | `false` | If `true`, attempt a best-effort WebFetch of the vendor's recent security advisories page for any identified product. Use sparingly — slow and noisy. Not available in `offline` mode. |
| `max_cves_per_service` | int | `10` | Cap on CVE records emitted per service to keep the output bounded |
| `per_lookup_timeout_seconds` | int | `15` | Per-API-call timeout (applies to circl/nvd modes only — offline mode is filesystem-local) |
| `snapshot_dir` | string | `./cve-snapshots/` | (Offline mode only) Root directory of local NVD snapshots. Overridable via env var `CVE_SNAPSHOT_DIR`. |
| `snapshot_version` | string | `null` (most-recent) | (Offline mode only) Pin a specific snapshot version `v<YYYY-MM-DD>` for reproducibility. Default uses the most-recent snapshot found under `snapshot_dir`. Mutually exclusive with `snapshot_versions`. |
| `snapshot_versions` | array | `null` | (Offline mode only) Array of snapshot versions to merge at lookup time, e.g. `["v2026-01-01", "v2026-05-15-kev-only"]`. Later entries override earlier on duplicate CVE IDs. Useful pattern: full historical + recent KEV-only refresh. |
| `max_snapshot_age_days` | int | `30` | (Offline mode only) Emit a non-fatal warning if the selected snapshot is older than this many days. |
| `verify_signature` | bool | `false` | When `true`, refuse to load a snapshot whose manifest doesn't have a valid gpg signature from a key in `trusted_signers`. Requires `gpg` on PATH. |
| `trusted_signers` | array | `[]` | gpg key IDs / fingerprints allowed as snapshot signers. Only consulted when `verify_signature=true`. |

If `host.ports` is empty or missing, return an empty findings block (not an error — silent hosts are a valid input).

## Process

1. **Filter services to those with concrete identification.** Skip ports where `service.product` OR `service.version` is null FOR THE CVE-LOOKUP PATH. CVE correlation needs both to be useful; partial data produces too many false positives. However — those skipped services still get analyzed via the "embedded-device fallback" below; don't drop them from the output entirely.

2. **For each (product, version) pair, query the CVE source in parallel** (the agent should launch the lookups concurrently — they're independent network calls).

   **CIRCL invocation:**
   ```sh
   curl -s --max-time <per_lookup_timeout_seconds> \
     "https://cve.circl.lu/api/search/<product>/<version>"
   ```

   **NVD invocation (requires vendor + product as CPE):**
   ```sh
   curl -s --max-time <per_lookup_timeout_seconds> \
     "https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=cpe:2.3:a:<vendor>:<product>:<version>"
   ```

   If the source is unreachable, emit `{"code": "cve_source_unreachable", "service": "..."}` to the host's `errors` array and continue with the other services. Do not abort the whole correlator.

   **Offline mode (`cve_source="offline"`):** instead of an HTTP request, perform a filesystem read:

   ```sh
   cat <snapshot_root>/by-product/<vendor_slug>/<product_slug>/<version>.json
   ```

   Where `<snapshot_root>` is `<snapshot_dir>/<snapshot_version>/` (e.g. `./cve-snapshots/v2026-05-17/`). Vendor/product slugs use the same normalization as the snapshot manager: lowercase, non-alphanumeric → underscore, collapse consecutive underscores. A missing file means zero known CVEs for that (vendor, product, version) — this is NOT an error.

   See "Offline lookup mode" section below for snapshot discovery, version selection, staleness check, and missing-snapshot refusal rules.

3. **Normalize results into `Cve` records.** For each CVE:
   - `id` — CVE ID
   - `severity` — derived from CVSS: `<4` low, `<7` medium, `<9` high, `≥9` critical
   - `cvss` — base score, prefer v3 if present
   - `source` — `"circl"` or `"nvd"`
   - `fetched_at` — ISO 8601 UTC
   - `confidence` — `"version_match"` (exact version reported by source matches discovered version) or `"banner_hint"` (partial match — vendor only or version-range overlap)

4. **Sort each service's `Cve` array by `cvss` descending, truncate to `max_cves_per_service`.**

5. **Optional vendor advisory check (`vendor_advisory_check=true`):** for each identified product, WebFetch the canonical advisory URL if known (e.g. `https://www.openssh.com/security.html`, `https://www.cisa.gov/known-exploited-vulnerabilities-catalog`). Extract advisory titles published within the last 90 days. Emit as `advisories[]` entries `{title, published, url}`. This is best-effort — if the page structure isn't parseable, skip silently.

### Embedded-device fallback (when no product+version is available)

Embedded admin UIs (routers, printers, IoT controllers, ISP CPE) frequently omit version info from their HTTP responses on purpose, to avoid fingerprinting. The `service-enumerator` skill captures additional signals for these cases — use them instead of returning "no findings."

For each service without `product` AND `version`, do the following analysis using whatever the service-enumerator captured:

| Signal | What to derive |
| --- | --- |
| `tls.subject_org` / `tls.issuer_org` | Vendor identity. `O=Verizon` → Verizon Fios CPE; `O=Canon`, `O=HP`, `O=Brother` → printer brand; `O=Hikvision`, `O=Dahua` → IP camera vendor. Record as `vendor_identified_by: "tls_cert"`. |
| `tls.validity_window_years` | A 50-year cert window is a strong signal of ISP-managed CPE (treat as "well-managed, low concern"). A 90-day window is typical of actively-managed services. A self-signed cert with no rotation history is a concern for end-user-managed devices. |
| `headers.security_headers_present` | Check against the standard set: HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy. Each missing one is a hardening recommendation (NOT a CVE — emit as a `finding` not as a `Cve`). |
| `headers.last_modified_year` | If a static asset header reveals `Last-Modified: <YYYY>` and that year is more than 5 years ago, treat it as a strong signal of EOL firmware. Emit a finding like "Web UI assets dated `<YYYY>` indicate firmware not updated in approximately `<N>` years; vendor likely no longer issues security patches." |
| `behavior.redirect_to_https` | If HTTP serves content directly instead of 301-redirecting to HTTPS, emit a finding for unencrypted credential submission. |
| `port == 515` AND service name is `printer` | LPR is a legacy unauthenticated protocol. Emit a finding regardless of vendor. |
| `port == 631` AND service name is `ipp` | IPP is modern but commonly ships with no LAN access control by default. Emit a finding to verify ACLs. |
| `port == 9100` | Raw printer protocol, no auth. Emit a finding to firewall or restrict. |
| `snmp_community_public_works == true` | Default community string accepted — high-severity unauthenticated-read finding. |

These findings get added to the `findings[]` array and do NOT populate the `cves[]` array (they're not CVEs). When the output is consumed downstream by `assessment-report`, these findings still feed into the recommendation derivation.

The embedded-device fallback is **`cve_source`-agnostic** — it works identically in `circl`, `nvd`, and `offline` modes because it operates entirely on data already in the input (TLS cert details, header inventory, asset-age, port presence). No CVE source is consulted for the fallback findings.

### Offline lookup mode (`cve_source="offline"`)

This mode reads CVEs from a local NVD snapshot maintained by the [cve-snapshot-manager skill](../cve-snapshot-manager/SKILL.md) (operator-invoked via [/netd:cve-snapshot](../../commands/netd/cve-snapshot.md)). Designed for engagements where outbound HTTPS to CIRCL/NVD is blocked, unreliable, or against the engagement contract.

#### Snapshot discovery

The agent SHALL resolve the snapshot root in this priority order:

1. Explicit `snapshot_dir` input → use that path
2. `CVE_SNAPSHOT_DIR` environment variable → use that path
3. Default: `./cve-snapshots/` (relative to the caller's cwd)

Inside that directory, snapshots live as subdirectories `v<YYYY-MM-DD>/`. The agent SHALL select the most-recent (lexically-sorted last) `v<YYYY-MM-DD>` subdirectory UNLESS the operator pinned `snapshot_version` for reproducibility.

#### Staleness check

After selecting the snapshot, read `<snapshot_root>/manifest.json` and compare `snapshot_version` to the current date (UTC). If the snapshot is older than `max_snapshot_age_days` (default 30), emit a non-fatal warning into the output envelope's `errors[]`:

```json
{
  "code": "snapshot_stale",
  "snapshot_version": "v2026-04-01",
  "age_days": 46,
  "threshold_days": 30,
  "message": "Snapshot is older than the configured freshness threshold. CVE data may miss recent disclosures. Run /netd:cve-snapshot to refresh."
}
```

Continue with the lookup — staleness is a warning, not a refusal.

#### Lookup invocation

Parallel filesystem reads, one per (vendor, product, version) tuple, exactly as documented in the CVE correlation step above. Read failures (file missing) mean zero CVEs for that tuple — record nothing, no error.

#### Output: snapshot version reporting

When `cve_source="offline"` is used, the output envelope SHALL include:

```json
{
  "cve_source_metadata": {
    "source": "offline",
    "snapshot_version": "v2026-05-17",
    "snapshot_path": "./cve-snapshots/v2026-05-17",
    "snapshot_age_days": 0,
    "snapshot_cve_count": 245712
  }
}
```

This makes the engagement reproducible — a future re-run pinned to the same `snapshot_version` produces the same CVE results.

#### Missing-snapshot refusal

If `cve_source="offline"` is selected but no snapshot exists at the resolved path (the directory is missing OR contains no `v<YYYY-MM-DD>` subdirectories), the agent SHALL refuse with:

```json
{
  "errors": [{
    "code": "snapshot_missing",
    "message": "No CVE snapshot found at <resolved_path>. Run /netd:cve-snapshot to create one.",
    "resolved_snapshot_dir": "<path>",
    "hint": "/netd:cve-snapshot"
  }],
  "result": {}
}
```

The agent SHALL NOT silently fall back to `circl` or `nvd`. Offline mode is opt-in for a reason — operators selecting it have decided the engagement should be egress-free.

#### Multi-snapshot merge (`snapshot_versions` array)

When the operator passes `snapshot_versions=["v2026-01-01", "v2026-05-15-kev-only"]`, the skill queries each in the supplied order and merges results, deduplicating CVEs by ID. **Later snapshots override earlier ones** — so the typical pattern is `[<full historical>, <recent KEV-only refresh>]`: you get January's full CVE corpus plus May's fresh KEV catalog, without re-running the full 2 GB NVD ingest.

If any listed snapshot doesn't exist, the skill refuses with `snapshot_missing` naming the missing entry — does NOT partially proceed with the snapshots that do exist.

`snapshot_version` (singular) and `snapshot_versions` (plural) are mutually exclusive; passing both is a validation error.

#### KEV (Known Exploited Vulnerabilities) check

After fetching CVEs for each service, the skill SHALL look up each CVE ID in the snapshot's `kev/index.json`. When a match is found:

- Set `cve.actively_exploited = true` on that Cve record
- Bump `cve.severity` to at least `high` (KEV CVEs are by definition under active exploitation; lower severity values are misleading and would cause the operator to deprioritize a real threat)
- Annotate the Cve record with a `kev` sub-object containing the KEV catalog fields (`vendorProject`, `product`, `vulnerabilityName`, `dateAdded`, `requiredAction`, `dueDate`)

For multi-snapshot mode, the KEV index from the last-listed snapshot wins.

If the snapshot has no `kev/index.json` (older snapshots, KEV download failed during ingest), the skill SHALL set `cve.actively_exploited = null` (unknown — not checked) and emit a single informational `errors[]` entry per host noting the missing KEV index. The skill does NOT refuse — KEV is enrichment, not a hard requirement.

In online modes (`circl`/`nvd`), the same KEV check runs if a local snapshot is available. If no local snapshot exists in online mode, the KEV check is silently skipped (`actively_exploited` stays `null`, no error emitted — online mode operators may not have chosen to maintain a local snapshot).

#### Signature verification (`verify_signature=true`)

When enabled, the skill SHALL verify the snapshot's manifest signature before loading any data:

```sh
gpg --verify <snapshot_root>/manifest.json.sig <snapshot_root>/manifest.json
```

Refusal conditions:

- Missing signature file at `<snapshot_root>/manifest.json.sig` → `{"code":"snapshot_unsigned"}`
- gpg returns non-zero (signature invalid / corrupt / wrong key) → `{"code":"signature_invalid"}`
- Signature verifies but the signing key ID is not in `trusted_signers` → `{"code":"signer_untrusted","signing_key":"<id>"}`

In all three refusal cases, the skill does NOT load the snapshot. No partial data is consumed.

Signature verification is opt-in because most single-operator setups don't need it. Enable when:
- The snapshot is shared across a team (one operator maintains the canonical snapshot, others fetch and verify)
- The operator wants a tamper-detection layer on their own snapshot

`verify_signature=true` requires `gpg` on PATH. If gpg is missing, the skill refuses with `missing_binary`.

6. **Compose a short narrative `findings` array** — 3–6 plain-language bullets summarizing what the operator should care about. Each finding ties to specific evidence:

   - "OpenSSH 6.6.1p1 (2014) is end-of-life. Known issues: CVE-2016-XXXX, CVE-2018-XXXX. Recommend upgrading to 9.x."
   - "Apache httpd 2.4.7 is from 2014. Multiple high-severity CVEs published since. Upgrade strongly recommended."
   - "Port 9100 open (raw printer protocol). Anyone on the local network can submit print jobs without authentication. Consider firewalling at the printer or restricting to a management VLAN."
   - "Admin interface on HTTP (not HTTPS) — credentials sent in plaintext over the LAN. Cleartext-credentials class issue, not a CVE."

   `findings` is for human consumption; the structured `cves[]` array is the machine-readable artifact.

## Output structure

```json
{
  "skill": "vuln-correlator",
  "host_ip": "192.168.1.164",
  "host_hostname": "canon-printer",
  "device_class": "printer",
  "scanned_at": "2026-05-17T20:30:00Z",
  "services_analyzed": 4,
  "cves_found": 7,
  "services": [
    {
      "port": 80,
      "protocol": "tcp",
      "product": "Apache httpd",
      "version": "2.4.7",
      "cves": [
        { "id": "CVE-2024-XXXXX", "severity": "high", "cvss": 8.1, "source": "circl", "confidence": "version_match", "fetched_at": "..." }
      ],
      "advisories": []
    }
  ],
  "findings": [
    "Apache httpd 2.4.7 is from 2014. ...",
    "Port 9100 open — local-network print injection possible. ..."
  ],
  "errors": []
}
```

## Sub-agent invocation pattern

This skill is shaped to be invoked **one per host, in parallel**, from a main loop (e.g. `/netd:vuln-scan` does exactly this):

1. Main loop completes scan/enum phase, ends up with `nodes[]`.
2. Filter `nodes` to those with at least one open port.
3. Spawn N parallel sub-agents (use the `Agent` tool with `subagent_type=general-purpose` or `Explore` depending on whether vendor-advisory WebFetch is wanted). Pass each agent:
   - One `host` record from the node list
   - A pointer to this SKILL.md
4. Each sub-agent runs the steps above and returns a single output block.
5. Main loop concatenates the per-host blocks into the merged report.

Each sub-agent's context stays bounded (one host's data), and N agents run in wall-time `~ max(slowest_per_host_analysis)` instead of `N × that`.

## Failure handling

| Condition | Behavior |
| --- | --- |
| `curl` missing | Fatal `missing_binary` |
| CIRCL/NVD unreachable | Non-fatal per-service error; output `cves: []` for that service; note source in `errors` |
| API returns malformed JSON | Treat as unreachable for that service; continue |
| Product name has unusual characters | URL-encode before request |
| Zero services with both product AND version | Use the "Embedded-device fallback" path above. Only return the no-version-data finding when the embedded-device signals are ALSO absent (no TLS cert info, no headers, no behavior data). |

## Example invocation (single-host, from main loop)

```
vuln-correlator:
  host = {
    "ip": "192.168.1.164",
    "hostname": "canon-printer",
    "ports": [
      { "port": 80, "service": { "product": "Apache httpd", "version": "2.4.7", "name": "http" } },
      { "port": 9100, "service": { "name": "printer", "product": null } }
    ]
  }
  cve_source = "circl"
```

## Example invocation (parallel sub-agents)

See [/netd:vuln-scan](../../commands/netd/vuln-scan.md) for the orchestration pattern.

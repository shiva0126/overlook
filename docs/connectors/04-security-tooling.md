# Domain 04 — Security Tooling

**17 connectors · 87 collectors** · [Index](00-index.md)

**Band 5 — overlays.** Almost nothing here creates nodes. These connectors attach properties, findings and `PROTECTS` edges to entities that other domains already created. That is why they run last in every cycle: run them earlier and they produce orphaned properties with nothing to attach to.

⚠ **The domain-wide rule:** an overlay connector alone produces almost no graph. A customer with only EDR and vulnerability scanners connected gets an inventory with annotations and zero attack paths.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · CrowdStrike Falcon

```
  api_surface   configuration
  functions     collect · respond (RTR — separate manifest, separate keys)
  auth          OAuth2 API client
  least priv    read scopes only: Hosts, Detections, Incidents,
                Spotlight, Discover, Zero Trust Assessment.
                RTR write scopes go in the response manifest.
  ⚠             this is where process trees come from. Our agent
                deliberately does NOT collect them (01 §12.1) —
                8,500 hosts × continuous process capture is ~2.4
                billion records/day.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `hosts` | Host inventory, OS, IPs, MAC, OU, sensor state | `ASSET`, `PROTECTS` | 4h | ★ |
| `host_logons` | Logged-on user history | `AUTHENTICATES_TO` (weak, corroborating) | 1h | ★ |
| `detections` | Detections and their process context | `FINDING` | 15m | |
| `incidents` | Correlated incidents | `FINDING` | 15m | |
| `vulnerabilities` | Spotlight CVE findings per host | `VULNERABILITY` properties | 12h | |
| `applications` | Discover software inventory | software properties, AI tooling | 12h | ★ |
| `assets_unmanaged` | Discover unmanaged/rogue assets | `ASSET`, coverage gap findings | 12h | ★ |
| `zta_scores` | Zero Trust Assessment posture | posture properties | 12h | |
| `firewall_rules` | Falcon firewall policy | `PROTECTS` | 12h | |
| `process_context` | Process trees, network connections per host | investigation context | on-demand | |
| — `rtr` | Real Time Response | **RESPOND: isolate, kill, run** | — | |

**10 collect + 1 respond.**

---

## 2 · Microsoft Defender for Endpoint

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `machines` | Device inventory, health, onboarding state | `ASSET`, `PROTECTS` | 4h | ★ |
| `logged_on_users` | Logged-on users per device | `AUTHENTICATES_TO` | 1h | ★ |
| `alerts` | Alerts and evidence | `FINDING` | 15m | |
| `incidents` | Correlated incidents | `FINDING` | 15m | |
| `tvm_vulnerabilities` | Threat & Vulnerability Management CVEs | `VULNERABILITY` | 12h | |
| `tvm_software` | Installed software inventory | software properties, AI tooling | 12h | ★ |
| `tvm_baselines` | Security baseline compliance | posture | 24h | |
| `advanced_hunting` | KQL queries for specific context | investigation context | on-demand | |
| — `respond` | Isolate, restrict, run investigation | **RESPOND** | — | |

**8 collect + 1 respond.**

---

## 3 · Microsoft Defender for Cloud

```
  ⚠ dep: azure connector. This attaches findings to Azure resources
    the cloud connector created.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `assessments` | Security assessments per resource | `FINDING` | 1h | ★ |
| `secure_score` | Secure score by control | posture summary | 12h | |
| `regulatory_compliance` | Compliance standard results | compliance mapping | 24h | |
| `alerts` | Cloud workload alerts | `FINDING` | 15m | |
| `recommendations` | Remediation recommendations | remediation text | 12h | |

**5 collectors.**

---

## 4 · SentinelOne

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `agents` | Agent/endpoint inventory, health | `ASSET`, `PROTECTS` | 4h | ★ |
| `threats` | Threat detections | `FINDING` | 15m | |
| `applications` | Application inventory | software properties | 12h | ★ |
| `vulnerabilities` | Application vulnerability findings | `VULNERABILITY` | 12h | |
| `policies` | Protection policies and exclusions | `PROTECTS`, exclusion findings | 12h | |
| `deep_visibility` | Query for process/network context | investigation context | on-demand | |
| — `respond` | Disconnect, kill, quarantine | **RESPOND** | — | |

**6 collect + 1 respond.**

---

## 5 · FortiEDR

```
  ⚠ NAMING COLLISION: FortiEDR calls its endpoint agents
    "Collectors." Their Collector = our Agent. Nothing to do with
    our collector concept.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `devices` | FortiEDR agent inventory, state, version | `ASSET`, `PROTECTS` | 4h | ★ |
| `events` | Security events and classifications | `FINDING` | 15m | |
| `policies` | Protection and exception policies | `PROTECTS`, exception findings | 12h | |
| `inventory` | System and software inventory | software properties | 12h | ★ |
| `communication_control` | Application communication policy | `PROTECTS` on egress | 12h | |
| — `respond` | Isolate device, block execution | **RESPOND** | — | |

**5 collect + 1 respond.**

---

## 6 · Palo Alto Cortex XDR

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `endpoints` | Endpoint inventory, agent state | `ASSET`, `PROTECTS` | 4h | ★ |
| `incidents` | Incidents and alerts | `FINDING` | 15m | |
| `policies` | Prevention profiles and exceptions | `PROTECTS` | 12h | |
| `assets` | Asset inventory from all sources | `ASSET` | 12h | |
| `xql_query` | XQL for specific investigation context | investigation context | on-demand | |
| — `respond` | Isolate, terminate, block | **RESPOND** | — | |

**5 collect + 1 respond.**

---

## 7 · Tenable (Nessus / tenable.io / tenable.sc)

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `assets` | Discovered asset inventory | `ASSET` (often the widest inventory) | 12h | ★ |
| `vulnerabilities` | Vulnerability findings per asset | `VULNERABILITY` properties | 12h | ★ |
| `scans` | Scan config, targets, schedules, coverage | **scan coverage gaps** | 24h | ★ |
| `compliance` | Compliance audit results | posture | 24h | |
| `plugins` | Plugin metadata for finding context | remediation text | 7d | |

**5 collectors.** The `scans` collector is load-bearing for an unobvious reason: it tells us **what the customer is not scanning**, which is a coverage finding in its own right.

---

## 8 · Qualys

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `assets` | Asset inventory, tags | `ASSET` | 12h | ★ |
| `vulnerabilities` | Detection results per host | `VULNERABILITY` | 12h | ★ |
| `scan_config` | Scan profiles, schedules, target scope | coverage gaps | 24h | ★ |
| `policy_compliance` | PC posture results | posture | 24h | |
| `certificates` | Certificate inventory and expiry | `CERTIFICATE` | 24h | |

**5 collectors.**

---

## 9 · Rapid7 InsightVM

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `assets` | Asset inventory | `ASSET` | 12h | ★ |
| `vulnerabilities` | Findings per asset | `VULNERABILITY` | 12h | ★ |
| `sites` | Scan sites and scope | coverage gaps | 24h | ★ |
| `policies` | Policy compliance results | posture | 24h | |

**4 collectors.**

---

## 10 · Wiz

```
  ⚠ Wiz is both a competitor and a useful source. Where a customer
    already runs it, ingesting its findings avoids duplicating
    scan effort. Its GRAPH is not ingested — only findings and
    inventory — because our closure must be ours.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `issues` | Wiz issues and control failures | `FINDING` | 1h | ★ |
| `inventory` | Cloud resource inventory | `ASSET` corroboration | 12h | |
| `vulnerabilities` | Vulnerability findings | `VULNERABILITY` | 12h | |
| `data_findings` | DSPM classification results | `DATA_CLASS` properties | 24h | ★ |

**4 collectors.**

---

## 11 · Orca Security

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `alerts` | Alerts and risk findings | `FINDING` | 1h | ★ |
| `inventory` | Cloud asset inventory | `ASSET` corroboration | 12h | |
| `vulnerabilities` | CVE findings | `VULNERABILITY` | 12h | |
| `data_classification` | Sensitive data findings | `DATA_CLASS` | 24h | ★ |

**4 collectors.**

---

## 12 · Snyk

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Monitored projects mapped to repos | `REPOSITORY` linkage | 12h | ★ |
| `issues` | Dependency, code, container, IaC issues | `VULNERABILITY` | 12h | ★ |
| `sbom` | Dependency inventory | `ARTIFACT`, `CONTAINS` | 24h | |
| `licences` | Licence findings | compliance properties | 24h | |
| `orgs_members` | Org membership and roles | `IDENTITY` | 24h | |

**5 collectors.**

---

## 13 · Checkmarx

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Projects mapped to repos | `REPOSITORY` linkage | 12h | ★ |
| `scan_results` | SAST findings by severity | `VULNERABILITY` | 12h | ★ |
| `presets_policies` | Scan presets and policy config | coverage context | 24h | |
| `sca_results` | Software composition findings | `VULNERABILITY`, `ARTIFACT` | 12h | |

**4 collectors.**

---

## 14 · Veracode

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `applications` | Application profiles | `APPLICATION` linkage | 12h | ★ |
| `findings` | Static, dynamic and SCA findings | `VULNERABILITY` | 12h | ★ |
| `sandboxes` | Sandbox scans | pre-production context | 24h | |
| `policy_compliance` | Policy pass/fail state | compliance properties | 24h | |

**4 collectors.**

---

## 15 · Semgrep

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `deployments` | Deployment and repo linkage | `REPOSITORY` linkage | 12h | ★ |
| `findings` | Rule findings by severity | `VULNERABILITY` | 12h | ★ |
| `secrets` | Secret detection results | `SECRET` → identity bridge | 4h | ★ |

**3 collectors.**

---

## 16 · Aqua Security

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `images` | Scanned images, registries, layers | `ARTIFACT`, `CONTAINS` | 12h | ★ |
| `vulnerabilities` | Image and host findings | `VULNERABILITY` | 12h | ★ |
| `runtime_policies` | Runtime protection policies | `PROTECTS` | 12h | |
| `enforcers` | Enforcer deployment and coverage | coverage gaps | 12h | |
| `secrets` | Secrets found in images | `SECRET` | 12h | ★ |

**5 collectors.**

---

## 17 · Prisma Cloud

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `alerts` | Cloud and workload alerts | `FINDING` | 1h | ★ |
| `assets` | Cloud asset inventory | `ASSET` corroboration | 12h | |
| `vulnerabilities` | Host, image and serverless findings | `VULNERABILITY` | 12h | ★ |
| `compliance` | Compliance standard results | compliance mapping | 24h | |
| `iam_findings` | CIEM findings and net effective permissions | closure **corroboration** | 12h | ★ |
| `data_findings` | DSPM classification | `DATA_CLASS` | 24h | |

**6 collectors.** The `iam_findings` collector is interesting: it is another vendor's closure output, useful as a cross-check against ours. Disagreement between their effective permissions and our closure is a strong diagnostic signal.

---

## 18 · Black Duck

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Projects and versions | `APPLICATION` linkage | 12h | ★ |
| `components` | Component inventory / BOM | `ARTIFACT`, `CONTAINS` | 24h | ★ |
| `vulnerabilities` | Component vulnerabilities | `VULNERABILITY` | 12h | |

**3 collectors.**

---

## Domain summary

| Connector | Collect | Respond |
|---|---|---|
| CrowdStrike Falcon | 10 | 1 |
| Defender for Endpoint | 8 | 1 |
| Defender for Cloud | 5 | — |
| SentinelOne | 6 | 1 |
| FortiEDR | 5 | 1 |
| Cortex XDR | 5 | 1 |
| Tenable | 5 | — |
| Qualys | 5 | — |
| Rapid7 InsightVM | 4 | — |
| Wiz | 4 | — |
| Orca | 4 | — |
| Snyk | 5 | — |
| Checkmarx | 4 | — |
| Veracode | 4 | — |
| Semgrep | 3 | — |
| Aqua | 5 | — |
| Prisma Cloud | 6 | — |
| Black Duck | 3 | — |
| **Total** | **91** | **5** |

### The three things this domain uniquely contributes

```
  1  PROTECTS EDGES
     An asset with a healthy EDR sensor, or covered by a runtime
     policy, is harder to traverse. PROTECTS does not create
     reachability — it REDUCES THE WEIGHT of adjacent CAN_EXECUTE
     edges (01 §6.4). This is the only domain that produces them
     at scale.

  2  COVERAGE GAPS AS FINDINGS
     scan_config, sites, enforcers, assets_unmanaged — these tell
     us what the customer is NOT covering. An unscanned subnet or
     an EDR-less host is a finding, and only these connectors know.

  3  VULNERABILITY PROPERTIES
     We never scan for vulnerabilities ourselves. We ingest them as
     properties on assets other connectors created, and they become
     one input to path scoring rather than a finding list.
```

### And the thing it cannot contribute

No connector in this domain produces a `CAN_ASSUME` edge, a permission, or a policy document. **The entire domain, fully deployed, cannot produce a single attack path on its own.** It makes existing paths more accurate and better prioritised. That is genuinely valuable — and it is not a substitute for domains 01 and 02.

---

*Next: [Network and edge](05-network-edge.md)*

# Domain 05 — Network and Edge

**15 connectors · 96 collectors** · [Index](00-index.md)

Band 4. This is the domain where the **configuration-versus-logs** distinction is starkest: a firewall's syslog stream is ~1.24 TB/day and yields ~40 edges; its rulebase is ~40 MB and yields ~14,000. Roughly **350× more value per byte from the config path.**

⚠ **Domain-wide rule:** the config collectors within a firewall connector are useless individually. A rule referencing `addrgrp_DMZ` on `svcgrp_WEB` means nothing without the object definitions. Policy, address objects and service objects must be enabled together — a manifest dependency, not a suggestion.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · Palo Alto NGFW / Panorama

```
  api_surface   hybrid — configuration (XML/REST API) + log_stream (syslog)
  functions     collect · respond (separate — dynamic address groups)
  auth          dedicated read-only admin, API key
  least priv    a custom admin role with read-only on config and logs
  coverage      config collectors emit; syslog never does
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `security_policies` | Security rulebase, in evaluation order | `ROUTES_TO` reachability | 12h | ★ |
| `address_objects` | Address objects and groups | resolves rule endpoints | 12h | ★ |
| `service_objects` | Service objects and groups | resolves rule ports | 12h | ★ |
| `zones_interfaces` | Zones, interfaces, VLANs, virtual routers | `NETWORK` nodes | 12h | ★ |
| `nat_policies` | NAT rulebase | reachability transforms | 12h | ★ |
| `routing` | Static and dynamic routes, virtual routers | `ROUTES_TO` | 12h | |
| `vpn` | IPsec and GlobalProtect config, tunnels | cross-boundary `ROUTES_TO` | 12h | ★ |
| `security_profiles` | AV, IPS, URL filtering profiles and groups | `PROTECTS` | 12h | |
| `admins` | Device admin accounts, roles, auth profiles | `IDENTITY` — forgotten privileged accounts | 24h | ★ |
| `device_groups` | Panorama device groups and templates | scope and inheritance | 12h | |
| `traffic_logs` | Syslog traffic logs | aggregated `CONNECTS_TO` | continuous | |
| `threat_logs` | Threat/UTM logs | `FINDING` overlay | continuous | |
| `config_logs` | Configuration change events | change attribution | continuous | ★ |

**13 collectors.**

---

## 2 · Fortinet FortiGate / FortiManager

```
  api_surface   hybrid
  auth          REST API token with a read-only profile
  ⚠             firmware upgrades change log formats. Parse-rate
                monitoring per device is mandatory here, not optional.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `firewall_policies` | Policy table in order | `ROUTES_TO` | 12h | ★ |
| `address_objects` | Addresses, groups, wildcards | resolves rule endpoints | 12h | ★ |
| `service_objects` | Services, groups, custom services | resolves rule ports | 12h | ★ |
| `interfaces_zones` | Interfaces, zones, VLANs, VDOMs | `NETWORK` | 12h | ★ |
| `routing` | Static and policy routes | `ROUTES_TO` | 12h | |
| `nat` | VIPs, IP pools, central NAT | reachability transforms | 12h | ★ |
| `vpn` | IPsec and SSL-VPN config, portals, tunnels | cross-boundary `ROUTES_TO` | 12h | ★ |
| `security_profiles` | AV, IPS, webfilter, application control | `PROTECTS` | 12h | |
| `admins` | Admin accounts, profiles, trusted hosts | `IDENTITY` | 24h | ★ |
| `traffic_logs` | Syslog traffic | aggregated `CONNECTS_TO` | continuous | |
| `utm_logs` | IPS/AV/webfilter events | `FINDING` overlay | continuous | |
| `event_logs` | System and config change events | change attribution | continuous | ★ |

**12 collectors.**

---

## 3 · Check Point

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `access_rulebase` | Access layers and rules | `ROUTES_TO` | 12h | ★ |
| `network_objects` | Hosts, networks, groups, dynamic objects | resolves rule endpoints | 12h | ★ |
| `services` | Services and service groups | resolves rule ports | 12h | ★ |
| `gateways` | Gateways, clusters, interfaces, topology | `NETWORK` | 12h | ★ |
| `nat_rulebase` | NAT rules | reachability transforms | 12h | ★ |
| `vpn_communities` | VPN communities and encryption domains | cross-boundary `ROUTES_TO` | 12h | ★ |
| `threat_prevention` | IPS/AV/anti-bot profiles and layers | `PROTECTS` | 12h | |
| `administrators` | Admins, permission profiles | `IDENTITY` | 24h | ★ |
| `logs` | Traffic and audit logs | `CONNECTS_TO`, change attribution | continuous | |

**9 collectors.**

---

## 4 · Cisco ASA / Firepower (FMC)

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `access_rules` | ACLs / access control policies | `ROUTES_TO` | 12h | ★ |
| `network_objects` | Object groups, networks | resolves rule endpoints | 12h | ★ |
| `service_objects` | Service and port object groups | resolves rule ports | 12h | ★ |
| `interfaces` | Interfaces, security levels, zones | `NETWORK` | 12h | ★ |
| `nat_rules` | NAT configuration | reachability transforms | 12h | ★ |
| `vpn` | Site-to-site and remote access VPN | cross-boundary `ROUTES_TO` | 12h | |
| `intrusion_policies` | IPS policies and variable sets | `PROTECTS` | 12h | |
| `users` | Device users, AAA config | `IDENTITY` | 24h | ★ |
| `syslog` | Traffic and system events | `CONNECTS_TO` | continuous | |

**9 collectors.**

---

## 5 · Juniper SRX

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `security_policies` | Zone-pair policies | `ROUTES_TO` | 12h | ★ |
| `address_books` | Address book entries and sets | resolves rule endpoints | 12h | ★ |
| `applications` | Application and application-set definitions | resolves rule ports | 12h | ★ |
| `zones_interfaces` | Security zones and interfaces | `NETWORK` | 12h | ★ |
| `nat` | Source, destination and static NAT | reachability transforms | 12h | |
| `routing` | Routing instances and static routes | `ROUTES_TO` | 12h | |
| `admins` | Login classes and users | `IDENTITY` | 24h | |

**7 collectors.**

---

## 6 · F5 BIG-IP

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `virtual_servers` | Virtual servers, VIPs, ports | `LOAD_BALANCER`, `EXPOSES` | 12h | ★ |
| `pools` | Pools and pool members | backend mapping → `ROUTES_TO` | 12h | ★ |
| `irules_policies` | iRules and local traffic policies | routing logic, exposure edge cases | 12h | |
| `ssl_profiles` | Client/server SSL profiles, certificates | `CERTIFICATE`, TLS posture | 12h | |
| `apm_policies` | Access Policy Manager policies | `PROTECTS`, `AUTHENTICATES_TO` | 12h | ★ |
| `asm_policies` | WAF policies and enforcement mode | `PROTECTS` | 12h | |
| `admins` | Administrative users and roles | `IDENTITY` | 24h | |

**7 collectors.**

---

## 7 · Citrix ADC / NetScaler

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `vservers` | Virtual servers and bindings | `EXPOSES` | 12h | ★ |
| `services_groups` | Services and service groups | backend mapping | 12h | ★ |
| `policies` | Responder, rewrite, authentication policies | routing and auth logic | 12h | |
| `gateway` | Gateway virtual servers, session policies | remote access `ROUTES_TO` | 12h | ★ |
| `certificates` | SSL certificates and bindings | `CERTIFICATE` | 24h | |
| `admins` | System users and command policies | `IDENTITY` | 24h | |

**6 collectors.**

---

## 8 · VMware NSX

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `segments` | Logical segments and transport zones | `NETWORK` | 12h | ★ |
| `dfw_rules` | Distributed firewall rules and sections | **east-west** `ROUTES_TO` | 12h | ★ |
| `groups` | NSX groups and dynamic membership criteria | rule endpoint resolution | 12h | ★ |
| `services` | Service definitions | rule port resolution | 12h | ★ |
| `gateways` | Tier-0/Tier-1 gateways, routing | `ROUTES_TO` | 12h | |
| `gateway_firewall` | Gateway firewall rules | north-south `ROUTES_TO` | 12h | ★ |
| `vpn` | IPsec and L2VPN | cross-boundary reachability | 12h | |
| `users_roles` | NSX users and role bindings | `IDENTITY`, `CAN_MODIFY` | 24h | |

**8 collectors.** NSX distributed firewall is the only practical source of **east-west** segmentation inside a virtualised data centre — the traffic perimeter firewalls never see.

---

## 9 · Zscaler (ZIA / ZPA)

```
  ⚠ this connector matters disproportionately for SHADOW AI.
    It sees which AI services users reach and how much they upload,
    without any TLS interception on our part (../01 §13.1).
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `url_policies` | URL filtering rules and categories | `PROTECTS`, blocked/allowed AI services | 12h | ★ |
| `cloud_apps` | Cloud application visibility and control | `AI_APPLICATION`, shadow AI discovery | 4h | ★ |
| `app_segments` | ZPA application segments and policies | `ROUTES_TO`, zero-trust access | 12h | ★ |
| `access_policies` | ZPA access policy and criteria | `CAN_ACCESS`, `PROTECTS` | 12h | ★ |
| `dlp_policies` | DLP rules and dictionaries | `PROTECTS`, data-egress findings | 12h | |
| `web_logs` | Web transaction logs | `USES` on AI services, volume | continuous | ★ |
| `dlp_incidents` | DLP incidents | `FINDING`, data exposure | 15m | |
| `admins` | Admin accounts and roles | `IDENTITY` | 24h | |

**8 collectors.**

---

## 10 · Netskope

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `cloud_apps` | Discovered SaaS/AI apps, CCI risk score | `AI_APPLICATION`, shadow AI | 4h | ★ |
| `policies` | Real-time protection policies | `PROTECTS` | 12h | ★ |
| `dlp_profiles` | DLP rules and classifiers | `PROTECTS` | 12h | |
| `app_instances` | Sanctioned vs unsanctioned instances | approval state | 12h | ★ |
| `alerts` | DLP, malware, anomaly alerts | `FINDING` | 15m | |
| `events` | Application and page events | `USES`, volume | continuous | ★ |
| `private_apps` | NPA private application access | `ROUTES_TO` | 12h | |

**7 collectors.**

---

## 11 · Cisco Umbrella

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `policies` | DNS policies, destination lists | `PROTECTS` | 12h | ★ |
| `destinations` | Allowed/blocked destination lists | AI service approval state | 12h | ★ |
| `dns_logs` | DNS query activity | `USES` on AI services, shadow AI | continuous | ★ |
| `identities` | Umbrella identities and mappings | identity↔network correlation | 12h | |

**4 collectors.** Even without a proxy, DNS activity is enough to establish *user → AI service* relationships (`../01 §25.1`).

---

## 12 · Infoblox / core DNS-DHCP

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `dns_records` | Forward and reverse zones, records | `DNS_NAME`, name↔IP resolution | 12h | ★ |
| `dhcp_leases` | Active and historical leases | **time-bounded IP↔asset binding** | 1h | ★ |
| `ipam` | Networks, ranges, utilisation | `NETWORK` | 12h | ★ |
| `dns_queries` | Query logs where available | `USES`, shadow AI | continuous | |
| `admins` | Admin accounts and permissions | `IDENTITY` | 24h | |

**5 collectors.** The `dhcp_leases` collector is quietly essential: without time-bounded IP→asset bindings, enrichment misattributes one host's activity to another after every lease change (`../engines/04-normalizer-and-enrichment.md §4`).

---

## 13 · NetFlow / IPFIX / sFlow

```
  api_surface   log_stream only
  auth          none — flow exporters push to a listener
  ⚠             aggregate in memory at receive. NEVER journal
                individual records: 4.1 billion/day is not storable
                and not useful individually.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `netflow_v9` | NetFlow v9 with templates | aggregated `CONNECTS_TO` | continuous | ★ |
| `ipfix` | IPFIX with templates | aggregated `CONNECTS_TO` | continuous | ★ |
| `sflow` | sFlow samples | aggregated `CONNECTS_TO` (sampled) | continuous | |
| `exporter_registry` | Which devices export, and their templates | coverage, source identification | 12h | |

**4 collectors.** Flow uniquely reveals **east-west traffic inside a segment**, which perimeter firewalls never observe and where lateral movement lives.

---

## 14 · Cloudflare

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `zones_dns` | Zones and DNS records | `DNS_NAME`, external exposure | 12h | ★ |
| `waf_rules` | WAF rulesets and custom rules | `PROTECTS` | 12h | |
| `access_policies` | Cloudflare Access applications and policies | `CAN_ACCESS`, `PROTECTS` | 12h | ★ |
| `tunnels` | Cloudflare Tunnel config and routes | **cross-boundary** `ROUTES_TO` | 12h | ★ |
| `members` | Account members and roles | `IDENTITY` | 24h | |
| `logs` | HTTP and firewall event logs | `CONNECTS_TO`, `FINDING` | continuous | |

**6 collectors.** Tunnels deserve attention — they create inbound reachability into a private network that no firewall rulebase records.

---

## 15 · Akamai

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `properties` | Property configurations, hostnames, origins | `EXPOSES`, origin mapping | 12h | ★ |
| `waf_configs` | Security configurations and policies | `PROTECTS` | 12h | |
| `network_lists` | IP and geo network lists | `PROTECTS` scope | 12h | |
| `certificates` | Certificate provisioning and expiry | `CERTIFICATE` | 24h | |
| `users_roles` | Users, roles, group access | `IDENTITY` | 24h | |

**5 collectors.**

---

## Domain summary

| Connector | Config collectors | Stream collectors | Total |
|---|---|---|---|
| Palo Alto NGFW | 10 | 3 | 13 |
| FortiGate | 9 | 3 | 12 |
| Check Point | 8 | 1 | 9 |
| Cisco ASA / Firepower | 8 | 1 | 9 |
| Juniper SRX | 7 | 0 | 7 |
| F5 BIG-IP | 7 | 0 | 7 |
| Citrix ADC | 6 | 0 | 6 |
| VMware NSX | 8 | 0 | 8 |
| Zscaler | 6 | 2 | 8 |
| Netskope | 5 | 2 | 7 |
| Cisco Umbrella | 3 | 1 | 4 |
| Infoblox | 4 | 1 | 5 |
| NetFlow / IPFIX | 1 | 3 | 4 |
| Cloudflare | 5 | 1 | 6 |
| Akamai | 5 | 0 | 5 |
| **Total** | **92** | **18** | **110** |

### Configured versus observed reachability

Both matter, and their **disagreement** is a finding in itself:

```
  ROUTES_TO      configured reachability, from the rulebase
                 → what is POSSIBLE

  CONNECTS_TO    observed reachability, from flow and traffic logs
                 → what HAPPENED

  configured but never observed   → an unused rule. Attack surface
                                    with no business justification.
  observed but not configured     → a shadowed rule, a bypass, or a
                                    path through a device we do not
                                    collect from.
```

### The secondary contribution nobody expects

Every connector in this domain has an `admins` collector, and they are all starred. Network device administrative accounts are:

- rarely in the IdP
- rarely rotated
- frequently shared
- almost never reviewed
- and hold `CAN_MODIFY` over the reachability of the entire estate

They are among the highest-privilege, least-governed identities in a typical enterprise, and this domain is the only place they appear.

---

*Next: [Data platforms](06-data-platforms.md)*

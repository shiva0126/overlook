# Collector Specs — The Deployment Set

**Series:** [Collector documentation](00-anatomy.md)

Full specifications for the collectors in the picked infrastructure: FortiGate, CrowdStrike, FortiEDR, Scalefusion and the Agent.

`crowdstrike.hosts` is specified as the minimal worked example in [01 §6](01-specification-format.md#6-worked-minimal-example) and is not repeated here.

These five connectors between them exercise **all four ingress classes** — PULL, STREAM, PUSH and AGENT — which is why they are a good first set for the pipeline even though they need one cloud IAM connector alongside them to feed the derivation engines.

---

## 1 · `fortigate.firewall_policies`

A configuration collector with hard dependencies on two others. Individually useless, collectively the source of all network reachability.

```yaml
collector:
  id: fortigate.firewall_policies
  connector: fortigate
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides CONFIGURED network reachability — what is possible
    rather than what happened. 40 MB of rulebase yields ~14,000
    ROUTES_TO edges; the device's 1.24 TB/day of traffic syslog
    yields ~40. This is the highest-value-per-byte collector in
    the network domain.

  source:
    api_surface: configuration
    calls:
      - GET /api/v2/cmdb/firewall/policy
      - GET /api/v2/cmdb/system/vdom
    object_type: Firewall policy table, in evaluation order
    scope_unit: device
    scope_dimensions: [device, vdom]

  auth:
    requires_scope:
      - read access to firewall.policy
      - read access to system.vdom
    optional_scope: []
    degrades_gracefully: false
    failure_remediation: >
      Create a read-only REST API admin with an access profile
      granting Read on Firewall and System. Restrict by trusted
      host to the Edge Node address.

  fetch:
    pagination: none          # FortiOS returns the full table
    page_size: null
    max_pages: 1
    stream_downstream: true
    estimated_calls: "1 per VDOM"
    rate_limit_domain: [fortigate, device, cmdb]

  cursor: { supported: false, field: null,
            advance_when: after_durable_write }

  coverage:
    emits_window: true
    scope: [device, vdom]
    completeness_signal: >
      HTTP 200 with a complete policy array and a matching
      revision identifier

  schedule: { interval: 12h, full_enumeration_interval: 12h,
              jitter_pct: 15, blackout_aware: true }

  produces:
    relationships:
      - predicate: ROUTES_TO
        from: NETWORK
        to: NETWORK
        confidence: 0.95
        significant_attributes: [port, protocol, action, conditions]
        precondition: >
          EVERY srcaddr, dstaddr and service reference must resolve
          against fortigate.address_objects and
          fortigate.service_objects. An unresolved reference
          produces NO edge — a rule referencing addrgrp_DMZ means
          nothing without the group's members.

  mapping:
    policyid:      properties.rule_id     → ordering is significant
    srcintf / dstintf:
                   → zone/interface, resolved via
                     fortigate.interfaces_zones
    srcaddr[] / dstaddr[]:
                   → resolved via fortigate.address_objects
                     → NETWORK canonical keys
    service[]:     → resolved via fortigate.service_objects
                     → port + protocol
    action:        accept → edge emitted
                   deny   → edge NOT emitted, but recorded as a
                            constraint for shadowing analysis
    status:        disable → no edge
    schedule:      → attributes.conditions where time-bounded
    nat:           → flagged; reachability is transformed
    utm-status:    → PROTECTS edge to the profile group
    comments:      properties.description

  health:
    success_criteria: "policies_enumerated >= 1 AND unresolved_refs == 0"
    baseline_metric: policies_enumerated
    silent_threshold_pct: 5
    staleness_threshold: 24h
    field_presence_monitored: [srcaddr, dstaddr, service]

  depends_on:
    hard: [fortigate.address_objects, fortigate.service_objects,
           fortigate.interfaces_zones]
    soft: [fortigate.nat, fortigate.routing]
    consumed_by: [E7, E12]

  failure_modes:
    - condition: a policy references an address object that does not
                 resolve
      behaviour: >
        NO edge for that policy. Increment unresolved_refs, which
        fails the success criteria and therefore withholds the
        coverage window. A partially-resolved rulebase would
        tombstone reachability that still exists.
    - condition: rule order changes but rule content does not
      behaviour: >
        Order affects evaluation. Rule ordering is part of the
        significant attribute set via the derived effective action.
    - condition: VDOMs enabled but the account can read only one
      behaviour: >
        Coverage window for that VDOM only. Other VDOMs are
        uncollected, not empty.

  fixtures:
    required:
      - a 400-rule policy table with nested address groups
      - a rule referencing an unresolvable object
      - a deny rule shadowing a later accept rule
      - a disabled rule
      - a rule with a schedule
      - a multi-VDOM device
      - a rule with NAT enabled
```

**The lesson:** `unresolved_refs == 0` in the success criteria. This is how a hard dependency is enforced at runtime rather than only declared in a manifest — the collector refuses to claim completeness when its companions have not delivered.

---

## 2 · `fortigate.traffic_logs`

The counterexample: enormous volume, minimal graph contribution, and **structurally incapable of driving retraction**.

```yaml
collector:
  id: fortigate.traffic_logs
  connector: fortigate
  version: 1.0.0
  load_bearing: false

  purpose: >
    Provides OBSERVED reachability, to be compared against the
    configured reachability from firewall_policies. The disagreement
    is the value: configured-but-never-observed is an unused rule;
    observed-but-not-configured is a shadowed rule or a bypass.

  source:
    api_surface: log_stream
    calls: []                 # push, not pull
    listener: syslog/tcp-tls:6514
    object_type: FortiOS traffic log
    scope_unit: device
    scope_dimensions: [device]

  auth:
    requires_scope: []
    transport_auth: source IP allow-list + optional TLS client cert

  fetch:
    pagination: none
    stream_downstream: true
    aggregate_at_receive: true
    aggregation:
      key: [src_subnet, dst_subnet, dst_port, protocol, action]
      window: 15m
      expected_reduction: "~26,000:1"
    rate_limit_domain: []     # inbound; governed by shedding not quota

  cursor: { supported: false, field: null, advance_when: n/a }

  coverage:
    emits_window: false
    reason: >
      A STREAM CAN NEVER PROVE COMPLETENESS. "I did not see it" is
      indistinguishable from "it was not sent." This collector can
      only ADD to the graph. It can never retract.

  schedule: { interval: continuous }

  produces:
    entities:
      - type: FLOW_AGGREGATE
        canonical_key_source: [aggregation key hash]
    relationships:
      - predicate: CONNECTS_TO
        from: NETWORK
        to: NETWORK | ASSET
        confidence: 0.60
        significant_attributes: [port, protocol]

  mapping:
    srcip / dstip:   → subnet, then NETWORK canonical key
    dstport:         attributes.port
    proto:           attributes.protocol
    action:          → accept only; denied traffic is a FINDING input,
                       not a CONNECTS_TO edge
    srcintf/dstintf: → zone context
    sentbyte/rcvdbyte: → volume, BUCKETED
    user:            → identity attribution where FSSO is in use
    date/time:       → aggregation window assignment

  health:
    success_criteria: "records_received > 0 AND parse_rate >= 0.95"
    baseline_metric: records_per_hour
    silent_threshold_pct: 20    # traffic legitimately varies
    staleness_threshold: 1h
    field_presence_monitored: [srcip, dstip, dstport, user]

  depends_on:
    hard: []
    soft: [fortigate.interfaces_zones, infoblox.dhcp_leases]
    consumed_by: [E1, E12]

  failure_modes:
    - condition: firmware upgrade changes the log format
      behaviour: >
        Parse rate collapses. Records QUARANTINED, never dropped.
        Samples retained. Source re-fingerprinted. Reachability
        edges from this device go STALE — and because no coverage
        window was ever emitted, nothing is tombstoned.
    - condition: UDP transport in use
      behaviour: >
        Estimate loss from sequence gaps and DISPLAY it. Never
        present the feed as complete.
    - condition: aggregation key cardinality spikes
      behaviour: >
        Spill to disk. If spill fails, shed the lowest-priority
        key ranges and record what was shed.

  fixtures:
    required:
      - 100k records across 4 aggregation windows
      - a FortiOS 7.4 format sample and a 7.6 format sample
      - records with FSSO user attribution and without
      - a malformed record mid-stream
      - a cardinality spike
```

**The lesson:** `emits_window: false` with an explicit `reason` field. Every stream collector must state this, because the temptation to let a high-volume source drive retraction is exactly how a graph deletes half of itself after a quiet weekend.

---

## 3 · `scalefusion.devices`

The collector that justifies including Scalefusion at all.

```yaml
collector:
  id: scalefusion.devices
  connector: scalefusion
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides the AUTHORITATIVE device-to-user binding from
    enrollment. Every other source infers this from logon events —
    weak, time-bounded and wrong on shared machines. Enrollment is
    a fact, and it is what lets an agent observation on a laptop
    resolve to a person.

  source:
    api_surface: configuration
    calls:
      - GET /api/v1/devices
      - GET /api/v1/devices/{id}
    object_type: Enrolled device
    scope_unit: account
    scope_dimensions: [account]

  auth:
    requires_scope: [device read]
    optional_scope: [application inventory read]
    degrades_gracefully: true
    failure_remediation: >
      Generate an API key with read-only device scope in
      Scalefusion → Admin → API.

  fetch:
    pagination: offset
    page_size: 100
    max_pages: 10000
    stream_downstream: true
    estimated_calls: "1 per 100 devices"
    rate_limit_domain: [scalefusion, account, devices]

  cursor:
    supported: true
    field: last_updated
    advance_when: after_durable_write

  coverage:
    emits_window: true
    scope: [account]
    completeness_signal: "a page returns fewer records than page_size"

  schedule: { interval: 4h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    entities:
      - type: ASSET
        subtype: endpoint
        canonical_key_source: [hwuuid, serial, fqdn, mac]
        properties: [os, os_version, model, enrollment_date,
                     compliance_state, last_seen, ownership]
    relationships:
      - predicate: AUTHENTICATES_TO
        from: IDENTITY
        to: ASSET
        confidence: 0.98
        significant_attributes: [mechanism]
        attributes:
          mechanism: enrollment      # ⚠ NOT observed_logon
        precondition: >
          the device must have an assigned user that resolves to a
          known IDENTITY. An unassigned device produces an ASSET
          and no edge.

  mapping:
    device_id:        properties.vendor_id
    serial_number:    canonical_key via hwuuid or serial
    imei / udid:      canonical_key candidates for mobile
    device_name:      canonical_key via fqdn where domain-joined
    mac_address:      canonical_key via mac
    assigned_user.email:
                      → subject canonical key via email:
                        ⚠ LOWERCASED, per contracts Part IV
    os / os_version:  properties
    compliance_status: properties.compliance_state
    enrollment_date:  first_seen candidate
    ownership:        corporate | byod → properties.ownership

  health:
    success_criteria: "devices_enumerated >= 1 AND auth_errors == 0"
    baseline_metric: devices_enumerated
    silent_threshold_pct: 5
    staleness_threshold: 12h
    field_presence_monitored: [assigned_user.email, serial_number]

  depends_on:
    hard: []
    soft: [scalefusion.users, entra.users, ad.users]
    consumed_by: [E6, E9, E12]

  failure_modes:
    - condition: device has no assigned user
      behaviour: >
        Emit ASSET, no edge. A shared or kiosk device is legitimate.
        Raise a low-severity finding only if it is a corporate
        endpoint with recent activity.
    - condition: assigned_user.email does not resolve to a known
                 identity
      behaviour: >
        Emit ASSET. Hold the binding as a pending alias for E6.
        Do NOT create an identity from an MDM record — MDM is
        usually downstream of the real directory.
    - condition: MDM covers only managed devices
      behaviour: >
        This is the normal case and must be reported as coverage,
        not completeness. Identity coverage from MDM is not
        workforce coverage.

  fixtures:
    required:
      - a corporate laptop with an assigned user
      - a BYOD device
      - an unassigned shared device
      - a device whose assigned user is unknown to the directory
      - a device with a serial but no FQDN
      - a non-compliant device
```

**The lesson:** `mechanism: enrollment` on the `AUTHENTICATES_TO` edge. This is the reconciliation from `../13-contracts.md §XI.2` in practice — rather than a separate `ASSIGNED_TO` predicate, the mechanism attribute distinguishes an authoritative enrollment binding from a weak observed logon, and because `mechanism` is significant, the two never merge into one edge.

---

## 4 · `agent.mcp_configs`

The differentiated collector. Nothing else in the catalog can produce this.

```yaml
collector:
  id: agent.mcp_configs
  connector: overlook_agent
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides MCP servers configured by hand on endpoints — the layer
    that appears in no registry, no admin API and no cloud console.
    Every AI-security competitor starts from a registry; this is the
    unregistered remainder, and it holds real credentials.

  source:
    api_surface: agent
    calls: []                 # local filesystem read
    paths:
      - ~/.claude/claude_desktop_config.json
      - ~/.cursor/mcp.json
      - ~/Library/Application Support/Claude/*
      - %APPDATA%/Claude/*
      - VS Code / Windsurf / Zed settings with MCP blocks
      - project-local .mcp/ directories in known workspace roots
    object_type: MCP client configuration
    scope_unit: host
    scope_dimensions: [host, user]

  auth:
    requires_scope: [local filesystem read as the logged-on user]
    optional_scope: []
    degrades_gracefully: true
    failure_remediation: >
      macOS: deploy the PPPC profile via MDM. Without it, reads of
      ~/Library trigger per-user TCC consent prompts and coverage
      will be partial and inconsistent.

  fetch:
    pagination: none
    max_pages: 1
    stream_downstream: true
    estimated_calls: "n/a — local read, ~2 KB per host"
    rate_limit_domain: []
    resource_budget:
      cpu_pct_max: 1
      io_mbps_max: 5

  cursor:
    supported: true
    field: config file mtime
    advance_when: after_durable_write

  coverage:
    emits_window: true
    scope: [host]
    completeness_signal: >
      every known config path was checked and returned either a
      parsed document or a definitive not-found.
      ⚠ A TCC-denied or permission-denied path means NO window
        for that host — partial coverage would tombstone servers
        that still exist.

  schedule: { interval: 4h, jitter_pct: 25, blackout_aware: false }

  produces:
    entities:
      - type: MCP_SERVER
        canonical_key_source: [mcp]
        properties: [server_name, transport, command, args,
                     root_paths, package_source, config_path]
      - type: MCP_TOOL
        canonical_key_source: [mcptool]
        properties: [tool_name]
      - type: SECRET
        canonical_key_source: [secretfp]
        properties: [credential_type, location_kind]
        ⚠ PRESENCE ONLY. The value is never read, never hashed,
          never transmitted. credential_type is inferred from the
          ENV VAR NAME, not from any value.
    relationships:
      - predicate: RUNS_ON
        from: MCP_SERVER
        to: ASSET
        confidence: 0.99
        significant_attributes: []
      - predicate: CAN_READ
        from: MCP_SERVER
        to: DATASTORE
        confidence: 0.90
        significant_attributes: [mechanism, conditions,
                                 resource_pattern]
        precondition: >
          only for filesystem-type servers with a resolvable root
          path. The root path becomes a DATASTORE canonical key.
      - predicate: CONTAINS
        from: MCP_SERVER
        to: SECRET
        confidence: 0.95
        significant_attributes: [containment_type]

  mapping:
    mcpServers.<name>:
      key:               → canonical_key via mcp:<name>@<asset key>
      command:           properties.command
      args[]:            properties.args
                         → for filesystem servers, args containing a
                           path become root_paths
      env:               KEYS ONLY → SECRET entities
                         ⚠ VALUES ARE NEVER READ. The agent parses
                           the key names and skips the values without
                           loading them into memory.
        GITHUB_TOKEN / GH_TOKEN     → credential_type github_token
        AWS_ACCESS_KEY_ID           → credential_type aws_key
        DATABASE_URL / *_DSN        → credential_type db_connection
        OPENAI_API_KEY / ANTHROPIC_API_KEY → credential_type ai_key
        <unrecognised>              → credential_type unknown
      transport:         stdio | sse | http → properties.transport
    tools[] (where the config declares them):
                         → MCP_TOOL entities

  health:
    success_criteria: "paths_checked == paths_known"
    baseline_metric: hosts_reporting
    silent_threshold_pct: 15    # laptops are legitimately offline
    staleness_threshold: 24h
    field_presence_monitored: [mcpServers]

  depends_on:
    hard: []
    soft: [scalefusion.devices, crowdstrike.hosts]
      # both provide the ASSET this attaches to, and the
      # device→user binding that makes it resolvable to a person
    consumed_by: [E6, E9, E12]

  failure_modes:
    - condition: config file exists but is malformed JSON
      behaviour: >
        Quarantine with a REDACTED sample — structure only, values
        stripped. Continue with other paths. No coverage window.
    - condition: TCC or permission denied on macOS
      behaviour: >
        NO coverage window for that host. Surface as a deployment
        gap with the PPPC remediation, not as a collection error.
    - condition: filesystem server root path does not exist
      behaviour: >
        Emit the MCP_SERVER, no CAN_READ edge. A stale config is
        not an access path.
    - condition: host offline for an extended period
      behaviour: >
        Host marked STALE with last_seen. Its MCP servers are NOT
        tombstoned. A closed laptop is not a removed configuration.

  fixtures:
    required:
      - claude_desktop_config.json with 3 servers, one filesystem
      - a config with GITHUB_TOKEN in env
      - a config with an unrecognised env var name
      - a filesystem server rooted at a non-existent path
      - malformed JSON
      - a host with no MCP configuration at all
      - a macOS TCC-denied read
      - the same server name on 14 hosts (fan-out case)
```

**The lesson:** the env-var handling. `credential_type` is inferred from the **key name only**; the value is skipped without being loaded. Detecting presence is sufficient to build the identity edge, and reading the value would make us the risk we are reporting.

---

## 5 · What the deployment set proves

```
  INGRESS CLASSES EXERCISED
    PULL     scalefusion.devices · crowdstrike.hosts ·
             fortigate.firewall_policies · fortiedr.devices
    STREAM   fortigate.traffic_logs · fortigate.utm_logs
    PUSH     (none in this set — add a GitHub webhook to cover it)
    AGENT    agent.mcp_configs and its eight siblings

  COVERAGE SEMANTICS EXERCISED
    emits window, bounded scope      firewall_policies, devices
    emits window, per-host           agent collectors
    NEVER emits a window             traffic_logs
    withholds on partial             firewall_policies with
                                     unresolved_refs

  RESOLUTION PATHS EXERCISED
    authoritative binding            scalefusion enrollment
    weak inferred binding            crowdstrike logon
    resolved via directory alias     agent local username
    unresolvable, held pending       MDM user unknown to the IdP

  AND WHAT IT STILL CANNOT DO
    no permission closure input      → E7 idle
    no escalation matching           → E8 idle
    no attack paths                  → the product's output absent

    Adding ONE cloud IAM connector alongside changes that, which is
    why the recommendation was five connectors plus one, not five.
```

---

*End of the collector series. Back to the [anatomy](00-anatomy.md).*

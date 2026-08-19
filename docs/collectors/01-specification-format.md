# Collectors — Specification Format

**Series:** [Collector documentation](00-anatomy.md)

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Why a format

826 collectors cannot be specified consistently by prose. They need a **template** so that any collector — written by anyone, at any time — carries the same information in the same shape, and so the conformance suite can be generated rather than hand-written.

A specification is also the handover artifact: it is what a connector author reads before writing code, and what a reviewer checks the code against.

---

## 2. The template

```yaml
collector:
  id: aws.iam.roles
  connector: aws
  version: 1.0.0
  load_bearing: true

  # ── WHAT QUESTION DOES THIS ANSWER ────────────────────────────
  purpose: >
    One sentence. What the graph gains that it would not have
    without this collector.

  # ── SOURCE ────────────────────────────────────────────────────
  source:
    api_surface: configuration        # configuration | log_stream
    calls:
      - ListRoles                     # exact API operations
      - GetRole
      - ListRolePolicies
      - GetRolePolicy
      - ListAttachedRolePolicies
    object_type: IAM Role
    scope_unit: account               # what one instance covers
    scope_dimensions: [account]       # + region where applicable

  # ── AUTHORISATION ─────────────────────────────────────────────
  auth:
    requires_scope:
      - iam:ListRoles
      - iam:GetRole
      - iam:GetRolePolicy
      - iam:ListAttachedRolePolicies
    optional_scope: []
    degrades_gracefully: false
    failure_remediation: >
      Exact text shown to the operator, including the policy
      statement to add.

  # ── FETCH BEHAVIOUR ───────────────────────────────────────────
  fetch:
    pagination: marker                # cursor|offset|link|time|marker
    page_size: 100
    max_pages: 10000                  # hard ceiling — infinite-loop guard
    stream_downstream: true           # never buffer the enumeration
    estimated_calls: "3 per role + 1 per page"
    rate_limit_domain: [aws, account, iam, list]

  # ── DELTA AND COVERAGE ────────────────────────────────────────
  cursor:
    supported: false                  # IAM has no change feed
    field: null
    advance_when: after_durable_write

  coverage:
    emits_window: true
    scope: [account]
    completeness_signal: "IsTruncated == false on the final page"

  schedule:
    interval: 4h
    full_enumeration_interval: 24h
    jitter_pct: 10
    blackout_aware: true

  # ── OUTPUT ────────────────────────────────────────────────────
  produces:
    entities:
      - type: ROLE
        subtype: assumable_role
        canonical_key_source: [arn]
        properties: [path, max_session_duration, created_at,
                     last_used, description, tags]
    relationships:
      - predicate: CAN_ASSUME
        from: IDENTITY | ROLE | PIPELINE
        to: ROLE
        confidence: 0.97
        significant_attributes: [mechanism, conditions,
                                 privilege_level, granted_via]
        precondition: >
          The trust policy must name the principal, or a service
          this principal can invoke. No edge without it.
    properties: []
    findings: []

  # ── MAPPING ───────────────────────────────────────────────────
  mapping:
    entity:
      Arn:                 canonical_key      # via arn: scheme
      RoleName:            properties.name
      Path:                properties.path
      CreateDate:          first_seen         # normalised to UTC
      MaxSessionDuration:  properties.max_session_duration
      Tags:                properties.tags
      RoleLastUsed.LastUsedDate: properties.last_used
    relationships:
      AssumeRolePolicyDocument.Statement[]:
        each_statement:
          Principal.AWS | Principal.Federated | Principal.Service:
            → subject canonical key
          Effect == "Allow":
            → emit edge, else skip
          Condition:
            → attributes.conditions, classified for satisfiability
          mechanism:
            derived: sts_assume_role | oidc_federation |
                     saml_federation | service_trust

  # ── HEALTH ────────────────────────────────────────────────────
  health:
    success_criteria: "roles_enumerated >= 1 AND auth_errors == 0"
    baseline_metric: roles_enumerated
    silent_threshold_pct: 5
    staleness_threshold: 18h
    field_presence_monitored: [AssumeRolePolicyDocument]

  # ── DEPENDENCIES ──────────────────────────────────────────────
  depends_on:
    hard: []
    soft: [aws.organizations]         # improves scope naming
    consumed_by: [E7, E8]

  # ── FAILURE MODES ─────────────────────────────────────────────
  failure_modes:
    - condition: 403 on GetRolePolicy for a subset of roles
      behaviour: >
        Emit those roles as entities without their inline policies.
        Flag reduced confidence. NO coverage window.
    - condition: trust policy references a deleted principal
      behaviour: >
        Condition classified UNSATISFIABLE. No edge created.
        Not an error.

  # ── FIXTURES ──────────────────────────────────────────────────
  fixtures:
    required:
      - 3-page enumeration, 250 roles
      - a role with an inline policy and an attached managed policy
      - a role whose trust policy names a service principal
      - a role whose trust policy has an OIDC condition
      - a role whose trust policy has a wildcard principal
      - a 429 mid-enumeration
      - a 403 on one role's policy
      - a zero-role account
```

---

## 3. Section rules

```
  purpose            ONE sentence, and it must name what the graph
                     gains. "Collects IAM roles" is not a purpose.
                     "Provides the ROLE nodes and trust preconditions
                     without which no CAN_ASSUME edge can exist" is.

  source.calls       EXACT operation names. This is what the
                     least-privilege policy is generated from, and
                     what a customer's security team reviews.

  auth.requires_scope
                     Must be minimal and verified. An over-broad
                     declared scope is a procurement objection.

  fetch.max_pages    MANDATORY. A provider returning the same cursor
                     forever is a real failure mode, not a
                     hypothetical.

  coverage.completeness_signal
                     Must name the EXACT field or condition that
                     proves the enumeration ended. If none exists,
                     emits_window MUST be false.

  produces.*.precondition
                     Where an edge requires state beyond the object
                     itself. Skipping preconditions is the primary
                     source of confident false positives.

  mapping            The bulk of the spec. Field-by-field, source to
                     Overlook. Declarative where the framework
                     supports it.

  failure_modes      Collector-specific only. Generic failures
                     (network, 429, 401) are handled by the framework
                     and are documented once, in the anatomy.

  fixtures.required  Every failure mode listed above must have a
                     corresponding fixture. This is how the
                     conformance suite is generated.
```

---

## 4. What the format deliberately excludes

```
  ✕ retry logic          framework
  ✕ backoff parameters   framework
  ✕ rate-limit mechanics framework, driven by rate_limit_domain
  ✕ credential handling  broker
  ✕ journaling           receive
  ✕ severity             E9 and the risk model
  ✕ normalization rules  E4, and ../13-contracts.md Part IV
```

If a specification is describing any of the above, the framework has a gap and that is the thing to fix.

---

## 5. Specification completeness gate

A collector spec is complete when:

```
  [ ] purpose names a graph consequence, not an activity
  [ ] every API operation is listed by exact name
  [ ] requires_scope is minimal and has been verified against a real
      least-privilege role
  [ ] pagination scheme and max_pages are declared
  [ ] completeness_signal is named, or emits_window is false
  [ ] every produced entity declares its canonical_key_source
  [ ] every produced relationship declares its significant attributes
      and, where applicable, its precondition
  [ ] the mapping covers every field consumed
  [ ] health has a baseline metric and a silent threshold
  [ ] every listed failure mode has a fixture
  [ ] consumed_by names which engines depend on it
```

---

## 6. Worked minimal example

For contrast — a small overlay collector, to show the format is not only for large ones.

```yaml
collector:
  id: crowdstrike.hosts
  connector: crowdstrike
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides ASSET nodes for every endpoint with a Falcon sensor, and
    the PROTECTS edges that reduce traversal weight on CAN_EXECUTE
    into them.

  source:
    api_surface: configuration
    calls: [QueryDevicesByFilterScroll, GetDeviceDetailsV2]
    object_type: Host
    scope_unit: tenant
    scope_dimensions: [cid]

  auth:
    requires_scope: ["hosts:read"]
    optional_scope: ["zero-trust-assessment:read"]
    degrades_gracefully: true
    failure_remediation: >
      Grant the API client the Hosts:read scope in Falcon
      → Support → API Clients.

  fetch:
    pagination: cursor
    page_size: 5000
    max_pages: 2000
    stream_downstream: true
    estimated_calls: "1 per 5000 hosts + 1 detail call per 100"
    rate_limit_domain: [crowdstrike, tenant, hosts]

  cursor:
    supported: true
    field: last_seen
    advance_when: after_durable_write

  coverage:
    emits_window: true
    scope: [cid]
    completeness_signal: "empty scroll token returned"

  schedule: { interval: 4h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    entities:
      - type: ASSET
        subtype: endpoint
        canonical_key_source: [cloudid, hwuuid, fqdn, mac]
        properties: [os_version, platform, agent_version,
                     last_seen, ou, site_name, external_ip,
                     local_ip, mac_address, serial_number]
    relationships:
      - predicate: PROTECTS
        from: SECURITY_CONTROL
        to: ASSET
        confidence: 0.99
        significant_attributes: [control_type]
        precondition: "sensor status is normal, not reduced_functionality"

  mapping:
    device_id:        properties.vendor_id
    hostname:         canonical_key via fqdn
    serial_number:    canonical_key via hwuuid (preferred)
    instance_id:      canonical_key via cloudid (preferred where present)
    mac_address:      canonical_key via mac
    os_version:       properties.os_version
    agent_version:    properties.agent_version
    last_seen:        last_seen
    status:           → PROTECTS precondition

  health:
    success_criteria: "hosts_enumerated >= 1 AND auth_errors == 0"
    baseline_metric: hosts_enumerated
    silent_threshold_pct: 10
    staleness_threshold: 12h
    field_presence_monitored: [serial_number, instance_id]

  depends_on:
    hard: []
    soft: []
    consumed_by: [E6, E9, E12]

  failure_modes:
    - condition: host has no serial, no instance id, no FQDN
      behaviour: >
        Falls back to mac. Resolution confidence drops to 0.85 and
        the entity is flagged weak-key.
    - condition: sensor in reduced_functionality mode
      behaviour: >
        ASSET emitted, PROTECTS edge NOT emitted. The host is
        covered on paper and not in practice, which is a finding.

  fixtures:
    required:
      - 2-page scroll, 8000 hosts
      - a cloud host with an instance id
      - an on-prem host with only a serial and FQDN
      - a host with neither serial nor instance id
      - a host in reduced_functionality
      - a 429 mid-scroll
      - a zero-host tenant
```

---

## 7. How specs are stored

```
  Specs live BESIDE the connector, not in documentation:

    connectors/aws/
      manifest.yaml            the connector manifest (13 §VIII)
      collectors/
        iam.roles.yaml         this format
        iam.policies.yaml
        ...
      policies/
        aws-reader.json        generated from requires_scope
      fixtures/
        iam.roles/
          3page-250roles.json
          oidc-condition.json
          ...

  The documentation series (this one) carries the FORMAT and the
  worked specs for the collectors being built. The specs themselves
  are code-adjacent, versioned with the connector, and validated in CI.
```

---

*Next: [Identity and IAM specs](02-spec-identity-and-iam.md)*

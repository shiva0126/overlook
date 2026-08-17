# E9 Posture · E10 Correlation · E11 Classification

**Series:** [Engine documentation](00-index.md) · **v1:** E9 required · E10 and E11 deferred

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

Three engines documented together because they occupy the same slot in the derivation stage and are frequently — and wrongly — lumped together as "the analytics engine."

They have nothing in common except their position. They differ in algorithm, state model, resource profile and scheduling, which is exactly why treating them as one component produces a subsystem with five conflicting resource profiles and no coherent scheduling model.

```
  E9   POSTURE          stateless rules over current state
  E10  CORRELATION      stateful, windowed, over event streams
  E11  CLASSIFICATION   content inspection over data sources
```

E7 → E8 is ordered. These three are independent of each other and of that pair.

---

## 2. Position

```
  ALL THREE
    INPUT   observations (collected and derived)
    OUTPUT  FINDING or PROPERTY observations → E12

  E9   reads current entity and edge state
  E10  reads a windowed event stream, holds window state
  E11  reads sampled content from data sources, holds crawl cursors
```

---

## 3. Mechanics

### 3.1 E9 — Posture Rules

```
  SHAPE      stateless predicate evaluation over current state
  RESOURCE   cheap, CPU-bound, batch
  CADENCE    once per collection cycle, after E7/E8 complete
  STATE      none — the rule set is content

  A rule is a predicate over entities, edges and properties:

    RULE  ai.mcp.overscoped_filesystem
      when  MCP_SERVER m
            AND m.type = "filesystem"
            AND m CAN_READ DATASTORE d
            AND d.data_classes CONTAINS any of [PII, PCI, PHI]
            AND EXISTS AI_AGENT a WHERE a INVOKES m
      then  FINDING severity=HIGH
            evidence = [m.config_ref, d.classification_ref]
```

Rules are shipped content, versioned alongside escalation primitives. They differ from primitives in output: a primitive **synthesizes an edge**; a rule **raises a finding** about state that already exists.

**Determinism is the point.** A posture finding is either true or false given the state, it is reproducible, and it is arguable with an engineer. That is why E9 is in v1 and E10 is not.

### 3.2 E10 — Correlation

```
  SHAPE      stateful, windowed, over event sequences
  RESOURCE   memory-bound — window state grows with cardinality
  CADENCE    continuous
  STATE      per-key sliding or tumbling windows

  PATTERNS
    sequence     failed auth ×20 → success → privilege change
    threshold    datastore returns 400× its normal row count
    first-seen   this service account has never called this API
    rare-value   this identity has never authenticated from this ASN
    absence      a scheduled job did not run

  OUTPUT   FINDING or EVENT_SUMMARY observations.
           Raw events NEVER leave the collector.
```

**Why it is deferred.** Overlook's findings are structural, not temporal. "This permission creates a path" needs no time window. Correlation answers "is something happening?" — which is the question an XDR exists for, and which Meridian's CrowdStrike already answers better than we would. E10 earns its place only where a temporal signal materially changes a *structural* conclusion, such as detecting that a new admin grant appeared three hours after a suspicious authentication.

**Its real hazard is unbounded memory.** Window state scales with key cardinality, and a high-cardinality key on a busy source will exhaust the box. Every window needs a declared cardinality bound and an eviction policy before it is enabled.

### 3.3 E11 — Classification

```
  SHAPE      content inspection
  RESOURCE   IO-heavy, long-running, must be ISOLATED
  CADENCE    rolling and partitioned — never "scan everything"
  STATE      crawl cursors, coverage percentage by age

  PIPELINE
    read-only sample → detect → classify → discard content
    → emit PROPERTY observations only

    detection: regex + validators (Luhn, national ID checksums)
               + entropy analysis for secrets
               + provider-specific credential patterns
               + NER for names and addresses
               + customer-defined patterns

    OUTPUT is a label and a bucketed count. Never content.
```

**Why it is deferred at Meridian.** They own Forcepoint DLP. Ingesting its classification results as an overlay gives us `DATA_CLASS` properties for free, without running our own crawl over 40 TB of file shares. Building E11 duplicates a control the customer already paid for.

**Isolation is non-negotiable.** E11 runs in its own process with a hard resource ceiling and the lowest priority, and cannot borrow capacity. A classification crawl starving identity collection is the single most common way collectors like this fall over (`../04 §26`).

---

## 4. Considerations

**Do not merge them.** One "analytics engine" with three resource profiles cannot be scheduled coherently. Keep three subsystems sharing a state store.

**E9 rules must state their evidence.** A finding without an evidence reference cannot be defended when challenged.

**Rule noise is the failure mode of E9.** A rule that fires 4,000 times is worse than no rule. Every rule needs an expected-cardinality estimate, and one that exceeds it should be suppressed and flagged rather than emitted.

**E9 findings should be suppressible with a reason, not just dismissed.** "Accepted risk, ticket ENG-4471, review 2027-01" is a first-class state, and it must survive rule version changes.

**E10 windows need eviction policies before they need patterns.** Design the memory model first.

**E11 must never retain content.** Sample, classify, discard. The classification result is a property; the content is not evidence and should not be stored — the file remains where it is, and the customer's own DLP holds the detail.

**Classification coverage is a percentage with an age distribution.** "88% classified, median age 6 days" is honest; "classified" is not.

---

## 5. Failure modes

| Engine | Failure | Behaviour |
|---|---|---|
| E9 | Rule fires excessively | Cardinality bound exceeded → suppressed, flagged, not emitted |
| E9 | Rule references stale state | Findings computed against a stale subgraph are marked stale, not withdrawn |
| E9 | Rule content regression | Version-stamped findings are bulk-retractable, like primitives |
| E10 | Window memory exhaustion | Declared cardinality bound + eviction; breach sheds the window and alarms |
| E10 | Clock skew across sources | Sequence rules produce nonsense — requires source-time vs receive-time discipline |
| E11 | Crawl starves the collector | Isolated process, hard ceiling, cannot borrow capacity |
| E11 | Scanning a source under load | Throttle by target-system load, not only by our own |
| E11 | Sampling misses the sensitive data | Coverage reported as a percentage with confidence, never as a binary |

---

## 6. Contracts

```
  E9   every finding carries rule id, version, evidence refs and
       the entities it concerns; findings are bulk-retractable by
       rule version; suppressions survive rule updates

  E10  raw events never leave the collector; every window declares
       a cardinality bound and eviction policy

  E11  content is never retained; output is a label plus a bucketed
       count; scans are read-only, throttled and resumable
```

---

## 7. Scope

```
  BUILD IN V1
    E9 posture rule engine + the v1 rule set
       (AI/MCP rules, identity hygiene, cloud misconfiguration
        combinations, toxic combinations)
    finding lifecycle: open, suppressed-with-reason, resolved,
       withdrawn-on-rule-retraction

  DEFER
    E10 correlation      — structural findings need no time window;
                           revisit when a temporal signal changes a
                           structural conclusion
    E11 classification   — consume the customer's DLP instead;
                           build only for customers without one
```

---

## 8. Example: Meridian, one cycle

### 8.1 E9 — 01:52, after closure and escalation matching

```
  RULE SET  184 rules evaluated against current state
  RUNTIME   38 seconds across both Edge Collectors

  FINDINGS RAISED                                            open
  ─────────────────────────────────────────────────────────  ────
  IAM-001  escalation to administrator                         41
  IAM-004  OIDC trust with insufficient subject condition        3  ← CRIT
  IAM-005  Entra role reaching the Azure resource plane          3  ← CRIT
  IAM-020  orphaned privileged NHI (no owner)                   28
  IAM-022  credential older than 365 days                       61
  IAM-026  human identity without MFA reaching a crown jewel     4  ← CRIT
  IAM-027  break-glass account excluded from Conditional Access  2
  IAM-062  RBCD writable by a low-privilege principal             8  ← CRIT
  IAM-064  GPO writable by a non-admin, linked to servers         2  ← CRIT
  AI-083   MCP server credential exceeds the agent's need        31
  AI-084   RAG retrieval identity broader than query audience     0
  ─────────────────────────────────────────────────────────  ────
                                                              183

  SUPPRESSED BY CARDINALITY BOUND
    IAM-022 was written expecting ~10 matches and found 61.
    Below its bound of 100, so it emits — but the Controller shows
    "61 matches, above the expected 10" so the operator can decide
    whether the rule or the environment is wrong.

    A different rule — "S3 bucket without access logging" — matched
    4,100 times against a bound of 200.
    → SUPPRESSED. Flagged as "rule may be miscalibrated for this
      environment." Not shown as 4,100 findings.
```

### 8.2 The rule that produced Meridian's flagship finding

```
  RULE  ai.mcp.unmanaged_credential_path            version 4

    when  MCP_SERVER m ON ASSET a
          AND m.credential_types CONTAINS "github_token"
          AND IDENTITY i AUTHENTICATES_TO a
          AND REPOSITORY r WHERE i CAN_WRITE r
          AND r HAS_OIDC_TRUST role
          AND role reaches DATASTORE d
          AND d.criticality > 80
    then  FINDING severity=CRITICAL

  MATCHED  1 time.

    m  = mcp-filesystem on lt-4471, env key GITHUB_TOKEN
         (presence only — the value was never collected)
    a  = AST-77c2
    i  = Priya S
    r  = meridian/payments-api
    role = GHADeployRole, via the OIDC trust E8 flagged as too broad
    d  = prod-payments-db, criticality 95, PII + PCI

  EVIDENCE
    agent scan ref     sha256:c41d…8f2a   (the config file, hashed)
    trust policy ref   sha256:8a1f…c4d2
    DLP classification ref  sha256:2b7e…9d13

  Every hop is independently defensible, and every hop has a
  retrievable artifact behind it.
```

### 8.3 E10 — what it would have added, and why it waited

```
  Meridian's CrowdStrike already does sequence detection on hosts,
  and does it better than we would.

  The one case where E10 would earn its place here:

    at 03:12, a new admin role assignment appeared in Entra
    at 03:09, an anomalous authentication for that identity was
              recorded by CrowdStrike

    A temporal correlation between a STRUCTURAL change (new
    privilege) and a BEHAVIOURAL signal (odd authentication) is a
    materially stronger finding than either alone.

  That is the shape of the E10 we would eventually build: not a
  general event-correlation engine, but a narrow one that ties
  graph changes to behavioural signals from connectors we already
  ingest.

  Until then, the change feed already reports "a new admin grant
  appeared 3 hours ago" — which is most of the value, structurally,
  with no windowing engine at all.
```

### 8.4 E11 — replaced by Forcepoint

```
  WHAT WE WOULD HAVE BUILT
    a rolling crawl over 40 TB of SMB shares, Oracle, MSSQL and
    S3 — weeks of IO, its own isolated process, a permanent
    pattern-content library, and a conversation with Meridian's
    storage team about scan windows

  WHAT WE DID INSTEAD
    ingested Forcepoint's classification results as a band-5
    overlay connector

    → 4,100 datastores received DATA_CLASS properties
    → prod-payments-db labelled PII + PCI, which is what set its
      criticality to 95 and made it a crown jewel
    → coverage: 71% of datastores classified, median age 4 days
      (honestly reported — the remaining 29% are shares Forcepoint
       does not cover, and the Controller says so)

  THE FINDING THAT DEPENDED ON IT
    Without the PII+PCI label on prod-payments-db, the critical
    path in 8.2 terminates at "a database" rather than "4.2M
    customer records," its criticality is unset, and it never
    reaches the top of the list.

  E11 was deferred. The capability was not.
```

---

*Next: [Graph Engine](09-graph-engine.md)*

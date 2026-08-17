# E12 — Graph Engine

**Series:** [Engine documentation](00-index.md) · **v1:** required · **one of the four hardest**

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

The Graph Engine is where everything converges. Every observation from every source, after normalization, resolution and derivation, becomes a node or an edge here — with a lifetime, a confidence and a provenance trail.

It is also where the product's defining question gets answered: not *what happened*, but *what is possible*. That requires traversal, not search, and it requires knowing when something stopped being true — which turns out to be the hardest part.

---

## 2. Position

```
  INPUT   observations from E6 (collected) and E7/E8/E9 (derived)
          coverage windows from E15

  OUTPUT  the graph itself: entities, edges, materialized closures
          change events (the diff stream)
          query results: neighbourhoods, paths, blast radius

  SERVES  Mode 1: the Controller UI directly
          Mode 2: E13 Fact Builder
          both:   the Resolution Directory's entity view
```

---

## 3. Mechanics

### 3.1 Bitemporal by default

Nothing is ever hard-deleted. Every node and edge carries a lifecycle.

```
  first_seen    when we first observed it
  last_seen     most recent observation
  removed_at    when a COMPLETE enumeration failed to observe it
                (null while live)

  This single decision buys, for free:
    "this path has existed 84 days"        → path age, a better
                                             prioritisation signal
                                             than severity
    "this admin grant appeared 3h ago"     → new-privilege detection
    "show me the graph last Tuesday"       → investigation and audit
    "did our fix actually break the path?" → verification, which
                                             closes the response loop
```

The alternative — daily snapshots — costs far more storage and answers fewer questions.

### 3.2 Coverage windows govern retraction

The most dangerous operation in the system, and the reason E15 emits coverage windows at all.

```
  removed_at is set ONLY when a source that WOULD have reported the
  edge ran to COMPLETION and did NOT report it.

  GIVEN a complete window:
    collector aws.iam.roles · scope account:123456789012
    started 04:00 · completed 04:03 · enumeration_complete: true
    → any role edge in that scope, from that collector, not observed
      within the window → tombstone

  WITHOUT a complete window:
    → retract NOTHING
    → mark the affected subgraph STALE with the reason
    → surface staleness ON THE PATH ITSELF, not on a health page
```

**Why this matters more than it sounds.** A connector breaks at 04:00 and nobody notices. Without coverage windows, 8,400 edges go unobserved, get retracted, and the customer's exposure score *improves* while their actual exposure is unchanged. One bug, total loss of credibility.

### 3.3 Storage shape — Postgres

```
  entities
    token TEXT PK · type · subtype · properties JSONB
    criticality · first_seen · last_seen · removed_at
    confidence · sources TEXT[]

  edges
    from_token · to_token · predicate · attr_signature
    properties JSONB · weight · confidence
    first_seen · last_seen · removed_at
    evidence_ref · collector_id
    PK (from_token, to_token, predicate, attr_signature)

  closure_can_assume        materialized transitive closure
    from_token · to_token · depth · min_confidence · via_path
  closure_member_of         same shape

  coverage_windows
    collector · scope · started · completed · complete · count
```

**Index strategy — two things carry the weight:**

```
  1  PARTIAL INDEXES ON removed_at IS NULL
     Tombstones accumulate forever. Every hot query must exclude
     them at the index level, not in the predicate.

       btree (from_token, predicate) WHERE removed_at IS NULL
       btree (to_token,   predicate) WHERE removed_at IS NULL   ← reverse BFS
       GIN   (properties)                                       ← attribute filters
       btree (last_seen)                                        ← staleness sweeps

  2  MATERIALIZED CLOSURE for the two dense predicates
     Computing CAN_ASSUME transitively at query time is where a
     naive implementation dies. Maintain incrementally on edge
     change, never on query.
```

### 3.4 Incremental closure maintenance

```
  edge added/removed on a closed predicate
    → identify affected closure rows via the reverse index
    → recompute only that region
    → typical: 40 rows, ~200 ms
    → versus full closure recompute: ~25 minutes

  Cycles collapse (A→B→A becomes A↔B at depth 1).
  Depth bounded at 6 for CAN_ASSUME, 12 for MEMBER_OF
  (AD nesting genuinely reaches 9 in the wild).
```

### 3.5 Path finding

Reverse BFS from crown jewels, not all-pairs.

```
  for each crown_jewel:
    frontier = {crown_jewel}
    for depth in 1..8:
      frontier = incoming_traversable_edges(frontier)
      prune  cumulative_weight > threshold
      prune  confidence < 0.5
      record predecessors
    emit paths whose terminal node is a START CONDITION

  Implemented as a recursive CTE over edges, seeded from crown
  jewels, using the reverse-direction partial index.

  Then: score → collapse by choke point → diff against last run.
  Only NEW paths generate notification.
```

**Path explosion control** matters more than path computation: depth limit, weight budget, choke-point collapsing, equivalence classes, and crown-jewel scoping. Presenting fewer than 50 actionable items out of 500,000 computed paths is the product problem, and it is harder than the engineering one.

### 3.6 The change feed

A byproduct of the writer, not a separate system.

```
  every write → diff against prior state → ChangeEvent
    { type, subject, predicate, object, before, after,
      detected_at, source, significance }

  significance is COMPUTED, not raw:
    created a new path to a crown jewel   → CRITICAL
    increased an existing path score >20  → HIGH
    granted privilege where none existed  → HIGH
    removed a control (PROTECTS edge)     → HIGH
    routine churn (autoscaling, ephemeral pods) → SUPPRESS
```

**Suppression by equivalence class, not by rate.** A Kubernetes cluster generates thousands of node/edge changes an hour from normal autoscaling. Learn that pods matching a workload template with an identical service account are one logical entity, and report the template's changes rather than each pod's.

---

## 4. Considerations

**Two graphs, one implementation.** The collector graph holds plaintext canonical keys, this site's scope, full condition detail, short retention. The console graph holds tokens, merged scope, resolved edges, full bitemporal history. Same engine, two configurations (`../04 §15.1`). Design for both from the start rather than building for one and retrofitting.

**The access layer must be an interface.** Postgres is the right v1 choice and the wrong choice above ~10M edges. Everything above the graph — path engine, blast radius, change feed — must be swappable onto a different store without rewriting.

**Confidence is a first-class column, not metadata.** Paths are only as trustworthy as their weakest edge, and that must be visible in every query result.

**Tombstone compaction is required, eventually.** At 1–5% edge churn per day, tombstones exceed live edges within a year. Compact beyond the retention horizon, but never before — the temporal questions are the point.

**`PROTECTS` does not traverse.** A control does not create reachability; it reduces the probability of traversal. Modelling it as traversable produces nonsense paths ("attacker traverses the EDR"). It reduces the weight of adjacent edges instead.

**Equivalence classes are underrated.** 380 identical Terraform-deployed roles, or 400 EC2 instances in one ASG with the same role, are one class with a multiplicity — faster to compute and vastly better in the UI.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Retraction without a coverage window | **Findings silently resolve; exposure score improves for no reason.** Credibility-ending | Coverage windows are mandatory; retraction is impossible without one |
| Closure explosion | Memory exhaustion, stalled cycle | Depth bounds, confidence pruning, cycle collapse, equivalence classes |
| Missing partial index | Query time degrades as tombstones accumulate — slowly, then suddenly | Partial indexes from the first migration |
| Path explosion | 47,000 paths shown, none actionable | Choke-point collapsing, crown-jewel scoping, hard display cap |
| Duplicate edges | Inflated path counts, destroyed trust in the numbers | Semantic identity on (from, to, predicate, attr_signature), upsert only |
| Change feed floods | Unreadable within a day | Equivalence-class suppression, computed significance |
| Graph writes contend with reads | Analyst queries slow during collection | Separate connection pools; closure maintenance batched |

---

## 6. Contracts

```
  MUST GUARANTEE
    no retraction without a complete coverage window
    nothing is hard-deleted; removal is a tombstone
    every node and edge carries first_seen, last_seen, confidence
      and its contributing sources
    identical observations upsert rather than duplicate
    incremental closure yields the same result as a full recompute
    every query result exposes the confidence of its weakest link
```

---

## 7. Scope

```
  BUILD IN V1
    entities, edges, bitemporal lifecycle
    coverage-window-governed retraction
    materialized closure for CAN_ASSUME and MEMBER_OF
    incremental closure maintenance
    reverse-BFS path finding with choke-point collapsing
    change feed with computed significance
    the storage interface, so Postgres is replaceable

  DEFER
    tombstone compaction        (needed at ~1 year, not at launch)
    equivalence-class learning  (start with declared classes)
    graph simulation / what-if  (Break Attack Path — high value,
                                 but after paths are trustworthy)
    a bespoke graph engine      (only when Postgres actually breaks)
```

---

## 8. Example: Meridian's graph

```
  STEADY STATE, both Edge Collectors combined

    entities                    ~2.9 million
      identities        14,100   (12,000 human + 2,100 NHI)
      roles              8,400
      assets            11,600   (8,500 endpoints + 1,100 VMs + cloud)
      datastores         4,100
      networks             340
      AI entities          510   (agents, MCP servers, models)
      other            ~2.86M    (policies, groups, repos, k8s objects)

    edges                       ~2.9 million live, ~180k tombstoned
      CAN_READ / CAN_WRITE   1.42M
      MEMBER_OF                412k   (after AD nesting closure)
      CAN_ASSUME               338k
      ROUTES_TO / CONNECTS_TO   24k
      synthesized (E8)           270
      other                   ~700k

    materialized closures
      closure_can_assume       1.1M rows, max depth 6
      closure_member_of        412k rows, max depth 9

    churn                      ~2.4% / day
```

### 8.1 One night's retraction decision

```
  01:58 — E12 applies coverage windows from the cycle.

  38 COMPLETE WINDOWS RECEIVED
    aws.iam.roles · account 123456789012 · complete · 412 roles
      → 6 roles in the graph from this collector+scope were NOT
        observed → TOMBSTONED. removed_at = 01:58.
      → 41 edges from those roles tombstoned with them.
      → 2 findings referencing them WITHDRAWN with a reason.

    ad.users · forest corp.meridian.com · complete · 12,004 users
      → 11 users not observed → tombstoned (leavers, correctly)

  3 PARTIAL MARKERS RECEIVED
    aws.* · account 445566778899 · INCOMPLETE (403 at preflight)
      → 1,204 entities and 4,100 edges from this account marked
        STALE, reason "credential expired 02:00"
      → NOTHING TOMBSTONED
      → any path traversing those edges now renders with a
        staleness badge ON THE PATH

    azure.rbac · subscription 7 · INCOMPLETE (throttled mid-page)
      → 340 assignments marked stale, nothing removed

    fortigate · fw-branch-02 · INCOMPLETE (parse failure)
      → 1,900 reachability edges marked stale

  WHAT WOULD HAVE HAPPENED WITHOUT COVERAGE WINDOWS
    5,304 edges silently tombstoned across three sources.
    Meridian's exposure score drops 14 points overnight.
    Eleven open findings auto-resolve.
    Nothing in their environment changed.
```

### 8.2 The critical path, assembled

```
  Reverse BFS from crown jewel DST-prod-payments (criticality 95):

    DST-prod-payments  ◄─CAN_READ─  ROL-ec2app
        w 0.10 · conf 0.99 · E7, unconditional
    ROL-ec2app         ◄─CAN_ASSUME─ ROL-ghadeploy
        w 0.15 · conf 0.92 · E8 SYNTHESIZED
        primitive aws.privesc.passrole_lambda v3
    ROL-ghadeploy      ◄─CAN_ASSUME─ PIP-gha-any-meridian-repo
        w 0.10 · conf 0.95 · E8 SYNTHESIZED
        primitive aws.oidc.subject_condition_too_broad
    PIP-gha            ◄─CAN_WRITE─  REP-payments-api
        w 0.10 · conf 0.99 · E7
    REP-payments-api   ◄─CAN_WRITE─  IDN-priya
        w 0.10 · conf 0.99 · E7
    IDN-priya          ◄─AUTHENTICATES_TO─ AST-lt-4471
        w 0.05 · conf 0.94 · E6 via Resolution Directory
    AST-lt-4471        holds MCP-filesystem with a GitHub token
        conf 0.91 · agent observation, credential PRESENCE only

  IDN-priya is a START CONDITION (phishable human, MFA present)

  PATH CONFIDENCE  0.91   — the weakest link is the MCP credential
                            inference, and the UI says so
  PATH AGE         84 days
  SCORE            94 / 100 · CRITICAL

  CHOKE POINT ANALYSIS
    PIP-gha ─CAN_ASSUME→ ROL-ghadeploy appears in 1,240 paths,
    of which 9 reach crown jewels and 3 are critical.
    → tighten the OIDC subject condition to repo + ref
    → eliminates 1,240 paths
    → estimated effort: 40 minutes
```

### 8.3 The change feed the next morning

```
  06:14  EDGE_ADDED    IDN-svc-reporting CAN_ASSUME ROL-dbadmin
         significance CRITICAL — creates a new path to
         prod-payments-db. Detected 3 minutes after the IAM change,
         via incremental closure (640 ms).

  06:22  EDGE_REMOVED  41 edges from 6 tombstoned roles
         significance LOW — routine decommissioning

  06:31  1,847 pod-level changes in the EKS clusters
         significance SUPPRESSED — equivalence class
         "workload template payments-api, identical service account"
         → reported as ONE change to the template, not 1,847 events
```

---

*Next: [Fact Builder](10-fact-builder.md)*

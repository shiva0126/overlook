# E13 — Fact Builder

**Series:** [Engine documentation](00-index.md) · **v1:** Mode 2 only · **one of the four hardest**

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

Everything upstream of the Fact Builder is mechanical — receive, parse, normalize, resolve, evaluate. Everything downstream is mechanical — tokenize, sign, queue, ship. **The Fact Builder is where the judgement lives.**

It decides when many observations become one assertion, which of several disagreeing sources to believe, when an assertion has changed enough to be worth transmitting, and when something that was true has stopped being true.

It exists only in Mode 2 (`../10 §5`). In Mode 1 the graph is the product and nothing crosses a boundary, so there is nothing to build facts for.

---

## 2. Position

```
  INPUT   observations (collected and derived)
          the existing local fact store
          coverage windows

  OUTPUT  Security Facts, in plaintext, ready for the Privacy Gate

  NOTE    E13 runs on plaintext canonical keys. Tokenization happens
          AFTER it, in E14 — merging, arbitration and retraction all
          require the real values to work.
```

---

## 3. Mechanics

### 3.1 Observation versus Fact

```
  OBSERVATION   one sighting, one source, one moment.
                Immutable. Cheap. Numerous. Never leaves.

  FACT          the accumulated, deduplicated, arbitrated assertion
                built from many observations.
                Mutable. Expensive. Few. Leaves.

  14,882 observations of "svc-deploy CAN_ASSUME DevOpsAdmin"
  collected over 61 days from 3 sources
        ↓
  ONE fact · observation_count 14,882 · first_seen 61 days ago
  · last_seen 4 minutes ago · sources [iam, cloudtrail,
    access_analyzer] · confidence 0.99
```

That collapse is where the 200,000:1 reduction is finally realised, and it is the structural difference from an event pipeline (`../11 §3.2`).

### 3.2 The merge key

Fact identity is **not** `fact_id` (a per-emission ULID used only for transport dedup). Semantic identity is:

```
  merge_key = hash(
      fact_type,
      subject.canonical_key,
      predicate,
      object.canonical_key,
      significant_attributes_signature
  )

  No tenant component — one collector serves one customer, and on
  the console side each customer's graph is a separate store
  (../09 §2.1).
```

### 3.3 Significant attributes — the decision that defines everything

An attribute is significant if changing it means **this is a different assertion**, not an updated one.

```
  SIGNIFICANT — part of the key
    mechanism         sts_assume_role vs oidc_federation are
                      genuinely different edges
    conditions        conditional vs unconditional are different
    privilege_level   a material change in the assertion
    granted_via       the path by which it was granted

  NOT SIGNIFICANT — merge, do not split
    last_seen, observation_count, confidence
    evidence_ref      accumulates as a list
    source            accumulates as a list
    collection metadata

  THE TWO FAILURE MODES
    TOO MANY significant → fact explosion. Every observation
                           creates a "new" fact. The reduction
                           never happens and the wire fills.
    TOO FEW  significant → distinct realities collapse. A
                           conditional edge and an unconditional
                           edge become indistinguishable.
                           The graph lies.
```

**Significance is declared per predicate in the schema, not inferred.** It is a small, reviewable table, and it is expensive to change later because altering it re-partitions every fact ever built.

### 3.4 Granularity — one fact per edge, with provenance inside

```jsonc
{
  "subject": "...", "predicate": "CAN_ASSUME", "object": "...",
  "confidence": 0.99,
  "sources": [
    {"id": "aws.iam",             "confidence": 0.99, "last_seen": "...", "agrees": true},
    {"id": "aws.access_analyzer", "confidence": 0.97, "last_seen": "...", "agrees": true},
    {"id": "aws.cloudtrail",      "confidence": 0.85, "last_seen": "...", "agrees": true}
  ],
  "disagreement": false
}
```

The alternative — one fact per (edge, source) — preserves provenance but multiplies wire volume by N and pushes arbitration to a place with less context. This keeps Model A's volume and Model B's provenance, and arbitration happens on the collector where full plaintext context exists.

### 3.5 Confidence arbitration

```
  INPUTS
    source authority        per (connector, collector, predicate)
                            aws.iam on CAN_ASSUME      = 0.99 authoritative
                            cloudtrail on CAN_ASSUME   = 0.85 observational
    resolution confidence   how sure are we WHO this is about?
    corroboration           independent sources agreeing
    recency                 decays toward the staleness horizon
    verification            provider simulation confirmed → 0.99

  COMBINATION
    confidence = min(resolution_confidence,
                     authority_weighted_agreement)
                 × recency_factor
                 × corroboration_boost

  The min() on resolution_confidence is deliberate: a fact can never
  be more trustworthy than our certainty about who it concerns.
```

### 3.6 Handling actual disagreement

```
  aws.iam says        no CAN_ASSUME edge exists
  aws.cloudtrail says this principal assumed that role yesterday

  DO NOT silently pick one.
    emit with disagreement=true
    confidence drops to the lower value
    record both claims in sources[]
    raise an internal diagnostic

  This case is diagnostically valuable: it means either a collection
  gap or a closure bug, and its RATE is the best available proxy for
  closure correctness (../06 §4 self-diagnostic signal).
```

### 3.7 Emission policy — where 12 MB/day comes from

Without this, the collector re-sends the entire graph continuously.

```
  NEW fact                        → emit immediately
  SIGNIFICANT attribute changed   → emit immediately (privilege changed)
  confidence changed > 0.05       → emit
  disagreement flag flipped       → emit
  fact RETRACTED                  → emit immediately
  ONLY last_seen / count changed  → DO NOT emit.
                                    Wait for the heartbeat (24h default)
                                    or the staleness horizon, whichever
                                    is sooner.

  A stable environment produces almost no emissions — which is
  correct. A graph that has not changed should not generate traffic.
```

### 3.8 Retraction

```
  A fact may be retracted ONLY given a COMPLETE coverage window
  from a source that WOULD have reported it.

  Retraction emits a fact with removed_at set. Nothing is deleted.

  Without a complete window: retract nothing, mark stale.
  (Same contract as E12 — stated in both places because violating
   it in either is equally fatal.)
```

---

## 4. Considerations

**Idempotency on replay is mandatory.** After a partition the collector resends; after a restart it may rebuild from local state. The console upserts on semantic identity, taking `max(last_seen)`, `min(first_seen)`, and the highest-confidence source. Getting this wrong produces duplicate edges, which inflate path counts and destroy trust in every number shown.

**The merge window is a real trade.** Merging forever gives maximum reduction and loses the ability to say "this edge was observed 400 times yesterday and twice today." Bucketed observation counts per period are a reasonable middle.

**Fact volume is not proportional to data volume.** Meridian's 1.24 TB of firewall syslog produces ~40 facts a day. Sizing the outbound queue on input volume is a category error.

**Arbitration needs source authority as content.** The authority table — which connector is authoritative for which predicate — is shipped, versioned config, not code. It changes as connectors improve.

**Heartbeat cadence versus staleness horizon.** If the heartbeat is longer than the console's staleness horizon, facts age out and reappear, generating churn. Heartbeat must be shorter, and the relationship enforced rather than configured independently.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Merge key too broad | Distinct realities collapse; conditional and unconditional edges become one | Significance declared per predicate, reviewed, versioned |
| Merge key too narrow | Fact explosion; the reduction never happens | Same table; monitor facts-per-observation ratio |
| Emission policy missing | Entire graph re-sent continuously; 12 MB becomes 12 GB | Emission policy is not optional |
| Non-idempotent replay | Duplicate edges in the console | Semantic identity + upsert, never insert |
| Retraction without coverage | Findings silently resolve | Coverage window required, enforced |
| Disagreement silently resolved | A real collection gap or closure bug goes unnoticed | `disagreement=true` is a first-class field with a tracked rate |
| Heartbeat > staleness horizon | Facts flap in and out | Enforce the relationship |

---

## 6. Contracts

```
  MUST GUARANTEE
    identical observations produce an identical merge key
    replay is idempotent — no duplicates on the console
    no retraction without a complete coverage window
    every fact carries provenance, confidence and evidence refs
    unchanged facts do not transmit outside the heartbeat
    disagreement is recorded, never silently resolved
```

---

## 7. Scope

```
  NOT IN V1 — Mode 1 has no boundary to cross.

  BUT THE CONTRACT IS DESIGNED NOW (../10 §5):
    the Security Fact schema
    the significant-attribute table per predicate
    the source authority table
    the emission policy

  Because Mode 2 must be a switch, not a rewrite — and because
  the significant-attribute table is one of the three decisions
  that is expensive to reverse (../04 §30 D1, D2).
```

---

## 8. Example: Meridian, one night of fact building

```
  02:04 — E13 runs after E12 has applied coverage windows.

  INPUT   ~4.1 million observations from the cycle
  OUTPUT  2,914 facts emitted · 11.8 MB after the Privacy Gate
```

### 8.1 The collapse

```
  OBSERVATIONS IN                                    FACTS OUT
  ────────────────────────────────────────────────   ─────────
  AWS IAM: 2.1M capability entries (E7)              ~310,000 held
    of which changed since yesterday                     1,204 emitted
  AD: 720,000 ACEs → 4,100 security-relevant edges      180 emitted
  Firewall aggregates: 120,000                           40 emitted
  NetFlow aggregates: 180,000                            50 emitted
  Agent reports: 51,000                                 310 emitted
  Entra delta                                            94 emitted
  CrowdStrike: 8,500 host records                       420 emitted
  Forcepoint: 4,100 classifications                      82 emitted
  E8 synthesized edges                                  270 held
    of which new or changed                              14 emitted
  E9 findings                                           183 held
    of which new                                        520 emitted
                                                     ─────────
                                                        2,914

  HELD BUT NOT EMITTED: ~2.9 million facts whose only change was
  last_seen. Their heartbeat is not due. This is the emission
  policy doing the entire job.
```

### 8.2 One fact, merged

```
  MERGE KEY INPUT
    fact_type   RELATIONSHIP
    subject     email:priya.s@meridian.com
    predicate   CAN_ASSUME
    object      arn:aws:iam::123456789012:role/GHADeployRole
    significant attributes
      mechanism        sts_assume_role
      conditions       []                    ← unconditional
      privilege_level  ELEVATED
      granted_via      identity_policy:pol-4f2a

  EXISTING FACT FOUND — first observed 2026-05-22

  MERGE
    first_seen         2026-05-22T09:14:22Z   (unchanged, min)
    last_seen          2026-08-14T00:41:03Z   (updated, max)
    observation_count  8,412 → 8,455
    sources            [aws.iam 0.99, aws.access_analyzer 0.97]
                       both agree, both refreshed
    confidence         recomputed → 0.99

  SIGNIFICANT ATTRIBUTES UNCHANGED → NOT EMITTED.
  Next heartbeat due in 6 hours.
```

### 8.3 One fact that was emitted

```
  NEW OBSERVATION from E8, 01:47

    subject     PIPELINE:github/meridian/*
    predicate   CAN_ASSUME
    object      arn:aws:iam::123456789012:role/GHADeployRole
    significant
      mechanism        oidc_federation        ← DIFFERENT mechanism
      conditions       [sub:repo:meridian/*]
      privilege_level  ELEVATED
      synthesized      true
      primitive        aws.oidc.subject_condition_too_broad v2

  → different merge key from the fact in 8.2, because MECHANISM is
    significant. These are genuinely two different edges to the
    same role: one for Priya directly, one for any workflow in any
    Meridian repo.

  → NEW FACT. Emitted immediately.
  → and it is the hop that completes Meridian's critical path.

  Had mechanism NOT been significant, these two would have merged
  into one edge and the OIDC exposure would have been invisible —
  hidden inside a fact that already existed.
```

### 8.4 A disagreement

```
  aws.iam          reports NO CAN_ASSUME from svc-reporting to
                   role/dbadmin — the trust policy does not name it
  aws.cloudtrail   reports svc-reporting successfully called
                   sts:AssumeRole on role/dbadmin at 03:12

  E13 does not choose.

    emit  disagreement: true
          confidence: 0.85  (dropped to the lower source)
          sources: [{aws.iam, 0.99, agrees:false},
                    {aws.cloudtrail, 0.85, agrees:true}]

  → the Controller raises an internal diagnostic
  → investigation finds the trust policy was edited at 03:40,
    AFTER the assumption and AFTER our 00:34 collection.

  So both sources were right, at different times. The disagreement
  was a collection-timing artifact, not a bug — and the rate of
  these is what tells us whether the closure engine is healthy.
```

### 8.5 Retraction

```
  COVERAGE WINDOW  aws.iam.roles · account 123456789012 · COMPLETE

  6 roles previously known were not observed.
    → 6 ENTITY facts emitted with removed_at set
    → 41 RELATIONSHIP facts referencing them, retracted
    → 2 FINDING facts withdrawn, with the reason recorded

  NO COVERAGE WINDOW  aws.* · account 445566778899 (403 at preflight)

    1,204 entities from that account were also not observed.
    → NOTHING RETRACTED. NOTHING EMITTED.
    → marked stale locally, surfaced in the Controller.

  Same absence of observations. Opposite conclusion. The coverage
  window is the entire difference between "this is gone" and
  "we did not look."
```

---

*Next: [Privacy Gate](11-privacy-gate.md)*

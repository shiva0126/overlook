# E6 — Entity Resolution Engine

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

Entity Resolution decides **which thing this record is about.** One person appears in eight systems under eight identifiers. One server appears in six. If E6 collapses them correctly, the graph is a coherent map. If it does not, the graph is a set of disconnected fragments and every cross-domain attack path — the entire product — silently ceases to exist.

It is also the engine with the most asymmetric failure cost. Under-merging costs a missed finding. Over-merging tells a customer their intern is a domain admin.

---

## 2. Position

```
  INPUT   enriched record with normalized identifiers

  OUTPUT  OBSERVATION — an assertion tied to resolved entity identities,
          carrying plaintext canonical keys, resolution confidence
          and resolution method

  WRITES  entity store, alias table, Resolution Directory
  RAISES  items into the Resolution Review Queue (human adjudication)

  DEPENDS ON
    canonical key priority rules   (shipped config, identical on
                                    every Edge Collector in a deployment)
    the Resolution Directory       (tenant-wide alias → canonical key)
```

---

## 3. Mechanics

### 3.1 Canonical keys

Every entity type has an **ordered list of identity attributes**, most authoritative first. The canonical key derives from the highest-confidence attribute available.

```
  IDENTITY (human)
    1  verified corporate email     email:priya.s@meridian.com
    2  IdP immutable object ID      okta:00u1a2b3c4d5
    3  AD objectGUID                adguid:8f14e45f-...
    4  UPN                          upn:priya.s@corp.meridian.com
    5  SAM account + domain         sam:corp\priyas

  ASSET (host)
    1  cloud instance ID            aws:i-0abc123def456
    2  hardware UUID                hwuuid:4c4c4544-0032-...
    3  FQDN                         fqdn:lt-4471.corp.meridian.com
    4  MAC address set              mac:00:1a:2b:3c:4d:5e
    5  IP + time window             ip:10.4.2.17@2026-08-14  ← weakest

  Descending confidence. An entity resolved by IP alone gets low
  confidence and a short validity window; one resolved by cloud
  instance ID is near-certain.
```

**Confidence propagates.** A path built on weak resolution must be visibly weaker in the UI, or analysts lose trust the first time a merge is wrong.

### 3.2 Three-stage resolution

```
  STAGE 1 — DETERMINISTIC             ~70% of volume, confidence 1.0
    direct match on a canonical key from the priority list.
    Two observations sharing okta:00u1a2b3 are the same identity.
    Cheap, unambiguous, no judgement.

  STAGE 2 — PROBABILISTIC             the ambiguous middle
    weighted attribute scoring across candidates:

      employee ID        0.35   exact
      email local-part   0.30   exact after normalization
      display name       0.15   fuzzy (Jaro-Winkler > 0.92)
      manager            0.10   resolved manager entity matches
      department         0.05   exact
      creation date      0.05   within 7 days

      >= 0.85            merge
      0.65 - 0.85        QUARANTINE for human review
      < 0.65             do not merge

  STAGE 3 — GRAPH REINFORCEMENT        evidence accumulation
    shared neighbours raise the score. Two candidates that are
    MEMBER_OF the same four groups and AUTHENTICATE_TO the same
    six assets are probably the same entity.

  NEGATIVE EVIDENCE — blocks merges regardless of score
    impossible-travel authentication within a short window
    explicitly conflicting owner attributes
    an operator's prior "these are different" decision
```

### 3.3 The Resolution Review Queue

The 0.65–0.85 band goes to a human. This is **an ongoing operational task, not a debug page**.

```
  Every adjudication becomes a STORED RULE, so the same ambiguity
  is never asked twice.

  The queue must show:
    both candidates side by side, with all evidence
    what the evidence FOR and AGAINST is
    THE GRAPH CONSEQUENCE of merging — "+14 edges, 1 new path
    to a crown jewel" — because that tells the operator whether
    the decision matters
```

### 3.4 The Resolution Directory

A tenant-wide mapping of `alternate_identifier → canonical_key`, replicated between Edge Collectors.

```
  WHY IT IS MANDATORY, NOT OPTIONAL

  COL-CLD sees "CORP\priyas" in a CrowdStrike record. It has no
  AD connector and no way to know who that is.

  It queries the Resolution Directory, which COL-DC1 populated
  hours earlier when it read AD, and learns the canonical key is
  email:priya.s@meridian.com.

  WITHOUT IT: two Edge Collectors produce different canonical keys for
  the same person, therefore different tokens, therefore two
  disconnected subgraphs, therefore no hybrid attack path.

  V1 IMPLEMENTATION: elect one Edge Collector as resolution primary;
  others query it with a cached fallback if unreachable.
```

### 3.5 Merge and split must both be reversible

```
  Every merge records: which observations were merged, on what
  evidence, by which stage, at what time, by whom if human.

  FORCE SPLIT must be one click. Over-merge is the unforgivable
  error, so un-merging cannot be a support ticket.
```

---

## 4. Considerations

**Bias toward under-merge, deliberately.** Set thresholds so that ambiguity goes to a human rather than to a guess. A missed finding is invisible and recoverable; a false accusation is visible and destroys credibility.

**Canonical key rules must be identical across Edge Collectors.** They are shipped configuration, versioned, and part of the deployment — not per-node settings. If DC1 prefers email and CLD prefers SAM name, they produce different keys for the same person and the graph fragments invisibly.

**Normalization happens before resolution and determines its success.** `Priya.S@Meridian.com` and `priya.s@meridian.com` must already be identical strings by the time E6 sees them (`04-normalizer-and-enrichment.md §8`).

**Service accounts and machine identities are harder than humans.** No email, no employee ID, often no owner. They resolve on ARN, SPN, or a service-account name plus namespace — and the same *logical* service may legitimately have distinct identities per environment that must NOT be merged.

**Time matters for assets.** An IP-based binding is only valid within a window. Resolving by IP without a time bound will merge two different machines.

**Resolution quality is a product metric, not an internal one.** Percentage deterministic, percentage probabilistic, queue depth, queue age, over-merges reported. Surface it in the Controller.

**Initial load versus steady state.** The first sweep resolves 24,000 entities at once and fills the review queue. Steady state adds a handful per day. Do not tune thresholds on the second regime and then be surprised by the first.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| **Over-merge** | Two people become one. Findings attributed to the wrong person. Catastrophic for trust | Conservative thresholds, negative evidence, human review band, one-click split |
| **Under-merge** | Graph fragments. Paths silently do not exist | Resolution Directory, graph reinforcement, review queue |
| Resolution Directory unreachable | Weak keys cannot resolve on secondary nodes | Cached alias set; mark affected entities "resolution degraded"; do not guess |
| Canonical key config drift between nodes | **Silent graph split.** The worst failure in the system | Config is versioned and part of the deployment; nodes assert their config hash on sync and mismatches alarm |
| Review queue ignored | Ambiguous entities stay unresolved and accumulate | Queue age surfaced in the attention inbox |
| A source changes its identifier format | Mass under-merge overnight | Resolution-rate monitoring against baseline |

---

## 6. Contracts

```
  MUST GUARANTEE
    every entity emits at least one canonical key from the priority list
    identical input produces identical canonical keys on every node
    resolution confidence and method travel with every observation
    every merge is recorded, attributable and reversible
    ambiguity in the middle band is escalated, never guessed
```

---

## 7. Scope

```
  BUILD IN V1
    canonical key derivation for IDENTITY, ASSET, ROLE, AI_AGENT
    stage 1 deterministic matching
    stage 2 probabilistic with the review queue
    Resolution Directory with a designated primary
    force merge / force split, both audited
    resolution quality metrics

  DEFER
    stage 3 graph reinforcement — valuable, but needs a populated
      graph first; add once the graph is stable
    cross-forest AD identity correlation
    ML-assisted matching — the weighted-attribute model is
      explainable, and explainability matters more here than accuracy
```

---

## 8. Example: Meridian, resolving Priya across eight sources

```
  T0   AD (COL-DC1, band 1)
       mail = priya.s@meridian.com
       → priority 1 key. STAGE 1 deterministic.
       → NEW entity E-00417, canonical email:priya.s@meridian.com
       → aliases registered: adguid:8f14e45f-...,
                             sam:corp\priyas,
                             upn:priya.s@corp.meridian.com
       → all three written to the RESOLUTION DIRECTORY
       confidence 1.0

  T1   Entra ID (COL-CLD, band 1)
       userPrincipalName / mail = priya.s@meridian.com
       → priority 1 key. Exact match on E-00417. STAGE 1.
       → merged. Aliases added: entra:00u1a2b3c4d5
       confidence 1.0

  T2   AWS IAM (COL-CLD, band 2)
       tag.email = priya.s@meridian.com
       → priority 1 key. Match. STAGE 1.
       → merged. Alias: arn:aws:iam::123456789012:user/priya.s
       → new observation: E-00417 CAN_ASSUME role/GHADeployRole

  T3   Azure (COL-CLD, band 2)
       priya.s@meridian.com, Contributor on 2 subscriptions
       → STAGE 1. Merged.

  T4   GitHub (COL-CLD, band 3)
       login "psharma-meridian", verified email priya.s@meridian.com
       → the LOGIN is not a canonical key. The VERIFIED EMAIL is.
       → STAGE 1 on the email. Merged.
       → alias: github:psharma-meridian
       ← note: had GitHub not exposed a verified email, this would
         have dropped to stage 2 and probably the review queue.

  T5   CrowdStrike (COL-DC1, band 5)
       user string "CORP\priyas" on host LT-4471
       → NOT a priority-1 key. Priority 5.
       → RESOLUTION DIRECTORY lookup:
           sam:corp\priyas → email:priya.s@meridian.com
       → deterministic match via alias. Merged.
       ← THIS IS THE DIRECTORY EARNING ITS EXISTENCE. Without it,
         CrowdStrike's view of Priya is a separate node and every
         path through her endpoint breaks.

  T6   The agent on LT-4471 (COL-DC1, continuous)
       local user "priyas", host lt-4471
       → E5 attached "corp\priyas" with confidence 0.9
       → directory lookup → E-00417. Merged, confidence 0.94
         (capped by the enrichment confidence, not raised above it)

  T7   Forcepoint DLP (COL-DC1, band 5)
       priya.s@meridian.com, 2 policy events
       → STAGE 1. Merged.

  RESULT
    ONE entity. Eight sources. Resolution confidence 1.0.
    Aliases: adguid, sam, upn, entra ID, AWS ARN, GitHub login,
             EDR user string, DLP subject
```

### 8.1 And the one that went to a human

```
  Same sweep, a different person.

  AD      p.sharma@meridian.com · Engineering · created 2024-03-14
          manager r.kumar · 11 group memberships
  Okta    priya.sharma@meridian.com · Engineering · created 2024-03-14
          manager r.kumar · 12 group memberships (10 shared)

  Two different priority-1 emails. Stage 1 fails.
  Stage 2 scoring:
    employee ID        —      not present in either source
    email local-part   0.00   "p.sharma" ≠ "priya.sharma"
    display name       0.15   "Priya Sharma" vs "Priya Sharma" exact
    manager            0.10   both resolve to the same entity
    department         0.05   both Engineering
    creation date      0.05   same day
                       ─────
                       0.35   → below 0.65, DO NOT MERGE

  But stage 3 reinforcement would have found 10 shared groups.
  Stage 3 is deferred in v1, so this lands in the queue anyway
  as a LOW-confidence candidate flagged for review.

  THE REVIEW QUEUE ITEM

    ARE THESE THE SAME IDENTITY?                       score 0.35

      A  p.sharma@meridian.com        B  priya.sharma@meridian.com
         AD · Engineering                Okta · Engineering
         created 2024-03-14              created 2024-03-14
         manager: r.kumar                manager: r.kumar
         11 groups                       12 groups (10 shared)

      EVIDENCE FOR      same manager, department, creation date,
                        10 shared group memberships
      EVIDENCE AGAINST  different email local-part

      IF MERGED     +14 edges · 1 new path to a crown jewel
      IF SEPARATE   both remain, each with reduced confidence

      [ Same identity ]  [ Different ]  [ Skip ]
      ☑ remember this rule for future matches

  The operator knows Meridian migrated email formats in 2025 and
  some AD records were never updated. Confirms the merge. The rule
  is stored, and the 40 other identities with the same pattern
  resolve automatically on the next sweep.
```

### 8.2 What under-merge would have cost

```
  Had T5 failed — no Resolution Directory, CrowdStrike's
  "CORP\priyas" unresolved — the graph would contain:

    E-00417   Priya, from AD/Entra/AWS/Azure/GitHub/DLP
    E-01882   "corp\priyas", from CrowdStrike and the agent

  And the critical path would break at its first hop:

    [E-01882] laptop LT-4471 → MCP config → GitHub token
        ✕ no edge to E-00417
    [E-00417] → GitHub → OIDC → GHADeployRole → prod-payments-db

  Two half-paths, neither alarming. The finding does not exist.
  Nobody knows it does not exist.

  That is why under-merge is described as a SILENT failure, and
  why the Resolution Directory is not optional.
```

---

*Next: [Permission Closure](06-permission-closure.md)*

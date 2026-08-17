# The Scoring Model

**Series:** [Analytics](00-index.md)

---

## 1. Purpose

Turn a path into a number a human will act on, and be able to defend that number when a cloud engineer disagrees.

Google SCC publishes its formula. That is the standard to meet — **an unpublished score is an unarguable score**, and the first time an engineer says "that's wrong," you need to be able to show the arithmetic rather than assert a model.

---

## 2. Position

```
  INPUTS
    path             the edge sequence from 03
    crown jewel      criticality and data sensitivity      (01)
    start condition  class and exposure weight             (02)
    edge attributes  weights, conditions, PROTECTS, provenance
    temporal         first_seen on every edge → path age
    analyst feedback dispositions, for calibration

  OUTPUTS
    path score 0-100 with every factor exposed
    severity band
    the ranked path list

  CONSUMED BY
    05 choke points · 06 exposure metrics · the UI
```

---

## 3. The formula

Two-step, following Tenable's pattern: aggregate the traversal first, then weight by consequence.

```
  STEP 1 — REACHABILITY   how likely is the traversal?

    reachability = Π (edge.weight × protects_modifier(edge))
                   over every edge in the path

    edge.weight       base per predicate (13 §7), adjusted by
                      condition satisfiability
    protects_modifier 1 − (0.35 × control_effectiveness)
                      for each PROTECTS edge on the target node

  STEP 2 — CONSEQUENCE   what is at the end, and how exposed is
                         the start?

    impact   = crown_jewel.criticality / 100
             × data_sensitivity_factor
             × blast_radius_factor

    exposure = start_condition.weight × start_modifiers

  COMBINE

    base = reachability × impact × exposure × 100

  ADJUST

    confidence_factor = min(edge.confidence) over the path
    age_factor        = 1 + min(0.20, path_age_days / 500)
    synthesis_factor  = 1 − (0.05 × synthesized_edge_count)

    score = base × confidence_factor × age_factor × synthesis_factor

  CLAMP to 0-100.
```

### 3.1 Every factor, and why it is there

| Factor | Range | Source | Rationale |
|---|---|---|---|
| `reachability` | 0–1 | edge weights | multiplicative — a chain is as strong as its product, not its weakest link |
| `protects_modifier` | 0.65–1.0 | `PROTECTS` edges | a control reduces probability, never removes an edge (`13 §8`) |
| `impact` | 0–1.5 | crown jewel | criticality, sensitivity, and what the terminal node itself reaches |
| `exposure` | 0–1 | start condition | how easily an attacker gets to the origin |
| `confidence_factor` | 0.5–1 | `min` over edges | **ours** — a path is only as trustworthy as its weakest inference |
| `age_factor` | 1.0–1.2 | `first_seen` | **ours** — an old path means the org is blind to it |
| `synthesis_factor` | 0.85–1.0 | E8 provenance | **ours** — a synthesized edge is an inference, not an observation |

### 3.2 The three that are ours

```
  CONFIDENCE
    Stellar Cyber has Fidelity — "is this detection a true
    positive?" Ours is different in object: "how certain are we
    this EDGE EXISTS?", derived from multi-source entity resolution
    and closure verification. Nobody else carries resolution
    uncertainty into a path score, because nobody else has our
    resolution problem.

  AGE
    from the bitemporal graph. An 84-day-old path has survived
    every review the organisation performs, which means those
    reviews cannot see it. Capped at +20% so age never dominates
    consequence.

  SYNTHESIS PENALTY
    a path made entirely of observed grants is more certain than
    one relying on three escalation primitives. Small penalty —
    5% per synthesized edge — because these paths are usually the
    important ones. It expresses honesty, not doubt.
```

---

## 4. The auto-elevation rule

Borrowed from Wiz, and it beats the arithmetic.

```
  IF a path terminates at a crown jewel with criticality ≥ 90
     AND confidence ≥ 0.70
  THEN severity is AT LEAST HIGH, regardless of computed score.

  IF additionally the path contains a synthesized escalation edge
     reaching ADMIN privilege
  THEN severity is CRITICAL.

  WHY A RULE AND NOT A WEIGHT
    a long, low-probability path to 4.2M PII records can compute
    to 34 and be buried on page four. The arithmetic is not wrong;
    the ranking is. A rule that says "anything reaching the crown
    jewels surfaces" is more honest than tuning weights until the
    right answers float up.
```

---

## 5. Severity bands

```
  90-100  CRITICAL   act now
  70-89   HIGH       this sprint
  40-69   MEDIUM     backlog
  15-39   LOW        awareness
   0-14   INFO       not shown by default

  Plus the auto-elevation floor above.
```

---

## 6. The calibration loop

None of the seven surveyed products documents doing this well, and it is what stops a prioritisation engine being tuned out within a quarter.

```
  ANALYST DISPOSITIONS
    confirmed              → no change
    not exploitable        → REQUIRES A REASON, and the reason
                             determines what adjusts
    accepted risk          → suppressed with an expiry, score
                             unchanged
    false path             → the edge is wrong. Escalate to
                             engineering, do NOT silently reweight

  REASON → ADJUSTMENT
    "compensating control not modelled"
        → suggest a PROTECTS edge; do not auto-create one
    "condition prevents this in practice"
        → the condition's satisfiability class is wrong; re-classify
    "this identity is decommissioned"
        → a collection freshness problem, not a scoring problem
    "the primitive doesn't apply here"
        → primitive precondition gap; flag for content update

  WHAT ADJUSTS AUTOMATICALLY
    per-tenant edge weight nudges, bounded to ±15% of the global
    default, requiring 5 consistent dispositions before applying

  WHAT NEVER ADJUSTS AUTOMATICALLY
    criticality · confidence · the formula itself
```

**The bounded-nudge design is deliberate.** A tenant whose analysts dismiss everything must not be able to drive all weights to zero and declare itself secure.

---

## 7. Considerations

**Publish the formula.** In the product, not only in this document. A score panel that expands to show each factor and its source is the difference between a defensible finding and an argument.

**Score stability matters as much as accuracy.** If a score swings 30 points because a DLP scan re-ran, the trend line is noise and nobody trusts the number. Damp changes, and surface large swings with their cause rather than applying them silently.

**Multiplicative, not additive.** Every surveyed product does this. Additive scoring lets a path accumulate points from many weak factors and outrank a short, certain, devastating one.

**Do not recompute what others already scored.** VPR and CVSS arrive as properties. They feed `impact` and start-condition weighting. Building our own vulnerability scoring would duplicate Tenable badly.

**Confidence floors before scoring, not after.** A path below the confidence floor is not scored low — it is not emitted at all (`03 §3.2`). Showing a low-confidence path with a low score invites an analyst to act on something we do not believe.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Formula unpublished | Findings unarguable; trust collapses on first dispute | Expose every factor in the UI |
| Score swings on connector change | Trend becomes noise | Damping plus cause attribution |
| Additive scoring | Weak-but-numerous outranks short-and-devastating | Multiplicative, enforced |
| Auto-elevation missing | Critical paths buried by arithmetic | The rule in §4 |
| Unbounded calibration | A tenant tunes itself to zero risk | ±15% bound, 5-disposition minimum |
| Age dominates | Old trivia outranks new critical paths | Capped at +20% |
| Confidence ignored | A path from a bad identity merge scores as certain | `min` over edges, floored before emission |

---

## 9. Example: Meridian

### 9.1 The flagship path, scored

```
  PATH  SEC-github-token-mcp → … → DST-prod-payments   depth 7

  STEP 1 — REACHABILITY
    CONTAINS          0.90
    AUTHENTICATES_TO  0.70   (mechanism: enrollment)
    CAN_WRITE         0.85
    CAN_ASSUME        0.90   synthesized · oidc_subject_too_broad
    CAN_ASSUME        0.85   synthesized · passrole_lambda
    CAN_READ          0.90
    PROTECTS modifier — none on this path
                            ────
    reachability            0.31

  STEP 2 — CONSEQUENCE
    criticality 95                          → 0.95
    data_sensitivity  PII + PCI, 1M-10M     → 1.30
    blast_radius      terminal reaches 2 more crown jewels → 1.15
    impact                                  = 1.42

    start S3 exposed credential             → 0.90
    modifier: never rotated                 × 1.10
    exposure                                = 0.99

  BASE = 0.31 × 1.42 × 0.99 × 100          = 43.6

  ADJUSTMENTS
    confidence_factor  min(0.91)            × 0.91
    age_factor         84 days → 1 + 0.168  × 1.168
    synthesis_factor   2 synthesized        × 0.90
                                            ────────
  SCORE                                       41.7

  AUTO-ELEVATION
    terminates at criticality 95 ✓
    confidence 0.91 ≥ 0.70       ✓          → at least HIGH
    contains a synthesized escalation edge
      reaching ADMIN             ✓          → CRITICAL

  FINAL  score 41.7 · severity CRITICAL
```

**This is the auto-elevation rule earning its place.** The arithmetic says 41.7 — MEDIUM, page three. The path reaches 4.2M customer records through two escalation primitives. The rule is right and the arithmetic is not wrong, it is just answering a different question.

### 9.2 What the analyst sees

```
  ┌───────────────────────────────────────────────────────────┐
  │ CRITICAL · score 41.7 · auto-elevated                     │
  │ 84 days old · confidence 0.91                             │
  │                                                           │
  │ [ why this score ▾ ]                                      │
  │   reachability   0.31   6 edges, no compensating controls │
  │   impact         1.42   crown jewel 95 · PII+PCI · 1M-10M │
  │   exposure       0.99   exposed credential, never rotated │
  │   confidence     0.91   weakest link: MCP credential      │
  │                         presence inferred, not read       │
  │   age            ×1.17  existed 84 days                   │
  │   synthesis      ×0.90  2 edges derived from primitives   │
  │                                                           │
  │   ELEVATED because it reaches a crown jewel through a     │
  │   synthesized escalation to ADMIN.                        │
  └───────────────────────────────────────────────────────────┘
```

### 9.3 A calibration event

```
  An analyst dispositions a different path "not exploitable —
  compensating control not modelled: this subnet is behind an
  IPS that blocks the exploit."

  WHAT HAPPENS
    · the path is suppressed with the reason recorded
    · Overlook SUGGESTS a PROTECTS edge from the IPS to the
      subnet — it does not create one
    · no weight changes yet; 1 disposition of 5

  Four more analysts disposition four more paths through the same
  ROUTES_TO edge with the same reason.

    · threshold reached
    · that edge's weight nudged 0.70 → 0.62 for this tenant only
    · bounded: it cannot fall below 0.595 (−15%)
    · the change is logged, attributed and reversible
    · a banner in the Controller states that a tenant-local weight
      adjustment is active, and why
```

---

*Next: [Choke points](05-choke-points.md)*

# Exposure Metrics

**Series:** [Analytics](00-index.md)

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

The numbers a customer quotes. One goes in a board deck, one drives an engineer's sprint, and confusing the two produces a metric that serves neither.

`../01-system-design.md §36` shows `Exposure Score 72 (-8 this month)` as though it exists. It does not. This document specifies it — and argues it should not be the only number on the page.

---

## 2. Three metric families, not one

Each of the surveyed products picked one. All three are useful and they answer different questions.

```
  PER PATH        risk score 0-100                    (04)
                  → "what do I fix first?"
                  → the engineer's number

  PER NODE        inbound compromise risk
                  → "which assets are dangerous to hold?"
                  → XM Cyber's model

  PER POPULATION  exposure percentage
                  → "how bad is this, overall?"
                  → BloodHound's model, and the most legible
                    metric any of them produce
```

---

## 3. Per-node — inbound compromise risk

XM Cyber computes this from *"the number of preceding breach points and the complexity of attack paths toward the device."*

```
  inbound_risk(node) =
      Σ over paths terminating at or passing through node:
          start_condition_weight × traversal_probability_to_node

  normalised 0-100 across the estate.

  ANSWERS
    which assets are most reachable by an attacker
    where a compromise would be most likely to begin spreading
    which nodes deserve a compensating control

  DIFFERENT FROM CRITICALITY
    criticality is what an asset is WORTH.
    inbound risk is how EXPOSED it is.
    High criticality + low inbound risk = well protected.
    Low criticality + high inbound risk = a stepping stone, and
    the thing most likely to be ignored.
```

---

## 4. Per-population — exposure percentage

The metric worth copying most directly. BloodHound reports *what percentage of principals can reach Tier Zero*, and publishes that in an average AD domain it exceeds 70%.

Nobody applies the shape beyond identity. We have the node types to.

```
  exposure_pct(population, destination_set) =
      | members of population with ≥1 path to destination_set |
      ─────────────────────────────────────────────────────────
      | members of population |

  POPULATIONS WORTH REPORTING
    human identities          → the BloodHound metric
    non-human identities      → usually far worse, never measured
    AI agents                 → ours alone
    CI/CD pipelines           → supply-chain exposure
    internet-facing assets    → perimeter exposure
    endpoints                 → workstation exposure

  Report the PERCENTAGE and the RAW COUNT. BloodHound added the
  count after shipping only the percentage, because "8% of 40,000"
  and "8% of 50" demand different responses.
```

**Why this metric lands where a score does not:** *"63% of your service accounts can reach a crown jewel"* needs no explanation, cannot be argued with arithmetic, and immediately implies the shape of the fix. *"Exposure score 72"* needs a paragraph.

---

## 5. The organisation exposure score

The board number. It has one hard requirement that outranks accuracy: **it must not be gameable.**

```
  exposure_score =
      Σ over crown jewels:
          criticality_weight × best_path_score(cj)
      ─────────────────────────────────────────────
      Σ over crown jewels: criticality_weight

    × coverage_adjustment
    × 100 / 100

  best_path_score(cj) = the highest-scoring path reaching it, or 0

  Reported 0-100, higher is worse.
```

### 5.1 Coverage adjustment — the anti-gaming term

```
  THE FAILURE THIS PREVENTS

    disconnect the DLP connector
      → datastores lose classification
      → criticality falls
      → fewer crown jewels
      → fewer paths
      → EXPOSURE SCORE IMPROVES

    A score that rewards blindness is worse than no score.

  coverage_adjustment = 1 + (1 − coverage_fraction) × 0.5

    coverage_fraction is the weighted completeness across domains
    (identity, cloud, network, data, runtime, code, AI) from the
    Controller's coverage view.

  → losing a connector RAISES the exposure score, because unknown
    exposure is assumed adverse.
  → the score cannot be improved by collecting less. It can only
    be improved by fixing things or by collecting MORE.
```

### 5.2 Stability

```
  · damp movements: no more than ±10 points per day without an
    explicit cause attached
  · every movement carries an attribution — "−14 from the OIDC
    trust fix on 2026-08-17", "+6 from GCP connector added,
    coverage improved and 3 new paths found"
  · a score that rises because coverage improved must SAY SO,
    or the customer will read a successful onboarding as a
    regression
```

---

## 6. The dashboard

Microsoft's headline is three numbers rather than one, and that is the better pattern.

```
  ┌──────────────────────────────────────────────────────────┐
  │  6              3                47                      │
  │  CRITICAL       CHOKE POINTS     CROWN JEWELS            │
  │  PATHS          fixing these     3 currently reachable   │
  │  2 new          removes 4,100                            │
  │                                                          │
  │  EXPOSURE 58   ▼14 this month · coverage-adjusted        │
  │                                                          │
  │  EXPOSURE BY POPULATION                                  │
  │    human identities         12%   (1,440 of 12,000)      │
  │    non-human identities     63%   (1,323 of 2,100)  ⚠    │
  │    AI agents                17%   (8 of 47)              │
  │    CI/CD pipelines          31%   (42 of 136)            │
  │                                                          │
  │  COVERAGE                                                │
  │    identity 97 · cloud 94 · data 71 · network 88         │
  │    AI 82 · runtime 91 · code 64  ⚠ 2 GitHub orgs missing │
  └──────────────────────────────────────────────────────────┘
```

**Choke points sit beside critical paths deliberately.** The first tells you how bad it is; the second tells you how much work fixing it is. Those two numbers together are the whole executive conversation.

---

## 7. Considerations

**Never report a single number alone.** An exposure score without coverage beside it is an assertion about a graph whose completeness is unstated.

**Percentages need denominators.** 63% of non-human identities means nothing without knowing there are 2,100 of them.

**Trend beats level.** A score of 58 is meaningless in isolation; 72 → 58 over a month, with attribution, is a report. Benchmarks against peers (`09 §3`, computed on tokens) come third.

**Do not benchmark without consent.** Cross-customer comparison is opt-in with published methodology, or it is a liability the first time a customer learns they were compared.

**The engineer's number and the board's number must reconcile.** If choke point #1 eliminates 1,240 paths, the exposure score must visibly move when it is fixed. A score that does not respond to work being done stops being watched.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Score gameable by disconnecting a source | Rewards blindness | Coverage adjustment, §5.1 |
| Score without coverage shown | Overstates certainty | Always paired |
| Percentages without counts | Misread by an order of magnitude | Both, always |
| Score swings without attribution | Read as noise, then ignored | Damping plus cause |
| Only a single number | No connection to action | Three-number headline |
| Coverage improvement reads as regression | Onboarding looks like failure | Explicit attribution on the movement |

---

## 9. Example: Meridian

### 9.1 The three families, computed

```
  PER PATH        31,400 paths scored
                  6 critical · 41 high · 380 medium
                  top: 41.7, auto-elevated CRITICAL

  PER NODE        inbound compromise risk, top 5
                  ROL-ghadeploy         94
                  IDN-svc-devops-ai     91
                  AST-lt-4471           78   ← a laptop, and a
                                              stepping stone nobody
                                              would have flagged
                  NET-vpn-pool          71
                  REP-payments-api      68

  PER POPULATION
                  human identities        12%  (1,440 / 12,000)
                  non-human identities    63%  (1,323 / 2,100)
                  AI agents               17%  (8 / 47)
                  CI/CD pipelines         31%  (42 / 136)
                  internet-facing assets  44%  (61 / 138)
```

**The number Meridian's CISO reacted to was 63%.** Not the exposure score, not the critical path count. *"Two thirds of our service accounts can reach something we designated as a crown jewel"* is a sentence that needs no explanation and cannot be argued with.

### 9.2 The organisation score, with its arithmetic

```
  47 crown jewels · weighted by criticality

  Σ (criticality_weight × best_path_score)   = 3,982
  Σ criticality_weight                       = 4,465
  ─────────────────────────────────────────────────
  raw                                        = 0.892 → 89.2? no:

  best_path_score is 0 for the 44 crown jewels with NO path.
  Only 3 are currently reachable.

  Σ (criticality_weight × best_path_score)
    prod-payments-db   95 × 41.7  = 3,962
    prod-identity-store 92 ×  8.1 =   745
    prod-backup-vault  90 ×  4.2  =   378
                                    ──────
                                     5,085
  Σ criticality_weight over ALL 47 crown jewels = 4,268

  → normalised: 5,085 / 4,268 / 100 × 100    = 11.9

  coverage_adjustment
    weighted coverage across domains = 0.844
    1 + (1 − 0.844) × 0.5            = 1.078

  EXPOSURE SCORE = 11.9 × 1.078 × ...        → 58 after scaling

  ▼14 this month
    −18  OIDC trust tightened (choke point #1) on 2026-08-17
    + 4  GCP connector added — coverage rose, 3 new paths found
```

That last line is the honesty requirement working. Onboarding GCP *raised* the score by 4 because it revealed real paths that already existed. Without the attribution, Meridian would have read a successful expansion as a regression.

### 9.3 What gaming would have looked like

```
  Suppose Meridian disconnected Forcepoint DLP.

    4,100 datastores lose classification
    crown jewels fall 47 → 16
    reachable crown jewels fall 3 → 1
    raw exposure would fall 58 → 31

  BUT
    data coverage falls 71% → 12%
    weighted coverage falls 0.844 → 0.702
    coverage_adjustment rises 1.078 → 1.149

    AND the coverage panel shows data 12% in red
    AND the score movement is attributed:
      "+? from data coverage loss — 31 crown jewels no longer
       classified. Exposure is UNKNOWN, not reduced."

  → the score does not improve, and the reason is on the screen.
```

---

*Next: [Blast radius and change](07-blast-radius-and-change.md)*

# The Attack Path Engine

**Series:** [Analytics](00-index.md)

---

## 1. Purpose

Compute every plausible route from a start condition (`02`) to a crown jewel (`01`), rank them, collapse them, and report only what changed.

The engineering problem is computing 500,000 paths. The **product** problem is presenting fewer than 50 that a human will act on, and it is the harder of the two.

---

## 2. Position

```
  INPUTS
    the graph               entities, edges, materialized closures
    crown jewels            traversal destinations           (01)
    start conditions        traversal terminators            (02)
    edge weights            per-predicate base + adjustments (04)
    change events           for incremental recompute

  OUTPUTS
    ranked paths with full provenance
    the path set, for choke-point analysis                   (05)
    a diff against the previous run — NEW paths notify

  RUNS ON
    Mode 1: the appliance, against the local graph
    Mode 2: the console, against the merged token graph
    ⚠ the engine is IDENTICAL. Tokens traverse exactly like
      canonical keys — the algorithm never reads a value.
```

---

## 3. Mechanics

### 3.1 Reverse BFS, not forward

```
  FORWARD from every start condition
    1,847 starts × branching factor ^ 8 = intractable
    and most branches never reach anything worth reporting

  REVERSE from every crown jewel
    47 destinations × incoming edges, depth-bounded
    and every branch is by construction heading somewhere that
    matters

  → reverse BFS, seeded from crown jewels, terminating at start
    conditions. This is also why 01 comes first: without
    destinations there is nothing to seed.
```

### 3.2 The traversal

```
  for each crown_jewel cj:
    frontier ← { cj }
    visited  ← { cj: (weight 1.0, confidence 1.0, depth 0) }

    for depth in 1..MAX_DEPTH:
      next ← ∅
      for node in frontier:
        for edge in incoming_traversable_edges(node):
          w ← visited[node].weight × edge.weight
          c ← min(visited[node].confidence, edge.confidence)

          PRUNE if w < WEIGHT_FLOOR            (default 0.02)
          PRUNE if c < CONFIDENCE_FLOOR        (default 0.50)
          PRUNE if depth > MAX_DEPTH           (default 8)
          PRUNE if edge.from already on this path   (cycle)

          record predecessor
          next ← next ∪ { edge.from }
      frontier ← next

    emit every path whose terminal node is a START CONDITION
```

### 3.3 What "traversable" means

Only the eighteen predicates marked traversable in `../13-contracts.md §7`. `OWNS`, `PROTECTS`, `HAS_VULNERABILITY` and `STORES` are never traversed — a control does not create reachability, it reduces the weight of adjacent edges.

```
  incoming_traversable_edges(node) uses the REVERSE partial index:
    btree (to_token, predicate) WHERE removed_at IS NULL

  This index exists specifically for this query. Without it,
  traversal degrades as tombstones accumulate — slowly, then
  suddenly.
```

### 3.4 Five explosion controls, applied together

```
  1  DEPTH LIMIT — 8
     beyond eight hops a path stops being an actionable narrative
     and becomes a curiosity. Tune per deployment; never remove.

  2  WEIGHT BUDGET — cumulative floor 0.02
     eight low-probability edges multiply to nothing. Stop early.

  3  CONFIDENCE FLOOR — 0.50
     a path is only as trustworthy as its weakest inference.
     Below half, do not show it at all.

  4  EQUIVALENCE CLASSES
     400 EC2 instances in one ASG with an identical role are ONE
     path with multiplicity 400, not 400 paths.
     380 identical Terraform-deployed roles across accounts
     collapse the same way.
     → typically the largest single reduction after choke points

  5  CROWN-JEWEL SCOPING
     only criticality ≥ 90 seeds traversal. Everything else is
     blast radius, computed on demand (07).
```

### 3.5 Path deduplication

```
  Two paths are the SAME PATH if they share:
    the same ordered sequence of (node_type, predicate) pairs
    AND the same start class
    AND the same crown jewel

  Differing only in which of 400 identical instances they traverse
  → one path, multiplicity 400.

  Differing in mechanism (sts_assume_role vs oidc_federation)
  → GENUINELY DIFFERENT PATHS. mechanism is significant
    (../13-contracts.md §9), and collapsing them hides exactly the
    kind of exposure the OIDC finding depends on.
```

### 3.6 Incremental recomputation

The claim that distinguishes this from a six-hourly batch (SCC runs every ~6 hours, at least daily).

```
  edge added / removed / weight changed
    → reverse index lookup: which paths traverse this edge?
    → recompute ONLY those paths
    → typical: 40 paths, ~200 ms
    → versus full recompute over 2.9M edges: ~25 minutes

  A full recompute still runs nightly, because:
    · incremental drift accumulates
    · crown-jewel or start-condition changes invalidate broadly
    · it is the reconciliation check that incremental is correct
```

**The reverse index is the whole trick.** `path_edges (edge_id → path_id)` maintained as paths are emitted. Without it, "which paths use this edge?" is a scan and incremental recompute is not possible.

### 3.7 Diff, and what notifies

```
  every run produces:  NEW · PERSISTING · RESOLVED

  ONLY NEW notifies.

  A path that has existed for 84 days is not news. It is important —
  and it scores higher for its age (04) — but it does not page
  anyone at 03:00. The change feed is where age becomes a signal;
  the notification channel is for what changed.
```

---

## 4. Considerations

**Path counts are not a metric.** *"You have 47,000 attack paths"* is noise theatre. The number a customer acts on is the choke-point count (`05`). Design the UI so choke points are the primary object and paths are the drill-down — the same conclusion Microsoft and XM both reached.

**Determinism matters more than it looks.** The same graph must produce the same paths in the same order. Non-determinism from map iteration or parallel traversal makes the diff meaningless and every run looks like change. Sort deterministically at every branch.

**Traversal on tokens must be identical to traversal on canonical keys.** In Mode 2 the console traverses tokens. If the algorithm ever needs to read a value — to parse an ARN, to infer a type — it breaks the privacy model. Everything the traversal needs must already be in edge attributes and node types.

**Conditions are already resolved.** By the time the path engine runs, `satisfiability` has been classified by E7 and folded into edge weight (`../engines/06 §3.4`). The path engine never re-evaluates a policy condition.

**A path through a stale subgraph must be visibly stale.** If a connector failed and its edges are marked stale, any path traversing them renders with a staleness badge **on the path**, not on a separate health page.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Missing reverse index | Incremental recompute impossible; every change is a full run | Index maintained as paths are emitted |
| Path explosion | Memory exhaustion, run never completes | Five controls in §3.4, applied together, not selectively |
| Non-deterministic ordering | Diff meaningless; everything looks new every run | Deterministic sort at every branch |
| Over-aggressive dedup | Distinct mechanisms collapse, OIDC exposure hidden | Mechanism is part of path identity |
| Cycle not detected | Infinite traversal | Path-local visited set, not global |
| Stale edges traversed silently | A path is reported that may not exist | Staleness propagates to the path and is displayed |
| Crown jewels change | Every path invalidated | Full recompute on crown-jewel set change, not incremental |

---

## 6. Example: Meridian

```
  INPUT
    2.9M live edges · 47 crown jewels · 1,847 start conditions
    (312 human classes after equivalence clustering)

  FULL RUN, nightly, EDGE-DC1 profile L
    reverse BFS from 47 seeds
    raw paths found                         412,000
    after depth limit (8)                   188,000
    after weight floor (0.02)                74,000
    after confidence floor (0.50)            61,200
    after equivalence collapse               31,400
    ─────────────────────────────────────────────────
    paths emitted                            31,400
    wall clock                               23 minutes

    after choke-point collapse (05)      6 critical · 41 high
    SHOWN TO A HUMAN                                47
```

### 6.1 One path, with its pruning history

```
  SEED  DST-prod-payments (criticality 95)

  depth 1   ← CAN_READ from ROL-ec2app
            w 1.00 × 0.90 = 0.90 · c 0.99
  depth 2   ← CAN_ASSUME from ROL-ghadeploy   [SYNTHESIZED, E8]
            w 0.90 × 0.85 = 0.77 · c 0.92
            primitive aws.privesc.passrole_lambda v3
  depth 3   ← CAN_ASSUME from PIP-gha-any-meridian-repo [SYNTHESIZED]
            w 0.77 × 0.90 = 0.69 · c 0.92
            primitive aws.oidc.subject_condition_too_broad v2
  depth 4   ← CAN_WRITE from REP-payments-api
            w 0.69 × 0.85 = 0.59 · c 0.92
  depth 5   ← CAN_WRITE from IDN-priya
            w 0.59 × 0.85 = 0.50 · c 0.92
  depth 6   ← AUTHENTICATES_TO from AST-lt-4471
            w 0.50 × 0.70 = 0.35 · c 0.91
            mechanism: enrollment (Scalefusion, authoritative)
  depth 7   ← CONTAINS from SEC-github-token-mcp
            w 0.35 × 0.90 = 0.31 · c 0.91

  TERMINAL  SEC-github-token-mcp is an S3 start condition
            (exposed credential, in an MCP config, weight 0.90)

  PATH   weight 0.31 · confidence 0.91 · depth 7 · age 84 days
```

### 6.2 Branches that were pruned, and why

```
  ✕ 41,000 paths pruned at the WEIGHT FLOOR
      mostly chains of CONNECTS_TO (0.60 each) — four network hops
      multiply to 0.13 and eight to 0.017

  ✕ 12,800 pruned at the CONFIDENCE FLOOR
      almost all traversing identities resolved probabilistically
      during the AD/Entra merge, or edges from the account whose
      credential expired and whose subgraph is stale

  ✕ 29,800 collapsed by EQUIVALENCE
      400 EC2 instances in one ASG → 1 path × 400
      380 identical Terraform roles → 1 class × 380
      312 human start classes standing for 12,000 identities

  ✕ 224,000 pruned at DEPTH
      real routes, but nine or more hops. Reported as a count in
      the coverage view — "224,000 paths exist beyond depth 8" —
      never as findings. Silent truncation would be dishonest;
      showing them would be useless.
```

### 6.3 The incremental case

```
  14:22  a Meridian engineer edits an IAM policy on svc-reporting

  14:22  E7 recomputes closure for 1 principal + 11 who can assume it
         → 3 changed edges emitted

  14:22  path engine: reverse index lookup on those 3 edges
         → 47 paths affected
         → recomputed in 190 ms
         → 1 NEW path to prod-payments-db, score 88

  14:23  change feed: significance CRITICAL — a new path to a
         crown jewel. Notification fires.

  Nightly full run at 02:00 reconciles and confirms.
```

---

*Next: [Scoring model](04-scoring-model.md)*

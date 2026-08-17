# Choke Points and Break Attack Path

**Series:** [Analytics](00-index.md)

---

## 1. Purpose

Find the edges where many paths converge, and prove what removing one would do — **before** anyone touches production.

This is the product's most concentrated value. Not *"you have 31,400 attack paths"* but *"one line in one policy, forty minutes of work, 26% of your critical exposure gone."*

XM Cyber and Microsoft both put choke points on the dashboard. Neither documents simulating the **operational impact of the fix**, which is the part that decides whether anyone actually does it.

---

## 2. Position

```
  INPUTS
    the path set from 03, with scores from 04
    the graph, for simulation overlay
    dependency data — what else uses the edge being cut

  OUTPUTS
    ranked choke points, each with paths eliminated and
    crown jewels protected
    a simulated post-fix graph delta
    an operational impact preview
    an applyable remediation artifact

  CONSUMED BY
    the UI (the primary object, not the drill-down)
    06 exposure metrics
    ../05 controller response workflow
```

---

## 3. Mechanics

### 3.1 Detection is edge frequency over the path set

```
  for each edge e appearing in any path:
    paths_through[e]        ← count
    critical_through[e]     ← count where severity = CRITICAL
    crown_jewels_via[e]     ← distinct crown jewels downstream
    score_mass[e]           ← Σ score of paths through e

  CHOKE POINT SCORE
    = crown_jewels_via × w1
    + critical_through × w2
    + log(paths_through) × w3
    + score_mass        × w4

  log() on path count is deliberate. An edge in 40,000 paths is
  not 40× more important than one in 1,000 — most of those paths
  are equivalence-class expansions of the same structural problem.
```

### 3.2 Rank by what is eliminated, not by frequency

```
  A high-frequency edge deep in the graph may carry 40,000 paths
  and protect nothing, because every path through it also has an
  alternative route.

  What matters is ELIMINATION:

    for each candidate choke point:
      simulate its removal
      recount paths that survive
      eliminated = before − after
      residual   = paths to the same crown jewels via other routes

  An edge whose removal eliminates 1,240 paths but leaves 2 residual
  paths to the same crown jewel is a GOOD fix that is not a
  COMPLETE fix, and the UI must say both.
```

### 3.3 Simulation on an overlay

```
  1  build an in-memory overlay of the graph
  2  apply the proposed change — remove an edge, tighten a
     condition, scope a permission
  3  recompute ONLY the affected paths, via the reverse index (03 §3.6)
  4  diff: paths eliminated, crown jewels protected, score delta,
     residual routes
  5  discard the overlay

  Nothing is written. Nothing is applied. This is the capability
  that requires a complete permission-resolved graph, which is
  why it is hard for anyone without one.
```

### 3.4 Operational impact — the part that gets it done

A security recommendation that breaks production is not adopted. Blast radius of the **fix**, not of the attack:

```
  for the edge being cut:
    who else uses it?
      → other principals with the same grant
      → pipelines, jobs and workloads depending on it
      → recent usage from audit data (the USED state, 02 §2)

  CLASSIFY
    breaks              actively used in the last 90 days
    probably safe       granted, never used
    needs review        used 90-365 days ago — quarterly jobs
```

**The "needs review" band is what makes this trustworthy.** A 90-day window silently misses quarterly processes; surfacing them is the difference between a recommendation someone applies and one they are afraid of.

### 3.5 Output artifacts

```
  [ Copy policy diff ]     the exact JSON/YAML change
  [ Terraform diff ]       where IaC is the source of truth
  [ Open ticket ]          via the ServiceNow/Jira response manifest
  [ Create PR ]            where the policy lives in a repo
  [ Apply via connector ]  optional, off by default, and gated by
                           the full response chain (../01 §30)
```

Most customers will never use the last one. The value is knowing *what* to change, which is a graph problem; doing it is a commodity.

---

## 4. Considerations

**Choke points are the headline, paths are the drill-down.** Both XM and Microsoft reached this independently, and Cyentia's research on XM's data quantifies why: **20% of choke points expose 10% or more of critical assets.** A UI that leads with path counts is showing the wrong object.

**A choke point can be a node, not only an edge.** Removing an over-privileged service account eliminates every path through it. Node choke points are usually more disruptive to fix and more effective — model both, and be explicit which one a recommendation is.

**Residual paths must be reported.** Eliminating 1,240 paths while leaving 2 routes to the same crown jewel is progress, not resolution. Reporting only the eliminated count is the kind of overstatement that gets discovered in a post-incident review.

**Do not recommend cutting an edge the customer cannot change.** A managed-service-linked role, a provider-imposed trust, a control-plane relationship — these appear as choke points and are not actionable. Flag them as structural rather than ranking them as recommendations.

**Simulation must be cheap enough to be interactive.** An operator will try five variations. If each takes a minute, they will try one. The reverse index makes this ~200 ms; without it, this feature does not exist.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Ranking by raw frequency | Recommends a high-traffic edge that protects nothing | Rank by simulated elimination |
| No residual reporting | Customer believes an exposure is closed when it is not | Residual count is mandatory in the output |
| No operational impact | Recommendation is feared and not applied | Blast radius of the fix, with the 90–365 day review band |
| Simulation writes to the graph | Corruption from a what-if | Overlay only, discarded |
| Unactionable choke points ranked | Trust erodes — "it keeps telling us to do impossible things" | Classify structural vs actionable |
| Slow simulation | Nobody explores alternatives | Reverse index; interactive budget |

---

## 6. Example: Meridian

```
  INPUT   31,400 paths · 6 critical · 41 high

  CHOKE POINT ANALYSIS                        candidates 2,104
  after removing structural / unactionable    →       1,847
  after elimination simulation, ranked        → top 12 shown
```

### 6.1 The top recommendation

```
  ┌──────────────────────────────────────────────────────────────┐
  │ CHOKE POINT #1                                               │
  │                                                              │
  │ EDGE   PIP-gha-any-meridian-repo ─CAN_ASSUME→ ROL-ghadeploy  │
  │        synthesized · aws.oidc.subject_condition_too_broad v2 │
  │                                                              │
  │ APPEARS IN     1,240 paths                                   │
  │   critical         3                                         │
  │   high            17                                         │
  │ CROWN JEWELS VIA   9                                         │
  │                                                              │
  │ PROPOSED CHANGE                                              │
  │   tighten the OIDC trust subject condition                   │
  │     from  "repo:meridian/*"                                  │
  │     to    "repo:meridian/payments-api:ref:refs/heads/main"    │
  │                                                              │
  │ SIMULATED EFFECT                                             │
  │   paths eliminated          1,240                            │
  │   crown jewels protected        9                            │
  │   exposure score           72 → 58   (−14)                   │
  │   RESIDUAL: 2 paths to prod-payments-db via a different       │
  │             route (ROL-ec2app assumed directly by             │
  │             svc-batch-processor) — SEE CHOKE POINT #3         │
  │                                                              │
  │ OPERATIONAL IMPACT                                           │
  │   breaks           2 workflows                               │
  │     gha-deploy-staging      uses this trust, ref refs/heads/* │
  │     gha-nightly-build       uses this trust, any ref          │
  │   probably safe    1 workflow — granted, never used in 90d    │
  │   needs review     1 workflow                                │
  │     gha-quarterly-audit  last used 118 days ago               │
  │     ⚠ likely a quarterly job. Confirm before applying.        │
  │                                                              │
  │ ESTIMATED EFFORT   40 minutes                                │
  │                                                              │
  │ [ Copy trust policy diff ]  [ Create PR ]  [ Open Jira ]     │
  └──────────────────────────────────────────────────────────────┘
```

### 6.2 What ranking by frequency would have chosen instead

```
  EDGE   NET-corp-lan ─ROUTES_TO→ NET-server-vlan
  appears in  8,900 paths   ← more than twice #1

  SIMULATED ELIMINATION
    paths eliminated:  140
    residual:        8,760

  Because almost every path through that segment has an
  alternative route through the NSX distributed firewall.

  Cutting it would be enormously disruptive — it is the corporate
  LAN reaching the server VLAN — and would eliminate 1.6% of the
  paths it carries.

  → correctly ranked #47, and flagged "high disruption, low
    elimination."
```

### 6.3 A node choke point

```
  ┌──────────────────────────────────────────────────────────────┐
  │ CHOKE POINT #2 — NODE                                        │
  │                                                              │
  │ NODE   IDN-svc-devops-ai   (also a crown jewel, criticality 92)│
  │                                                              │
  │ APPEARS IN     4,100 paths                                   │
  │ CROWN JEWELS VIA  3                                          │
  │                                                              │
  │ PROPOSED CHANGE                                              │
  │   replace AdministratorAccess with a scoped policy derived    │
  │   from 90-day usage: 12 actions across 3 services             │
  │                                                              │
  │ SIMULATED EFFECT                                             │
  │   paths eliminated      3,890                                │
  │   escalation edges out     4 → 0                             │
  │   AI PRIVILEGE GAP finding  RESOLVED                          │
  │   exposure score       58 → 41                               │
  │                                                              │
  │ OPERATIONAL IMPACT                                           │
  │   breaks         0 — every used action is in the new policy   │
  │   needs review   3 actions used 91–140 days ago               │
  │     ec2:CreateSnapshot · rds:RestoreDBInstance ·              │
  │     iam:UpdateAssumeRolePolicy                                │
  │     ⚠ these look like disaster-recovery operations. Include   │
  │       them, or accept that DR will fail.                      │
  │                                                              │
  │ [ Copy generated policy ]  [ Terraform diff ]                │
  └──────────────────────────────────────────────────────────────┘
```

The "needs review" block here is the whole point. A naive rightsizing recommendation built on a 90-day window would have removed three DR operations, and the failure would have surfaced during an actual disaster.

---

*Next: [Exposure metrics](06-exposure-metrics.md)*

# Blast Radius and Change Intelligence

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

Two capabilities that share one property: both are byproducts of design decisions already made, and both are nearly free.

**Blast radius** is the path engine inverted — forward traversal from a node instead of backward from a crown jewel. **Change intelligence** is the bitemporal graph read along its time axis instead of its structure axis.

Neither needs new machinery. Both are undocumented.

---

# PART A — BLAST RADIUS

## 2. What it is

```
  ATTACK PATH      what can reach THIS?         reverse BFS
  BLAST RADIUS     what can THIS reach?         forward BFS

  Same traversal, same weights, same pruning. Opposite direction.
```

## 3. Four uses, one computation

```
  1  INCIDENT RESPONSE
     "this host is compromised — what does the attacker now have?"
     seed: the compromised node, as an S6 start condition (02 §3)

  2  AI AGENT ASSESSMENT
     "if this agent is manipulated, what can it do?"
     seed: the agent's runs_as identity
     → this is the AI blast radius, and it is what makes an
       agent's over-privilege legible

  3  CHANGE PREVIEW
     "if we grant this role, what does it open up?"
     seed: the principal, on an overlay with the proposed grant

  4  RESPONSE IMPACT
     "if we quarantine this host, what breaks?"
     seed: the host, traversing DEPENDENCY rather than attack edges
     → feeds the impact preview in the response chain (../01 §30.3)
```

That one traversal serves all four is a good sign the abstraction is right.

## 4. Presentation matters more than computation

```
  AGENT COMPROMISED: DevOps-Agent

  CAN REACH        14 systems       (3 production)
  CAN READ          3 databases     (1 PII, 1M-10M records)
  CAN MODIFY        2 applications  (both customer-facing)
  CAN EXECUTE       cloud functions, EC2 via SSM
  CAN SEND          external email via SES
  CAN ASSUME        4 roles         (1 with Administrator)

  MOST SEVERE CONSEQUENCE
    read and exfiltrate 4.2M customer records via
    svc-devops-ai → DevOpsAdmin → prod-payments-db → ses:SendEmail
```

**Grouped by capability, not by node.** A list of 340 reachable resources is unreadable; six capability lines with counts is a briefing. The "most severe consequence" line is the single sentence that goes into an incident report.

## 5. Considerations

**Blast radius is unbounded without limits.** Forward traversal from a privileged identity reaches most of the estate. Apply the same depth, weight and confidence pruning as `03`, and report what was pruned rather than silently truncating.

**Depth means something different forwards.** Reverse BFS depth 8 is a reasonable attack narrative. Forward depth 8 from an admin role is the whole environment. Default forward depth to **4**, with expansion on request.

**Dependency edges are not attack edges.** Response impact (use 4) traverses "what depends on this" — `RUNS_ON`, `CONTAINS`, pipeline usage — not "what can this reach." Using attack edges for a quarantine preview would produce nonsense.

**Blast radius is computed on demand, never precomputed.** There are millions of possible seeds and no way to know which one an analyst will ask about.

---

# PART B — CHANGE INTELLIGENCE

## 6. Why change beats state

```
  A posture report is read once and filed.
  A change is acted on.

    "a new admin grant appeared 3 hours ago"
      → the highest-signal event in identity security
    "this path became reachable when a firewall rule changed at 14:22"
      → direct causation, and the person who made the change
    "this agent gained a new tool yesterday"
      → AI-specific, and invisible to everything else
    "this MCP server appeared on 14 laptops this week"
      → shadow adoption in progress
```

## 7. The change feed

A byproduct of the graph writer, not a separate system (`../engines/09 §3.6`).

```
  every graph write → diff → ChangeEvent
    { type, subject, predicate, object, before, after,
      detected_at, source, collector_id, significance }

  SIGNIFICANCE IS COMPUTED, NOT RAW
    created a new path to a crown jewel        → CRITICAL
    raised an existing path score by >20       → HIGH
    granted privilege where none existed       → HIGH
    removed a PROTECTS edge                    → HIGH
    changed a condition's satisfiability class → MEDIUM
    routine churn (autoscaling, ephemeral pods)→ SUPPRESS
```

## 8. Suppression by equivalence class, not by rate

```
  A Kubernetes cluster generates thousands of node and edge changes
  an hour from normal autoscaling. Rate-limiting the feed would
  drop real changes alongside noise.

  Instead: learn that pods matching a workload template with an
  identical service account are ONE LOGICAL ENTITY, and report the
  TEMPLATE's changes rather than each pod's.

  1,847 pod events → 1 change: "workload template payments-api
  scaled 4 → 11". And if the template's service account changes,
  that is ONE HIGH-significance event, not 11.
```

## 9. The temporal surface

What bitemporality buys, beyond the change feed:

```
  PATH AGE            "this path has existed 84 days"
                      → the age_factor in scoring (04 §3.1)
                      → and the single most useful sort order in
                        the UI, because old paths are the ones
                        every review has already failed to see

  POINT-IN-TIME       "show me the graph last Tuesday"
                      → investigation, audit, and answering
                        "was this true when the incident happened?"

  FIX VERIFICATION    "did our change actually break the path?"
                      → closes the loop on response. Without it,
                        a customer applies a remediation and has
                        no evidence it worked.

  PRIVILEGE VELOCITY  rate of permission accumulation per identity
                      → identities only ever gain permissions;
                        a steadily climbing curve on a service
                        account is a finding in itself
```

## 10. Considerations

**Absence of observation is not observation of absence.** Stated in E12, E13, the contracts, and again here. A change feed that reports "edge removed" when a connector merely failed is worse than no feed. Every removal event must cite the coverage window that authorises it.

**Attribution requires the audit connectors.** *"This changed at 14:22"* is useful; *"changed at 14:22 by priya.s via Terraform run 4471"* is actionable. That comes from CloudTrail, Activity Log, Cloud Audit Logs and Terraform run history — which is why those collectors exist despite producing few edges.

**The feed must be readable a week later.** Not a stream to watch, a record to review. Group by significance, then by subject, and default to the last 7 days.

**Do not alert on everything HIGH.** Only CRITICAL notifies. HIGH appears in the feed and the attention inbox. The distinction between "wake someone" and "someone should see this" has to exist or both become noise.

---

## 11. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Removal reported without coverage | False "resolved" events; trust destroyed | Every removal cites its coverage window |
| No equivalence suppression | Feed unreadable within a day | Template-level grouping |
| Blast radius unbounded | Traversal never completes | Forward depth 4, same pruning as `03` |
| Attack edges used for response impact | Nonsense quarantine previews | Dependency edges for use 4 |
| Everything alerts | Feed ignored | Only CRITICAL notifies |
| No change attribution | "Something changed" with no owner | Audit-log collectors are load-bearing for this |

---

## 12. Example: Meridian

### 12.1 Blast radius — the AI agent

```
  SEED  IDN-svc-devops-ai (the agent's runs_as identity)
  forward BFS, depth 4, weight floor 0.02

  CAN REACH        340 resources across 41 accounts
                     of which 61 in production
  CAN READ           3 databases
                     prod-payments-db   PII + PCI · 1M-10M   crit 95
                     prod-identity-store                     crit 92
                     analytics-warehouse                     crit 61
  CAN MODIFY         2 customer-facing applications
  CAN EXECUTE        EC2 via ssm:SendCommand on 184 instances
  CAN ASSUME         4 roles, 1 with AdministratorAccess
  CAN SEND           external email via ses:SendEmail

  MOST SEVERE CONSEQUENCE
    read and exfiltrate 4.2M customer records via
    svc-devops-ai → DevOpsAdmin → prod-payments-db → ses:SendEmail

  PRUNED
    2,100 nodes beyond depth 4
    880 below the weight floor
    → reported as counts, not hidden
```

This is the output that made Meridian confirm `svc-devops-ai` as a crown jewel (`01 §6.2`). Not the path analysis — the blast radius. *"If this one identity is manipulated, here is everything that follows"* is a more immediate argument than a path diagram.

### 12.2 A night on the change feed

```
  02:04  EDGE_ADDED    PIP-gha ─CAN_ASSUME→ ROL-ghadeploy
         significance CRITICAL — creates 1,240 paths, 3 to crown jewels
         source: aws.iam.roles + github.oidc_trusts
         attribution: trust policy modified 2026-05-22 by
                      terraform-cloud run 8841
         ⚠ detected 84 days after it was made — because the GitHub
           connector was only onboarded this week. The feed says so.

  06:14  EDGE_ADDED    IDN-svc-reporting ─CAN_ASSUME→ ROL-dbadmin
         significance CRITICAL — new path to prod-payments-db
         attribution: iam:AttachRolePolicy at 06:11 by
                      priya.s via console
         detected 3 minutes after the change, via incremental
         closure (190 ms)

  06:22  EDGE_REMOVED  41 edges from 6 tombstoned roles
         significance LOW — routine decommissioning
         coverage window: aws.iam.roles · account 123456789012 ·
                          complete · 04:03
         ← the window is cited. Without it, no removal event.

  06:31  1,847 pod-level changes across 3 EKS clusters
         significance SUPPRESSED — equivalence class
         "workload template payments-api, identical service account"
         → reported as ONE event: "scaled 4 → 11"
```

The 02:04 entry is worth noting. The edge is 84 days old and we detected it today because a connector was just onboarded. Reporting it as "new" would be a lie; reporting it as 84 days old with the detection reason is the honest form, and it is only possible because `first_seen` carries the *edge's* age rather than our discovery time.

### 12.3 Fix verification

```
  2026-08-17 11:40  Meridian applies choke point #1 — the OIDC
                    subject condition tightened

  11:44  github.oidc_trusts collector runs (forced)
  11:44  E8 re-evaluates: the primitive no longer matches
         → synthesized edge RETRACTED, with a coverage window
  11:44  path engine: reverse index → 1,240 paths recomputed
         → 1,240 ELIMINATED, 2 residual as predicted

  11:45  CHANGE FEED
         EDGE_REMOVED  PIP-gha ─CAN_ASSUME→ ROL-ghadeploy
         significance HIGH — 1,240 paths eliminated,
                             9 crown jewels protected
         attribution: remediation applied, ticket MER-4471

  11:45  EXPOSURE SCORE  72 → 58

  The customer applied a change and has evidence, four minutes
  later, that it did what was predicted — including the 2 residual
  paths that the simulation said would remain.
```

---

*Next: [Local analytics](08-local-analytics.md)*

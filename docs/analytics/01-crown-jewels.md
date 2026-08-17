# Crown Jewels — Criticality Designation

**Series:** [Analytics](00-index.md) · **Blocks:** everything else in this series

---

## 1. Purpose

Criticality answers *what is worth protecting*. It is the target the path engine computes toward, the multiplier in every score, and the input to the response protected-asset list.

`../13-contracts.md` gives every node `criticality: 0-100, default 0` and **no mechanism to set it.** Every competitor has one — Tenable's ACR, SCC's resource value configurations, XM's machine tagging, Microsoft's critical asset classification, BloodHound's Tier Zero. Until we have one, the path engine computes toward nothing and every score multiplies by zero.

This document is first in the series because everything else consumes it.

---

## 2. Position

```
  INPUTS
    customer declaration        CSV import, UI designation
    CMDB business services      ServiceNow tiers
    data classification         DLP, Purview, Macie, our own
    cloud and infra tags        env, owner, cost centre
    identity privilege          admin-tier roles and groups
    inference rules             derived, model-suggested

  OUTPUT
    criticality 0-100 on every node, with its derivation recorded

  CONSUMED BY
    03 path engine     the traversal target
    04 scoring         the impact multiplier
    05 choke points    which paths count as critical
    06 metrics         "crown jewels at risk"
    ../05 controller   the response protected-asset list
```

---

## 3. Mechanics

### 3.1 Six sources, weighted and reconciled

```
  1  CUSTOMER DECLARATION                     authority 1.00
     CSV import or UI designation.
     Always wins. A human said so.

  2  CMDB BUSINESS SERVICE TIER               authority 0.90
     ServiceNow business services and their criticality tiers.
     The customer already wrote down what matters — importing it
     means crown jewels come from their own definitions, which is
     far more defensible in a review than our inference.

  3  DATA CLASSIFICATION                      authority 0.85
     PII / PCI / PHI present, weighted by record-count bucket.
     From DLP, Purview, Macie, Snowflake classification, or ours.

  4  IDENTITY PRIVILEGE TIER                  authority 0.85
     Domain Admin, Global Administrator, org-management-account
     roles, Tier Zero equivalents. An identity that controls the
     estate IS a crown jewel.

  5  INFRASTRUCTURE ROLE                      authority 0.75
     identity providers · certificate authorities · CI/CD systems ·
     secret stores · backup systems · the Overlook appliance itself
     → compromise of any of these is compromise of everything
       downstream

  6  TAGS AND INFERENCE                       authority 0.50
     env=production · a cluster running a tier-1 service ·
     a datastore reached by a business-critical application
```

### 3.2 Reconciliation

```
  criticality = max over sources of (source_value × source_authority)

  MAX, not sum. A datastore that is both PII-classified and
  CMDB-tier-1 is not twice as critical — it is critical.
  Summing produces runaway scores and a top-10 list of things
  that merely have many labels.

  Every contributing source is RECORDED:
    criticality: 95
    derived_from:
      - { source: cmdb.business_service, value: 100, authority: 0.90,
          detail: "Payments — Tier 1" }
      - { source: dlp.classification, value: 95, authority: 0.85,
          detail: "PII + PCI, 1M-10M records" }
      - { source: tags, value: 80, authority: 0.50,
          detail: "env=production" }
    overridden_by: null
```

Recording derivation is not optional. The first question a customer asks about a crown jewel is *"why is this on the list?"*, and *"the model decided"* is not an answer.

### 3.3 Bands

```
  90-100   CROWN JEWEL     path computation targets these
   70-89   HIGH            included when the crown-jewel set is small
   40-69   MEDIUM          blast-radius destinations, not path targets
    1-39   LOW
       0   UNCLASSIFIED    the default, and a coverage gap
```

Only 90+ drives path computation. That is the scoping mechanism that keeps `03` tractable.

### 3.4 Model-suggested, customer-overridable

Following Tenable's ACR pattern explicitly:

```
  Overlook SUGGESTS a criticality with its reasoning.
  The customer CONFIRMS, ADJUSTS or REJECTS.
  An override is permanent, attributed, and survives re-derivation.

  ┌──────────────────────────────────────────────────────────┐
  │ SUGGESTED CROWN JEWELS                       14 pending  │
  │                                                          │
  │ prod-payments-db                    suggested 95         │
  │   CMDB: Payments, Tier 1                                 │
  │   DLP: PII + PCI, 1M-10M records                         │
  │   reached by 3 internet-facing applications              │
  │   [ Confirm ]  [ Adjust ]  [ Not critical ]              │
  │                                                          │
  │ svc-devops-ai                       suggested 92         │
  │   effective privilege: ADMIN across 41 accounts          │
  │   an identity that controls the estate is a crown jewel  │
  │   [ Confirm ]  [ Adjust ]  [ Not critical ]              │
  └──────────────────────────────────────────────────────────┘
```

### 3.5 Criticality is a shared attribute

Borrowed from Microsoft, where critical-asset classification feeds Defender for Cloud rather than living inside one engine.

```
  CONSUMED BY
    path engine        traversal target
    scoring            impact multiplier
    response policy    protected-asset list — a domain controller at
                       criticality 100 is NEVER auto-actioned
    connector priority a source feeding a crown jewel is collected
                       more often
    evidence retention longer retention for evidence touching crown
                       jewels
    alerting           the only class that pages anyone
```

---

## 4. Considerations

**If everything is critical, nothing is.** The single most common failure. A customer who marks 4,000 assets crown jewels gets a path engine computing 500,000 paths and a top-10 list that means nothing. Target **50–500 crown jewels** for an enterprise, and warn above that with the computational consequence stated.

**The opposite failure is worse and more common.** A customer who designates three gets a graph computing toward almost nothing, and the product looks empty. Hence the `crown_jewels` CSV collector (`../connectors/09 §6`) — if designation is one-at-a-time in a UI, they will designate three; if it is a spreadsheet import, they will designate two hundred.

**Identities are crown jewels too.** Every competitor except BloodHound treats crown jewels as data and compute. But `svc-devops-ai` with admin across 41 accounts is more critical than most databases, and Tier Zero is exactly this insight. Do not restrict designation to `DATASTORE` and `ASSET`.

**Criticality must be stable.** If it swings because a DLP scan re-ran, every score moves and the exposure trend becomes noise. Changes should be damped and, above a threshold, surfaced for confirmation rather than applied silently.

**Do not let criticality be gamed.** If disconnecting the DLP connector drops classification and therefore criticality and therefore the exposure score, the score is worse than useless. See `06 §5` — coverage-adjusted scoring.

**Record-count bucket, not exact count.** A datastore with 4.2M PII records is critical; one with 40 is a finding of a different kind. But the exact number never leaves the appliance (`../13-contracts.md §IX`), so criticality derivation uses the bucket.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| No crown jewels designated | Path engine computes nothing; product appears empty | Suggestion queue on first run; CSV import; onboarding survey question |
| Everything designated critical | 500k paths, meaningless top-10 | Warn above 500 with the computational cost stated |
| Criticality only on datastores | Privileged identities and CI systems invisible as targets | Designation applies to every node type |
| Derivation not recorded | "Why is this critical?" unanswerable | `derived_from` mandatory |
| Criticality swings on a rescan | Exposure trend becomes noise | Damping plus confirmation above a delta threshold |
| Disconnecting a source lowers criticality | Score is gameable | Coverage-adjusted scoring (`06 §5`) |
| Customer override lost on re-derivation | Trust destroyed once | Overrides are permanent and attributed |

---

## 6. Example: Meridian

```
  DESIGNATION SOURCES ACTIVE
    ServiceNow business_services   ✓  42 services with tiers
    Forcepoint classification      ✓  4,100 datastores labelled
    AWS/Azure tags                 ✓  env, owner, cost-centre
    identity privilege tier        ✓  derived from closure
    customer CSV                   ✓  imported at onboarding, 31 rows

  RESULT
    crown jewels (90-100)      47
    high (70-89)              212
    medium (40-69)          1,840
    unclassified (0)      2.89 million   ← the vast majority, correctly
```

### 6.1 One crown jewel, derived

```
  prod-payments-db                                  criticality 95

  derived_from
    cmdb.business_service   value 100 · authority 0.90 → 90
      "Payments processing — Tier 1"
    dlp.classification      value  95 · authority 0.85 → 81
      "PII + PCI · record_count_bucket 1M-10M"
    tags                    value  80 · authority 0.50 → 40
      "env=production, owner=payments-platform"
    customer_csv            value  95 · authority 1.00 → 95   ← MAX
      imported 2026-06-02, confirmed by shiva

  → criticality 95, from the customer's own declaration.
    The other three corroborate it, which is why nobody argued
    when it appeared at the top of the list.
```

### 6.2 A crown jewel nobody expected

```
  svc-devops-ai                                     criticality 92

  derived_from
    identity_privilege_tier  value 92 · authority 0.85 → 78
      "effective ADMIN across 41 accounts, reaches 3 crown jewels"
    inference                value 90 · authority 0.50 → 45
      "runs_as for an AI agent invocable by 340 identities"
    customer_csv             —  not declared
    cmdb                     —  no CI record

  → SUGGESTED at 92. Meridian's team had never thought of a service
    account as a crown jewel. They confirmed it.

  CONSEQUENCE
    the path engine now computes toward it as a destination, not
    only through it as a hop — which surfaced 4 paths where an
    attacker reaches the AGENT rather than the database, and then
    has everything the agent has.
```

### 6.3 What the count changed

```
  BEFORE designation was implemented
    crown jewels: 0
    paths computed: 0
    the product had nothing to say

  AFTER, with 47
    paths computed: 31,400
    after choke-point collapse: 6 critical, 41 high
    top recommendation: one OIDC subject condition, 1,240 paths,
    40 minutes of work

  HAD MERIDIAN DESIGNATED 4,000 (the failure mode)
    paths computed: ~2.1 million
    after collapse: 380 "critical"
    top recommendation: unusable
```

---

*Next: [Start conditions](02-start-conditions.md)*

# Overlook — Analytics and Intelligence

**Version:** 0.1
**Date:** 2026-08-17
**Parent:** `../01-system-design.md` §19–23, §31, §36
**Status:** Architecture. No implementation.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## ⚠ This entire series is SAAS-SIDE

Per handoff §3.2, everything in these nine documents — crown jewels,
start conditions, the path engine, scoring, choke points, exposure
metrics, blast radius, change intelligence — is **Overlook SaaS**, not
Edge Collector. It is outside the collector engineer's assignment.

The one exception is `08-local-analytics.md`, which describes
collector-local diagnostics and is subject to escalation **E6** (the
1 TB ceiling).

---

## Why this series exists

The doc set has an asymmetry:

```
  ~60,000 words     connectors · collectors · ingestion · parsing ·
                    normalization · resolution · closure · escalation ·
                    the graph · facts · privacy · transport
                    → everything that gets data INTO the graph

  ~5 pages          01 §19–23, §31, §36
                    → everything that happens AFTER
```

We specified the plumbing exhaustively and the thing the plumbing exists for barely at all. This series corrects that.

| # | Doc | Covers |
|---|---|---|
| 01 | [Crown jewels](01-crown-jewels.md) | Criticality designation — the input everything else consumes |
| 02 | [Start conditions](02-start-conditions.md) | Where an attacker begins |
| 03 | [Path engine](03-path-engine.md) | Traversal, pruning, incremental recompute |
| 04 | [Scoring model](04-scoring-model.md) | The published formula and the calibration loop |
| 05 | [Choke points](05-choke-points.md) | Collapse, fix simulation, Break Attack Path |
| 06 | [Exposure metrics](06-exposure-metrics.md) | The three metric families and the CISO number |
| 07 | [Blast radius and change](07-blast-radius-and-change.md) | Forward traversal and the temporal layer |
| 08 | [Local analytics](08-local-analytics.md) | What runs on the retained dataset |

---

## Reference — how six competing products do this

Recorded here because the design decisions in this series lean on it, and because it should not need re-researching.

### Wiz

```
  discovers assets and risks → maps them onto the graph → computes
  viable attacker traversals from an INTERNET ENTRY POINT to
  crown-jewel data → ranks by exploitability and blast radius

  TOXIC COMBINATIONS
    three individually-medium findings (vulnerable VM + public
    exposure + attached admin role) chaining into critical

  THE RULE WORTH STEALING
    any path ending at a crown jewel is AUTO-ELEVATED in priority
    regardless of intermediate scores
```

### XM Cyber

```
  correlates all validated paths to find CHOKE POINTS where paths
  converge

  COMPROMISE RISK SCORE, per device:
    inbound risk based on
      · the number of PRECEDING BREACH POINTS
      · the COMPLEXITY of attack paths toward the device
    → essentially inbound degree over the path graph, weighted by
      traversal difficulty

  machine tagging feeds risk appetite in — "high value" machines
  carry more weight in the exposure calculation

  CYENTIA RESEARCH ON THEIR DATA
    20% of choke points expose 10% or more of critical assets
```

### Google Security Command Center

The only one that publishes its formula.

```
  attack exposure score =
        priority value of exposed high-value resources
      × number of possible attack paths
      × successful attack percentages
      × number of high-value resources exposed

  resource scores 0–10 · finding scores unbounded
  HVR priority: HIGH 10 · MEDIUM 5 · LOW 1 · NONE 0
  up to 1,000 resource instances per value configuration

  ATTACKER MODEL   external, starting from the public internet
  TECHNIQUES       legitimate access, lateral movement, privilege
                   escalation, vulnerabilities, misconfigurations,
                   code execution
  CADENCE          every ~6 hours, at least daily
  INPUTS           CVE attack vectors, CVSS, Mandiant exploitability
```

### BloodHound Enterprise

```
  scores the DESTINATION, not the path.

  EXPOSURE PERCENTAGE TO TIER ZERO
    how many principals can reach Tier Zero by any path
    now also surfaced as a raw "Exposed Principals" count

  PUBLISHED FINDING
    70%+ of users in an average AD domain have at least one attack
    path to Tier Zero
```

### Stellar Cyber

A different philosophy — **detection** scoring, not exposure scoring.

```
  ALERT SCORE = Severity + Fidelity + Threat Intel
    Severity      security consequence and specificity
    Fidelity      LIKELIHOOD OF BEING A TRUE POSITIVE
    Threat Intel  suspiciousness of IPs, domains, URLs

    high fidelity or TI → score rises above base severity
    low  fidelity or TI → score falls below it

  CASE SCORE
    driven by the number of DIFFERENT ALERT TYPES associated
    Event Score per type = sum of max(Alert Score, Severity, Fidelity)

  correlation into cases runs on GraphML-based AI, organised
  around their XDR Kill Chain
```

Two things to take from it: **Fidelity is confidence as a first-class scoring dimension**, and **case score rewards breadth of corroboration** — more distinct sources agreeing raises the score.

### Tenable One

The cleanest formula hierarchy.

```
  VPR   0.1–10    vulnerability severity + exploitability
  ACR   1–10      ASSET CRITICALITY — model-generated OR user-defined
  AES             TWO-STEP:
                    1. vulnerability density = f(weakness count, VPR)
                    2. combine density with ACR, then scale
  CES   0–1000    average of AES across an exposure class
```

The two-step matters: aggregate the weaknesses *first*, then weight by criticality. Not one flat formula.

### Microsoft Security Exposure Management

```
  ENTERPRISE EXPOSURE GRAPH
    normalises asset data from Microsoft AND non-Microsoft tools,
    maps relationships, auto-generates paths from external entry
    points to business-critical targets

  DASHBOARD HEADLINE IS THREE NUMBERS, NOT ONE
    attack paths · CHOKE POINTS · critical assets

  CRITICALITY IS A SHARED SERVICE
    critical-asset classification feeds OTHER products — Defender
    for Cloud consumes it when evaluating risk
```

---

## What all seven agree on

```
  1  ASSET CRITICALITY IS SEPARATE FROM FINDING SEVERITY
     Tenable ACR · SCC priority values · XM machine tagging ·
     Microsoft critical assets · Wiz crown jewels · BloodHound Tier Zero
     → universal. And it is the field we currently have UNPOPULATED,
       which is why doc 01 comes first in this series.

  2  SCORING IS MULTIPLICATIVE AND STAGED
     aggregate first, then weight by criticality

  3  CRITICALITY IS MODEL-SUGGESTED, CUSTOMER-OVERRIDABLE

  4  CHOKE POINTS ARE THE HEADLINE, NOT PATH COUNTS
```

---

## What Overlook adds

Eight additions, each falling out of a design decision already made — which is what makes them defensible rather than aspirational.

```
  1  INFERENCE CONFIDENCE IN THE PATH SCORE
     Stellar does Fidelity for detections. Nobody does it for graph
     edges derived from multi-source entity resolution. A path built
     on a probabilistic identity merge is genuinely less trustworthy.
     → from: E6 resolution confidence (engines/05)

  2  PATH AGE
     "this path has existed 84 days" — an old path means the
     organisation is blind to it. Better than severity as a
     prioritisation signal.
     → from: the bitemporal graph (engines/09 §3.1)

  3  SYNTHESIZED-EDGE PROVENANCE IN SCORING
     an edge from an escalation primitive scores and DISPLAYS
     differently from an observed grant, with primitive id and
     rationale attached.
     → from: E8 (engines/07 §3.5)

  4  CONFIGURED VERSUS OBSERVED DISAGREEMENT
     ROUTES_TO from a rulebase vs CONNECTS_TO from flow.
     Configured-never-observed = attack surface with no
     justification. Observed-not-configured = a bypass.
     → from: collecting both paths (connectors/05)

  5  CHOKE-POINT FIX SIMULATION
     XM and Microsoft IDENTIFY choke points. Neither documents
     simulating the OPERATIONAL IMPACT of the fix.
     → from: the graph overlay (analytics/05)

  6  POPULATION-PERCENTAGE METRICS BEYOND IDENTITY
     BloodHound's "70% of users reach Tier Zero" is the most legible
     metric any of them produce. Nobody applies the shape to
     non-human identities, AI agents or CI pipelines.
     → from: having those as first-class node types

  7  THE AI PRIVILEGE GAP as a scored metric, and
     UNTRUSTED-INPUT-REACHING-AN-AGENT as a start condition
     → ours alone

  8  CROSS-CUSTOMER BENCHMARKING ON TOKENS
     "this escalation pattern appears in 9 of your 12 customers"
     → from: the MSSP model plus the privacy architecture (09 §3)
```

## And what we deliberately will not build

```
  ✕ ML anomaly scoring      Stellar's game. Needs volume and a
                            detection research team.
  ✕ Threat-intel scoring    commodity. Ingest as a property.
  ✕ Our own vuln scoring    ingest VPR and CVSS as properties.
                            Do not recompute what Tenable already did.
```

---

## The recurring example

Every document in this series ends with Meridian Financial, continuing the thread from `../12-end-to-end-deployment-story.md`.

```
  MERIDIAN FINANCIAL
    2.9M entities · 2.9M live edges · 270 synthesized escalation edges
    47 crown jewels · 183 open findings
    the critical path: laptop MCP config → GitHub token → OIDC trust
    → GHADeployRole → PassRole+Lambda → prod-payments-db
```

---

*Index. Documents follow in dependency order.*

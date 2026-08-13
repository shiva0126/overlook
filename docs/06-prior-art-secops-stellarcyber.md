# Overlook — Prior Art: Google SecOps and Stellar Cyber

**Version:** 0.1
**Date:** 2026-08-13
**Depth:** Quick survey, not deep research. Public documentation and vendor material only.
**Purpose:** Validate or refute Overlook's architectural assumptions against two products that have shipped something structurally similar.

---

## 1. Why these two

They are the two closest shipped analogues to parts of what Overlook proposes, and they fail in opposite directions:

```
   GOOGLE SecOps (Chronicle)
     solved: normalization at planetary scale, parser economics,
             non-volumetric pricing, entity aliasing
     did not attempt: local processing, data residency, privacy boundary
     → validates our PIPELINE. Says nothing about our PRIVACY claim.

   STELLAR CYBER (Open XDR)
     solved: distributed sensors + central processor, edge filtering,
             multi-tenancy from day one, regional data residency
     did not attempt: exposure graph, attack paths, relationship modelling
     → challenges our PRIVACY claim. Says nothing about our GRAPH.
```

Neither builds an exposure graph. Both have shipped infrastructure we are proposing to build.

---

## 2. Google SecOps (Chronicle)

### 2.1 What they built

```
   Customer environment                Google Cloud
   ────────────────────                ───────────
   Forwarder (software on a VM)  ───►  Ingestion
   OTel collectors                     Parsers → UDM normalization
   API / feeds / webhooks              Entity + event store
                                       Detection, search, UI
                                       ~1 year retention by default
```

**The forwarder is a forwarder.** It collects logs and network packets and ships them. It does not parse, does not normalize, does not analyse, does not decide what leaves. Everything of substance happens in Google's cloud.

### 2.2 What they got right that we should copy

**One normalization schema, enforced.** The Unified Data Model is a single canonical event schema every source is mapped into. Every UDM event carries a metadata section including both event time and ingestion time. This is exactly our Normalize stage (`04 §12`), and it confirms the design: *the value is not the parser, it is the shared vocabulary the parser targets.*

**Parser-to-logtype is one-to-one.** Chronicle enforces a strict 1:1 relationship between a parser and a log type. We should adopt that rule explicitly in the connector manifest — it makes parser health measurable per source, makes ownership unambiguous, and prevents the "one clever parser handles four vendors" mess that becomes unmaintainable.

**Parsers are a content library, forever.** They ship hundreds of default parsers and add continuously. This validates the content pipeline (`01 §17`) and confirms the uncomfortable estimate that parser maintenance never ends. It is not a project; it is a standing cost.

**Pricing decoupled from data volume.** Chronicle's commercial move was pricing by user/employee count rather than ingested GB. This is directly relevant: our architecture ships ~12 MB/day/tenant, so **volumetric pricing would be actively self-defeating** — we would be pricing a number we deliberately drove to near zero. Doc 03's open decision D10 leaned this way; this is external confirmation.

**Entity aliasing is a real, solved problem.** Chronicle maintains entity records and resolves assets across multiple identifiers. Our three-stage resolver (`01 §8.2`) is not exotic; it is table stakes for anyone building on multi-source telemetry.

### 2.3 Where they are structurally opposite to us

```
   Chronicle  = camera.  Unit of work: the EVENT. Cost scales with volume.
   Overlook   = map.     Unit of work: the RELATIONSHIP. Cost scales with entities.
```

More importantly: **Chronicle has no privacy boundary.** Everything goes to Google. That is a deliberate trade — it buys them scale, search, and a year of retention — but it structurally excludes them from every deal where data cannot leave the customer's jurisdiction or premises. That exclusion is the space Overlook is aiming at, and Chronicle's architecture cannot be retrofitted to occupy it.

---

## 3. Stellar Cyber (Open XDR)

### 3.1 What they built

```
   Customer / tenant sites              Central
   ──────────────────────               ───────
   Network sensors                ───►  Data Processor
   Security sensors                     - big data lake
   Server / endpoint sensors            - detection + correlation
   Container sensors                    - automated response
   Connectors / log forwarders          - UI
                                        cloud-native microservices,
                                        containerised

   Deployment models:
     ALL-IN-ONE   sensors + DP on one machine (turnkey)
     DISTRIBUTED  DP central, sensors spread across sites
     CLUSTERED    DP across multiple machines
```

### 3.2 What they got right that we should copy

**Three deployment topologies from one product.** All-in-one, distributed, clustered — the same software, configured differently. This is precisely the shape our S/M/L/XL profiles (`01 §10.3`) and four-process boundary (`04 §26`) should take, and it is reassuring that someone shipped it. It also validates that "one appliance image, splittable by role" is achievable rather than aspirational.

**Filtering before data leaves the source.** Their sensors apply traffic and application filters at the source rather than at a central ingestion point. That is the same principle as our aggregate-at-receive rule for flow data (`04 §6.2`) — and they market it as a differentiator, which suggests customers value it explicitly.

**Multi-tenancy built in from inception.** Full isolation of data, per-tenant storage options, retention, policies, reporting, and per-tenant ML. Their MSSP business depends on it. **We deferred multi-tenancy as "not a prerequisite" (`04 §31`).** That is defensible for a single-appliance-per-customer model, but if MSSP is ever a target channel, retrofitting tenancy is one of the most expensive things a platform can do. This deserves an explicit decision rather than a default.

**Regional data residency with centralised aggregates.** They keep data physically resident in a site or region to avoid cross-border movement, while centralising aggregate statistics for a single-pane UI and GDPR compliance.

### 3.3 The uncomfortable finding

That last bullet is the important one, and it requires us to correct our own positioning.

`01-system-design.md` §2.3 claims two defensible wedges, the first being:

> **Wedge 1 — Privacy-preserving architecture.** ... It cannot be retrofitted by a competitor without rewriting their platform, which is why it is defensible.

**That is too strong.** Stellar Cyber already ships local processing, source-side filtering, regional residency, and centralised aggregates. The *shape* — process locally, centralise derived summaries — is not novel and is not ours.

What remains genuinely differentiated is narrower and more specific:

```
   NOT DIFFERENTIATED (Stellar Cyber and others already do this)
     ✕ processing at the customer edge
     ✕ filtering before data leaves the source
     ✕ keeping raw data in-region
     ✕ centralising aggregates for a single UI

   ACTUALLY DIFFERENTIATED
     ✓ WHAT is centralised: a tokenized RELATIONSHIP GRAPH,
       not aggregate statistics or detections
     ✓ Deterministic tenant-keyed tokenization, with the key held
       by the customer and never transmitted (01 §7.2)
     ✓ Browser-to-Edge de-tokenization — plaintext names never
       transit the vendor at all (01 §7.4)
     ✓ The claim "total compromise of our SaaS yields tokens and
       edge types, nothing else" (01 §33)
     ✓ Cross-domain exposure graph including AI agents, MCP, RAG
       as first-class node types
```

The defensible position is not *"we process at the edge."* It is *"what crosses the boundary is a graph of tokens, and we cannot read it."* That is a sharper, more honest, and more defensible claim — and it should be rewritten into `01 §2.3`.

### 3.4 Where they are structurally opposite to us

Stellar Cyber is an **XDR/SIEM**: events, detections, alerts, response. Their Data Processor holds a data lake. Their unit of work is the event and their cost scales with volume — which is why filtering is a headline feature for them.

Overlook's unit of work is the relationship. We do not hold a lake. Our reduction is not 10:1 filtering, it is 200,000:1 abstraction. Different product, different economics, adjacent architecture.

Also worth noting: their Data Processor is typically deployed **by the MSSP or the customer**, in their own data centre or VPC. It is not vendor-hosted SaaS. So their "centralisation" is centralisation *within the customer's or partner's trust boundary* — which sidesteps the privacy problem rather than solving it, and produces a heavier operational burden for the customer.

---

## 4. Side by side

| | Google SecOps | Stellar Cyber | Overlook |
|---|---|---|---|
| Unit of work | Event | Event | **Relationship** |
| Customer-side component | Forwarder (ships raw) | Sensor (filters, some processing) | **Edge Node (full analysis)** |
| Where analysis happens | Vendor cloud | Central DP (customer/MSSP-hosted) | **Customer edge + vendor graph** |
| What crosses the boundary | Everything | Filtered events / aggregates | **Tokenized facts only** |
| Vendor can read customer data | Yes | N/A (self-hosted) | **No, by construction** |
| Normalization schema | UDM | Internal | Security Fact + entity model |
| Parser model | 1:1 with log type, content library | Connector library | 1:1 with source, content library |
| Multi-tenancy | Vendor-managed | **From inception, MSSP-first** | Deferred — needs a decision |
| Deployment topologies | Forwarder only | **All-in-one / distributed / clustered** | S/M/L/XL profiles |
| Pricing | **Non-volumetric (per user)** | Volume/tenant based | Undecided — should be non-volumetric |
| Attack paths / exposure graph | No | No | **Yes** |
| AI/agent/MCP as graph entities | No | No | **Yes** |

---

## 5. What we should change

```
  C1  REWRITE 01 §2.3 (the wedges)
      "Privacy-preserving architecture" is not defensible as stated.
      Replace with the four specific claims in §3.3 above.
      Edge processing is table stakes; tokenized graph centralisation
      is the differentiator.

  C2  ADOPT 1:1 PARSER-TO-SOURCE as an explicit manifest rule
      Chronicle enforces it. It makes parser health measurable and
      ownership unambiguous. Add to the connector manifest schema.

  C3  DECIDE MULTI-TENANCY NOW, not later
      Stellar Cyber's MSSP business is built on tenancy from inception.
      If MSSP is ever a channel, deferring this is a mistake that
      compounds. Needs an explicit yes/no.

  C4  COMMIT TO NON-VOLUMETRIC PRICING
      We deliberately drove data volume to ~12 MB/day/tenant.
      Pricing on volume would price our own best property at zero.
      Chronicle proves the market accepts per-user/per-asset pricing.

  C5  NAME THE DEPLOYMENT TOPOLOGIES EXPLICITLY
      Borrow Stellar Cyber's vocabulary: all-in-one, distributed,
      clustered. Clearer than S/M/L/XL, which describes size rather
      than shape. Use both: shape × size.

  C6  ADD A COMPETITIVE SECTION to the system design
      Right now no document states who we are unlike and why.
      One page, honest, including where we are NOT differentiated.
```

---

## 6. What was confirmed

Reassuring, and worth recording so we stop re-litigating it:

```
  ✓ A single canonical schema every source maps into — correct, both do it
  ✓ Parsers as versioned content, maintained forever — correct
  ✓ Entity resolution / aliasing as a core service — correct, not exotic
  ✓ One image, multiple deployment shapes — shipped by Stellar Cyber
  ✓ Filtering/reduction at the source — shipped, and marketed as valuable
  ✓ Regional residency with central aggregates — shipped, regulator-accepted
  ✓ Non-volumetric pricing viable in this market — proven by Chronicle
```

---

## 7. What this survey did not cover

Stated so the gaps are visible rather than assumed:

```
  - Chronicle's entity graph depth — do they model relationships,
    or only entity records with aliases? Matters for how close they
    could get to exposure analysis if they chose to.
  - Stellar Cyber's actual sensor-side processing depth — "filters"
    could mean anything from a drop rule to real analytics.
  - Neither vendor's IAM/permission modelling, because neither
    appears to do it. Worth confirming.
  - Pricing specifics for either.
  - The actual exposure-graph competitors (Wiz, XM Cyber, Tenable One),
    which are a separate and more direct comparison.
```

That last line is the important omission. These two validate our *plumbing*. The products that threaten our *thesis* are the exposure-graph vendors, and they deserve their own survey.

---

## 8. Open questions for the next session

```
  Q1  Is MSSP a target channel? Determines C3, and it is expensive
      to answer late.
  Q2  Do we accept the corrected positioning in §3.3, and should
      01 §2.3 be rewritten now?
  Q3  Should the next survey be Wiz / XM Cyber / Tenable One —
      the direct exposure-graph competitors?
  Q4  Deployment shape vocabulary: adopt all-in-one / distributed /
      clustered?
```

---

*End of document.*

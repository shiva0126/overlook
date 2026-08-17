# Overlook — Prior Art: Google SecOps and Stellar Cyber

**Version:** 0.1
**Date:** 2026-08-13
**Depth:** Survey of public documentation and vendor material. Chronicle covered in depth; Stellar Cyber at survey level.
**Purpose:** Validate or refute Overlook's architectural assumptions against two products that have shipped something structurally similar.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Why these two

They are the two closest shipped analogues to parts of what Overlook proposes, and they fail in opposite directions:

```
   GOOGLE SecOps (Chronicle)  + Security Command Center
     solved: normalization at planetary scale, parser economics,
             non-volumetric pricing, entity resolution and aliasing,
             data residency, and — in SCC — attack paths, toxic
             combinations and chokepoints across GCP and AWS
     did not attempt: local processing, any privacy boundary,
             on-prem directory depth, AI agent identity
     → validates our PIPELINE. Contests our GRAPH.
       Leaves only the BLINDNESS claim untouched.

   STELLAR CYBER (Open XDR)
     solved: distributed sensors + central processor, edge filtering,
             multi-tenancy from day one, regional data residency
     did not attempt: exposure graph, attack paths, permission modelling
     → contests our EDGE story. Says nothing about our GRAPH.
```

Between them they have shipped most of the infrastructure Overlook proposes. The survey's value is in identifying precisely what they have *not* shipped — which turns out to be narrower than the earlier drafts assumed.

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

### 2.3 The Entity Context Graph — closer than expected, but a different object

The first pass assumed Chronicle had no graph. That was wrong, and the correction matters.

Chronicle ships an **Entity Context Graph (ECG)**. It stores assets, users, resources, groups and IOCs. It merges context from identity providers, CMDB systems (ServiceNow, Duo), vulnerability management, Windows AD, Entra ID, Okta and cloud IAM into a single consolidated entity profile. It maintains an **Aliasing Service** that tracks users and assets over time and merges identifiers — which is, functionally, entity resolution. It forms relationships between entities and derives computed attributes such as prevalence and first/last seen.

That is a substantial overlap with `01 §6` (entity model) and `01 §8` (entity resolution). Our resolver is not novel; it is parity.

**But the ECG is a different kind of object, and two details give it away:**

```
  1. THE RELATIONSHIP SEMANTICS ARE ASSOCIATIVE, NOT PERMISSIVE.
     ECG relations express "this user is associated with this group",
     "this asset is associated with this IP".
     They do NOT express CAN_ASSUME, CAN_READ, or effective permission.
     There is no policy evaluation, no permission closure, no
     escalation primitive synthesis.

  2. RELATIONSHIPS ARE FORMED FOR A ONE-DAY INTERVAL.
     The ECG is rebuilt around event time to enrich detection.
     It is not a persistent structural graph with edge lifecycle.

     You cannot compute "this path has existed for 41 days" from a
     graph whose relations are scoped to a day. Our bitemporal model
     (01 §9.2) is answering a question the ECG is not built to ask.
```

So: **Chronicle's graph enriches detections. Ours computes exposure.** Same word, different object. The ECG makes an alert more informative; the TrustGraph tells you what to fix before there is an alert.

### 2.4 Data residency — they have it

Google SecOps enforces data residency controls, with regional endpoints, applied to data at rest by default, and to data in use and in transit on request.

This directly undercuts any Overlook positioning built on residency. Chronicle customers in regulated jurisdictions already have a residency answer. Residency is not our wedge and must not be sold as one.

### 2.5 Risk Analytics

Chronicle has entity-centric risk scoring: a Risk Analytics dashboard with behavioural analytics ranking entities by risk score, plus watchlists driven by internal enterprise risk calculations.

This is **behavioural** risk — an entity is risky because it behaved unusually. Ours is **structural** risk — an entity is risky because of what it can reach. Complementary rather than competing, but it means "we score entity risk" is not a differentiating sentence either.

### 2.6 The finding that changes the competitive picture

Google already ships attack path analysis. Not in SecOps — in **Security Command Center**.

```
   SCC RISK ENGINE
     attack path simulation
     attack exposure scores (difficulty for a hypothetical attacker
        to traverse from a public-internet entry point to a
        high-value resource)
     toxic combinations
     CHOKEPOINTS
     coverage: Google Cloud AND AWS
     requires organization-level activation
```

Attack paths, toxic combinations and chokepoints are the exact vocabulary of `01 §20`, `01 §19.2` and `01 §31`. A hyperscaler is shipping them, across two clouds, potentially bundled into a platform the customer already owns.

**Constraints that leave us room:**

```
   ✕ Cloud only. SCC cannot reach on-prem Active Directory ACLs,
     Kerberos delegation, Entra application privilege, or network
     device configuration.
   ✕ Org-level activation required — not available to project-level
     customers.
   ✕ No AI agent, MCP or RAG entities.
   ✕ Google reads everything. No privacy boundary of any kind.
   ✕ Strongest inside Google Cloud; a GCP-centric answer for a
     multi-cloud, hybrid problem.
```

### 2.7 Where they are structurally opposite to us

```
   Chronicle  = camera.  Unit of work: the EVENT. Cost scales with volume.
   Overlook   = map.     Unit of work: the RELATIONSHIP. Cost scales with entities.
```

And the boundary difference remains absolute: **everything goes to Google.** Residency controls place data in a chosen region; they do not make Google unable to read it. That distinction — residency versus blindness — is the one claim that survives this entire survey.

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

**Three deployment topologies from one product.** All-in-one, distributed, clustered — the same software, configured differently. This is precisely the shape our Edge S/M/L editions (`01 §10.3`) and four-process boundary (`04 §26`) should take, and it is reassuring that someone shipped it. It also validates that "one collector image, splittable by role" is achievable rather than aspirational.

**Filtering before data leaves the source.** Their sensors apply traffic and application filters at the source rather than at a central ingestion point. That is the same principle as our aggregate-at-receive rule for flow data (`04 §6.2`) — and they market it as a differentiator, which suggests customers value it explicitly.

**Multi-tenancy built in from inception.** Full isolation of data, per-tenant storage options, retention, policies, reporting, and per-tenant ML. Their MSSP business depends on it. **We deferred multi-tenancy as "not a prerequisite" (`04 §31`).** That is defensible for a single-collector-per-customer model, but if MSSP is ever a target channel, retrofitting tenancy is one of the most expensive things a platform can do. This deserves an explicit decision rather than a default.

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
| Customer-side component | Forwarder (ships raw) | Sensor (filters, some processing) | **Edge Collector (full analysis)** |
| Where analysis happens | Vendor cloud | Central DP (customer/MSSP-hosted) | **Customer edge + vendor graph** |
| What crosses the boundary | Everything | Filtered events / aggregates | **Tokenized facts only** |
| Vendor can read customer data | Yes | N/A (self-hosted) | **No, by construction** |
| Normalization schema | UDM | Internal | Security Fact + entity model |
| Parser model | 1:1 with log type, content library | Connector library | 1:1 with source, content library |
| Multi-tenancy | Vendor-managed | **From inception, MSSP-first** | Deferred — needs a decision |
| Deployment topologies | Forwarder only | **All-in-one / distributed / clustered** | Edge S/M/L, hard ceiling |
| Pricing | **Non-volumetric (per user)** | Volume/tenant based | Undecided — should be non-volumetric |
| Attack paths / exposure graph | No | No | **Yes** |
| AI/agent/MCP as graph entities | No | No | **Yes** |

---

## 5. What we should change

```
  C1  REWRITE 01 §2.3 (the wedges)                          ✓ DONE
      Replaced with three specific claims: residency-is-not-blindness,
      hybrid depth, and AI agents inside the permission closure.
      Also added 01 §2.5, a competitive landscape section stating
      plainly what we are NOT differentiated on.

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
      clustered. Clearer than Edge S/M/L, which describes size rather
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
  ✓ RESOLVED — Chronicle's entity graph depth. It exists (ECG), does
    entity resolution and aliasing, but its relationships are
    associative and scoped to a one-day interval. No permission
    semantics, no closure. See §2.3.
  ✓ RESOLVED — Chronicle data residency. It exists. See §2.4.
  ✓ RESOLVED — Google does ship attack paths and chokepoints, in
    Security Command Center, not SecOps. See §2.6.

  STILL OPEN
  - Stellar Cyber's actual sensor-side processing depth — "filters"
    could mean anything from a drop rule to real analytics.
  - Whether SCC's Risk Engine does any real IAM permission closure,
    or only reachability plus finding severity.
  - Pricing specifics for any of them.
  - The direct exposure-graph competitors: Wiz, XM Cyber, Tenable One.
  - BloodHound Enterprise — the closest thing to our AD/Entra depth.
```

The remaining line is the important one. These two validate our *plumbing*. The products that threaten our *thesis* are the exposure-graph vendors — Wiz, XM Cyber, Tenable One, and BloodHound Enterprise — and they deserve their own survey.

---

## 9. The decision taken

After the deeper Chronicle pass, the positioning has been rewritten in `01 §2.3`. Recorded here so the reasoning is not lost:

```
  DROPPED — no longer claimed as differentiation
    edge-local processing        Stellar Cyber ships it
    data residency               Google SecOps and Stellar Cyber ship it
    entity graph + resolution    Chronicle ECG ships it
    attack paths and chokepoints Google SCC Risk Engine ships it,
                                 across GCP and AWS

  KEPT — three claims, each narrow and testable
    1. RESIDENCY IS NOT BLINDNESS
       Everyone offers "your data stays in your region."
       Nobody offers "we cannot read what you send us."
       Mechanism: customer-held tokenization key + browser-to-Edge
       de-tokenization. Plaintext never transits the vendor.

    2. HYBRID DEPTH
       AD ACLs + Kerberos delegation + Entra app privilege +
       three cloud IAM models + K8s RBAC + federation trusts,
       in ONE permission closure.
       SCC and Wiz stop at the VPN. BloodHound has no cloud IAM.

    3. AI AGENTS INSIDE THAT CLOSURE
       Not an AI dashboard — AI_AGENT and MCP_SERVER as nodes in the
       same closure as ROLE and DATASTORE, so AI exposure is
       expressed in entitlement terms.

  THE ONE-SENTENCE VERSION
    The only exposure graph spanning on-prem directories, multi-cloud
    IAM and AI agent identity in a single permission closure — and the
    only one whose vendor cannot read the graph it stores.

  THE DISCIPLINE THAT FOLLOWS
    Every capability discussion ends with: what finding does this
    enable that Google, Wiz or XM Cyber structurally CANNOT produce?
    "A nicer version of theirs" is not a differentiator.
```

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

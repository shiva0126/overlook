# Overlook — Connector Benchmark and Alignment

**Version:** 0.1
**Date:** 2026-08-13
**Companion to:** `03-connectors.md`, `06-prior-art-secops-stellarcyber.md`, `07-competitive-landscape.md`
**Purpose:** Compare Overlook's proposed 118-connector catalog against what shipped products actually have, extract the takeaways, and decide what we must align with.

**Confidence:** ✔ verified in this session · ○ from prior knowledge, verify before external use.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. The headline

```
   Stellar Cyber   86+ connectors, 13 categories        ✔
   Overlook        118 connectors, 9 domains (proposed)

   OVERLAP:  roughly 30%.

   They are not the same kind of object.
```

Stellar Cyber connects to **alert producers**. Overlook must connect to **configuration and permission producers**. The vendor names overlap; the APIs, credentials, data shapes and purposes do not.

That single distinction is the most useful thing in this document, and it changes how we count, build and sequence connectors.

---

## 2. Stellar Cyber's actual catalog

Verified from their 5.3.x documentation. ✔

| # | Category | Count | Named examples |
|---|---|---|---|
| 1 | Password Management | 1 | 1Password |
| 2 | Endpoint Security | 17 | EDR/AV vendors |
| 3 | IdP | 5 | Active Directory, Duo, JumpCloud, Okta, OneLogin |
| 4 | Web Security | 6 | SWG / proxy |
| 5 | PaaS | 6 | AWS CloudTrail, AWS CloudWatch, Azure Event Hub, Google Cloud Audit, Generic S3, Oracle Cloud |
| 6 | Firewall | 13 | mostly **Respond** function |
| 7 | Email | 4 | Barracuda, Broadcom, Mimecast, Proofpoint |
| 8 | SaaS | 5 | Box, Google Workspace, Defender for Cloud Apps, O365, Salesforce |
| 9 | Vulnerability Scanner | 7 | Nessus, Qualys, Rapid7, Tenable.io/sc, CyberCNS, CYRISMA |
| 10 | DNS Security | 2 | Cisco Umbrella, HYAS Protect |
| 11 | NDR | 1 | ExtraHop Reveal(x) 360 |
| 12 | Cloud Security | 2 | Broadcom Workload Protection, Prisma Cloud |
| 13 | Webhook / Remote | 4 | Universal Webhook, ESET, Remote SSH, HanDreamNet |
| | **Total** | **~73 named, 86+ with third-party integrations** | |

### 2.1 Their connector function model ✔

```
   COLLECT       pull data in
   RESPOND       push an action out (block on firewall, disable user)
   THIRD-PARTY   integration maintained outside the core
```

A connector declares one or more functions. Their Firewall category is **primarily Respond** — 13 firewall connectors that mostly exist to push blocks, not to read rulebases.

This validates our read/response split (`03 §9`, `05 §21`) and confirms it as an industry norm rather than an Overlook invention.

### 2.2 Their configuration model ✔

Every connector takes roughly the same shape:

```
   Platform URL          base API endpoint
   API Token             authentication
   Interval (minutes)    collection frequency
   Content Type          which data classes to pull
   SSL verify            toggle
   Tenant                multi-tenancy assignment
   Device / DP           which Data Processor executes it
```

Two things to note:

- **Tenant is a first-class field on every connector.** Multi-tenancy is not bolted on; it is in the connector's primary configuration. This is what "built in from inception" actually looks like — and it shows the cost we avoid entirely by going multi-instance instead (`09 §2`).
- **"Device/DP selection"** — the operator chooses *which processor* runs the connector. Their distributed model exposes execution placement in the connector config. Overlook needs the same field: the hybrid archetype (`09 §4`) deploys two Edge Collectors per customer.

### 2.3 How they build connectors ✔

> API connectors are developed per request and released with new versions.

**This is a significant strategic weakness, and we should not copy it.** Connectors ship on the product release train, built to order. That means: slow turnaround for a customer with an unusual source, engineering time consumed by bespoke requests, and no community or customer self-service.

Our manifest-driven SDK plus a generic REST poller (`03 §4.2`, `03 §2.9`) is a better model precisely because it converts the long tail from an engineering queue into a configuration exercise.

---

## 3. The category mismatch

Overlay their 13 categories on our 9 domains and the gap is stark.

```
  THEIR CATEGORY          COUNT   OUR EQUIVALENT              OURS
  ─────────────────────   ─────   ────────────────────────    ────
  Endpoint Security         17    Security tooling              17   ≈ match
  Firewall                  13    Network & edge                15   ≈ match
  Vulnerability Scanner      7    Security tooling (subset)      -   ≈ match
  Web Security               6    Network & edge (subset)        -   they lead
  PaaS                       6    Cloud & infrastructure        10   DIFFERENT (§3.1)
  IdP                        5    Identity & access             16   WE LEAD 3x
  SaaS                       5    Business context / Data        -   partial
  Email                      4    —                              0   they lead
  Webhook / Remote           4    Generic ingestion              7   ≈ match
  DNS Security               2    Network & edge (subset)        -   ≈ match
  Cloud Security             2    Security tooling (subset)      -   ≈ match
  NDR                        1    Network & edge (subset)        -   ≈ match
  Password Management        1    Identity & access (subset)     -   ≈ match
  ─────────────────────   ─────   ────────────────────────    ────
  (nothing)                  0    Code, build, artifacts        12   THEY HAVE NONE
  (nothing)                  0    Data platforms                18   THEY HAVE NONE
  (nothing)                  0    AI platforms                  15   THEY HAVE NONE
  (nothing)                  0    Kubernetes                     -   THEY HAVE NONE
  (nothing)                  0    Business context (CMDB/HR)     8   THEY HAVE NONE
```

**Stellar Cyber has zero connectors for the sources that produce most of our graph edges.** No GitHub, no GitLab, no Jenkins, no Kubernetes, no databases, no data warehouses, no AI platforms, no ServiceNow CMDB, no Workday.

That is not an oversight on their part. It is a correct decision *for an XDR*. Those sources do not emit alerts.

### 3.1 The PaaS trap — the most important detail in this document

Their cloud connectors are **AWS CloudTrail, AWS CloudWatch, Azure Event Hub, Google Cloud Audit, Generic S3**.

Every one of those is a **log stream**. Not one is a configuration API.

```
   STELLAR CYBER connects to:        OVERLOOK must connect to:
   ──────────────────────────        ─────────────────────────
   AWS CloudTrail                    AWS IAM
     "who did what"                    "who CAN do what"
     event stream                      policy documents
     read from a log                   ListRoles, GetRolePolicy,
                                       SimulatePrincipalPolicy
     grows forever                     bounded, re-fetchable
     answers: did it happen?           answers: is it possible?
```

Same cloud, same vendor name in a catalog, **completely different connector**: different API surface, different IAM permissions to grant, different rate limits, different data shape, different failure modes, different value.

The practical consequence: **"AWS" appearing in a competitor's connector list tells you almost nothing about whether they can build a permission closure.** Stellar Cyber's AWS coverage is 6 log sources. Ours is 28 collectors, most of them configuration APIs.

This also means our connector-count comparisons with XDR/SIEM vendors are meaningless. We are not behind them at 118 versus 86. We are building a different catalog.

---

## 4. How the others do connectors

| Vendor | Model | Count | Notes |
|---|---|---|---|
| **Stellar Cyber** ✔ | API connectors + sensor log parsing. Built per customer request, shipped on release train. Collect/Respond/Third-party functions. Tenant and executing-DP as first-class config. | 86+ | Alert-producer oriented |
| **Google SecOps** ✔ | Forwarder + OTel + feeds + webhooks. **Strict 1:1 parser-to-logtype.** Large Google-maintained default parser library, continuously extended. All normalization to UDM in the cloud. | hundreds of log types | The biggest parser library in the industry |
| **Wiz** ✔ | **Agentless, API-only, connects in minutes.** One cloud role per account. The entire product is a small number of very deep cloud connectors plus scanning. | few, very deep | Depth over breadth — the opposite strategy |
| **BloodHound Enterprise** ✔ | Collectors for AD, Entra, AWS, Okta, GitHub. Directory-focused, deep permission semantics. | ~5, very deep | Depth over breadth |
| **XM Cyber** ○ | Hybrid: agents/scanners on-prem plus cloud APIs. Broad but heavier deployment. | dozens | Breadth with deployment cost |

### 4.1 The two strategies

```
   BREADTH               SIEM/XDR: Stellar Cyber, Google SecOps
                         Value = number of sources you can ingest.
                         Connector count IS the product.
                         86 to hundreds.

   DEPTH                 Exposure graph: Wiz, BloodHound
                         Value = semantic richness per source.
                         Connector count is nearly irrelevant.
                         Wiz connects to 3 clouds and beat everyone.
                         5 to 15 connectors.
```

**Overlook is a depth product that wrote a breadth plan.**

Wiz became the fastest-growing security company in history with essentially three connectors — AWS, Azure, GCP — done extraordinarily well. BloodHound owns identity attack paths with about five. Neither competed on catalog size.

Doc 03 proposed 118 connectors and 695 collectors. Against a depth strategy, that is the wrong target to optimise.

---

## 5. Key takeaways

```
  T1  CONNECTOR COUNTS ACROSS CATEGORIES ARE NOT COMPARABLE.
      86 alert-source connectors and 118 configuration-source
      connectors measure different things. Stop benchmarking
      against XDR catalogs.

  T2  "SUPPORTS AWS" IS MEANINGLESS WITHOUT SAYING WHICH APIS.
      CloudTrail is not IAM. Log stream is not configuration.
      Our marketing and our manifests must state the API surface,
      not the vendor name.

  T3  DEPTH BEATS BREADTH IN THIS CATEGORY — DEMONSTRATED TWICE.
      Wiz: 3 clouds. BloodHound: 5 sources. Both category leaders.
      118 connectors is a distraction from the thing that wins.

  T4  THE COLLECT/RESPOND SPLIT IS AN INDUSTRY NORM.
      Stellar Cyber's Firewall category is mostly Respond.
      Our separate read and response roles (03 §9) are correct
      and should stay separate in credentials and manifests.

  T5  MULTI-TENANCY LIVES IN THE CONNECTOR CONFIG — BUT NOT FOR US.
      Tenant is a primary field on every Stellar Cyber connector.
      RESOLVED 2026-08-13: Overlook is multi-INSTANCE (one collector
      per customer), so no tenant field is required. See doc 09.

  T6  EXECUTION PLACEMENT IS A CONNECTOR-LEVEL FIELD.
      They let the operator pick which Data Processor runs a
      connector. Multi-Edge Overlook needs the same field.

  T7  BUILDING CONNECTORS PER CUSTOMER REQUEST DOES NOT SCALE.
      Stellar Cyber ships them on the release train. Our
      manifest-driven SDK plus a configurable generic REST poller
      converts the long tail into configuration. Keep that.

  T8  1:1 PARSER-TO-SOURCE IS THE RIGHT RULE.
      Chronicle enforces it at scale. Adopt explicitly (06 C2).

  T9  PARSER/CONNECTOR LIBRARIES ARE PERMANENT COST.
      Google maintains hundreds and adds continuously. This is a
      standing operating expense, not a project. It argues further
      for depth over breadth in a solo build.

  T10 STELLAR CYBER'S GAP IS OUR ENTIRE THESIS.
      Zero code, data, Kubernetes, AI or business-context
      connectors. They cannot build a permission closure or an
      exposure graph from their catalog. That gap is structural,
      not a backlog item.
```

---

## 6. What we must align with

### 6.1 OCSF — adopt partially

The Open Cybersecurity Schema Framework is the industry's normalized telemetry schema. Enterprises map vendor logs to OCSF via ETL/ELT; AWS exposes a `ParseToOCSF` API; it is the lingua franca for security data exchange. ✔

**Should Overlook adopt OCSF as its normalization schema?**

```
   WHERE OCSF FITS
     event-shaped data — authentication, process activity, network
     activity, findings, vulnerabilities
     → our stage-4 Normalize output for STREAM and PUSH ingress
     → our FINDING and EVENT_SUMMARY fact types

   WHERE OCSF DOES NOT FIT
     OCSF is event-centric. It has no vocabulary for effective
     permission, CAN_ASSUME, permission closure, escalation
     primitives, or bitemporal edge lifecycle.
     → our ENTITY and RELATIONSHIP fact types must remain ours

   DECISION
     Adopt OCSF for the event/finding classes.
     Keep the Overlook entity model and predicate vocabulary for
     the graph.
     Publish the mapping between them.

   WHY IT MATTERS
     - existing community mappings reduce parser work
     - interoperability with AWS Security Lake and OCSF-native tools
     - customers can export Overlook findings into their own lake
     - it is a cheap credibility signal — "we speak the standard"
       for the parts where a standard exists
```

This is a real decision and should be recorded alongside the other contracts in `04 §29`.

### 6.2 Connector manifest changes

Concrete additions to the manifest schema in `03 §4.2`:

```yaml
connector:
  id: aws
  # NEW — state the API surface, not just the vendor (T2)
  api_surface: configuration        # configuration | log_stream | hybrid

  # NEW — industry-standard function model (T4)
  functions: [collect]              # collect | respond
                                    # response capability is a SEPARATE
                                    # manifest with separate credentials

  # RETRACTED 2026-08-13 — tenant_scoped is NOT needed.
  # Overlook is multi-INSTANCE, not multi-tenant: one collector per
  # customer, physical isolation. There is no tenant concept inside
  # the pipeline. See 09-deployment-and-tenancy-model.md §2.

  # NEW — execution placement for multi-Edge deployments (T6)
  # MORE relevant under the MSSP model: Archetype 3 (hybrid) deploys
  # two Edge Collectors per customer, on-prem and cloud.
  execution: { placement: any }     # any | pinned:<collector_id>

  collectors:
    - id: iam.roles
      api_surface: configuration
      # NEW — OCSF mapping where one exists (6.1)
      ocsf_class: null              # entity/relationship: no OCSF class
    - id: cloudtrail
      api_surface: log_stream
      ocsf_class: 3002              # Authentication
```

### 6.3 What we do NOT need to align with

```
  ✕ Connector COUNT. We are not competing on catalog size.
  ✕ Their category taxonomy. Theirs is organised by security-tool
    type because they ingest alerts. Ours is organised by entity
    source because we build a graph. Ours is correct for us.
  ✕ Email, Web Security, DNS Security as top-level categories.
    Those are alert sources. For us, DNS and proxy are inputs to
    Shadow AI detection and network reachability — collectors
    inside the Network domain, not categories.
  ✕ Per-request connector development. Explicitly reject.
```

---

## 7. How many connectors do we actually need

Revised against the feasible options in `07 §5`.

### 7.1 For Option C — escalation primitive engine

```
   ZERO connectors.

   The engine evaluates capability sets. It needs the capability
   schema and a policy parser, not a live connection. Test fixtures
   are enough. This is why Option C is the cheapest credible start.
```

### 7.2 For Option A — the unmanaged AI exposure map

```
   6 connectors + 1 agent

   1  Overlook Agent          local MCP configs, model runtimes,
                              IDE assistants, credential presence
   2  AWS                     IAM only — roles, policies, closure
                              (~6 collectors, not 28)
   3  Microsoft Entra ID      identities, app registrations, SPs
   4  GitHub                  repos, OIDC trusts, secret scanning
   5  Okta or AD              one workforce IdP for canonical keys
   6  Anthropic / OpenAI      org admin API — keys, projects, members

   That is enough to produce the demo in 07 §5 Option A:
   laptop MCP server -> credential -> cloud role -> production data.
```

### 7.3 For Option B — privacy-first identity exposure

```
   8-10 connectors

   AWS (IAM depth), Azure (RBAC), Entra ID, Active Directory,
   AD CS, Okta, GitHub, Kubernetes
   + optionally GCP, and one CMDB for ownership

   This is the doc 02 core. Note it is BloodHound's catalog plus
   Azure and Kubernetes — which is a fair description of what
   Option B competes with.
```

### 7.4 The revision to doc 03

```
   DOC 03 SAYS          118 connectors, 695 collectors
   REALITY              6-10 connectors for anything solo-feasible

   The 118 catalog is not wrong — it is the target architecture for a
   funded team, and the framework, manifest, orchestration bands, rate
   governance and coverage-window design in doc 03 are all correct and
   all still needed at 6 connectors.

   What changes is the TARGET. Doc 03 should be reframed: the framework
   is the deliverable, the catalog is the roadmap, and the first
   milestone is 6 connectors of exceptional depth — not 60 of
   average depth.
```

---

## 8. Open questions

```
  Q1  Adopt OCSF for event/finding classes? (recommended yes)
  Q2  RESOLVED — MSSP with single-tenant per-customer deployments.
      No tenant_scoped field needed. See doc 09.
  Q3  Should doc 03's headline number be restated from 118 to
      "a framework plus 6 deep connectors, with 118 as the map"?
  Q4  Verify Wiz's and XM Cyber's actual connector counts and
      API surfaces before using them in any comparison.
  Q5  Do we ever need Stellar Cyber's alert-source categories at all
      (endpoint, email, web security), or do we consume those only
      via the security-tooling overlay connectors that attach
      properties to existing nodes?
```

---

*End of document.*

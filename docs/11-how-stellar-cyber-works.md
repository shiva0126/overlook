# Overlook — How Stellar Cyber Actually Works

**Version:** 0.1
**Date:** 2026-08-13
**Companion to:** `06-prior-art-secops-stellarcyber.md`, `08-connector-benchmark-and-alignment.md`, `10-appliance-stack-and-engines.md`
**Purpose:** Explain the mechanics of Stellar Cyber's platform — how the sensors, connectors, parsers, engines and data lake actually fit together — and what that means for Overlook.

**Confidence:** ✔ verified from Stellar Cyber's own documentation in this session.

---

## 1. The question

*"They have every connector, parsers, engines, sensors, everything in place. But I don't understand how they work on things."*

Two separate questions hide in that. This document answers both:

1. **Mechanically, how does their machine work?** (§2–§7)
2. **Do they have "everything" — and does that mean we are behind?** (§8)

The short answer to the second: they have everything **for a different machine**. Their completeness is real and it is complete for an XDR. It is not completeness for an exposure graph, and the gap is structural rather than a backlog.

---

## 2. The five components

```
  ┌─────────────┐   ┌─────────────┐   ┌──────────────────────────┐
  │  SENSORS    │   │ CONNECTORS  │   │      DATA PROCESSOR      │
  │             │   │             │   │                          │
  │ deployed    │   │ run INSIDE  │   │  receivers               │
  │ across the  │   │ the DP as   │   │  connector tasks         │
  │ customer's  │   │ TASKS       │   │  detection engines (ML)  │
  │ network,    │   │             │   │  correlation engine (ML) │
  │ hosts,      │   │ actively    │   │  embedded web server     │
  │ cloud       │   │ pull from   │   │  REST API                │
  │             │   │ APIs        │   │                          │
  └──────┬──────┘   └──────┬──────┘   └────────────┬─────────────┘
         │                 │                       │
         └────── INTERFLOW records ────────────────┤
                                                   │
                                        ┌──────────▼──────────┐
                                        │     DATA LAKE       │
                                        │  Interflow records  │
                                        │  organised into     │
                                        │  INDICES            │
                                        └──────────┬──────────┘
                                                   │
                                        ┌──────────▼──────────┐
                                        │   UI / PLAYBOOKS    │
                                        │  alerts, cases,     │
                                        │  response actions   │
                                        └─────────────────────┘
```

### 2.1 Sensors — the customer-side collectors ✔

Three families, all producing the same output format:

```
  SERVER SENSORS  (software agents on hosts)

    Linux Server Sensor
      collects: process info, command execution, file events,
                audit logs, system resource usage
      feeds:    Traffic, Linux Events, Sensor Monitoring indices

    Windows Server Sensor
      runs as:  a Windows service
      collects: hardware, security and system events, Windows
                Firewall and Defender logs, PowerShell activity

    Both convert what they observe to METADATA and forward it
    as Interflow to the Data Processor.

  MODULAR SENSORS  (purpose-built appliances — host + software)

    always:   log ingestion
    optional: network traffic analysis (applications, response times)
              sandbox (malware detonation)
              IDS (intrusion detection)
              vulnerability scanning

    This is a bundle. One box, features toggled on.

  LEGACY (now folded into Modular)
    Network Sensor    traffic monitoring from physical/virtual switches
    Security Sensor   traffic + sandbox + anti-virus + IDS
```

The consolidation from Network/Security Sensor into one **Modular** Sensor with toggles is worth noting: they converged on *one artifact, features enabled by configuration* — the same conclusion `10 §7` reaches for Overlook with one binary and role flags.

### 2.2 Connectors — and the detail that matters ✔

> Connectors actively pull data from external sources (APIs, security tools, threat feeds) and convert it into Interflow format, **running as tasks within the Data Processor itself.**

**Connectors do not run on sensors. They run centrally, on the DP.**

This is the mechanical answer to the user's question, and it explains their whole shape:

```
   TWO INGESTION PATHS, TWO PROCESSING LOCATIONS

   PATH A — SENSOR DATA (push)
     host/network → SENSOR → filter + normalize AT THE EDGE
                           → Interflow → RECEIVER on the DP

     Normalization happens at the sensor. This is why they can
     claim "filtering before data leaves the source" — it applies
     to the high-volume path (traffic, host events), which is
     exactly where volume reduction matters.

   PATH B — API DATA (pull)
     SaaS/cloud API ← CONNECTOR TASK running ON THE DP
                    → Interflow → straight into the lake

     No sensor involved. The DP reaches out directly.
     Nothing is filtered "at the source" here, because the
     source is someone else's API.
```

So "edge filtering" is true for Path A and inapplicable to Path B. Both paths converge on the same record format at the DP.

### 2.3 Receivers

Passive listeners on the DP that accept sensor-pushed Interflow. The counterpart to connectors: receivers wait, connectors fetch. Same distinction as our PUSH/STREAM versus PULL ingress classes (`04 §2`).

### 2.4 Data Processor — the central hub ✔

Everything of consequence runs here:

```
  embedded web server + REST API   the UI downloads JavaScript once,
                                   then talks pure REST
  receivers                        accept sensor Interflow
  connector tasks                  pull from APIs
  detection engines                ML over the lake
  correlation engine               ML grouping alerts into cases
  cloud-native microservices       containerised, scalable, clusterable
```

Deployable all-in-one, distributed, or clustered (`06 §3.1`).

### 2.5 Data Lake ✔

Interflow records organised into **indices** for query and analysis. Multi-tenant, with per-tenant storage options and retention.

---

## 3. Interflow — their normalization contract

The single most important concept in their architecture, and their equivalent of Chronicle's UDM and our Security Fact.

```
  WHAT IT IS ✔
    "extensible JSON objects representing normalized security events"
    a universal data representation
    an INTERFLOW DICTIONARY defines the fields, for threat hunting
      and enrichment
    records are ENRICHED THROUGHOUT THEIR LIFECYCLE — geo, threat
      intel, context are added as the record moves
    designed to future-proof: new sources and new ML capabilities
      extend the record rather than breaking it
```

### 3.1 How a record is built ✔

Their stated pipeline:

```
   1. INGEST        from sensors (push) or connectors (pull)
   2. FILTER        at the source for sensor data; fine-grained
                    forwarder rules so only needed data is retained
   3. NORMALIZE     standardize disparate log formats and schemas
                    into one shape
   4. TRANSFORM     convert to metadata-rich Interflow records —
                    "significantly reducing size while retaining
                    essential information"
   5. ENRICH        geo, threat intelligence, context
   6. CORRELATE     relate records and alerts
   7. ROUTE/STORE   into the appropriate lake index
```

**Compare to ours** (`04 §3`): receive → identify → parse → normalize → enrich → resolve → derive → fact build → privacy gate → sign → queue.

Steps 1–5 are nearly identical. Everything after diverges completely, and §6 explains why.

### 3.2 The three normalization contracts side by side

| | Interflow | UDM (Chronicle) | Security Fact (Overlook) |
|---|---|---|---|
| Unit | normalized **event** | normalized **event** | **assertion about a relationship** |
| Shape | extensible JSON | fixed schema sections | typed fact, 5 classes |
| Lifecycle | enriched as it travels | written once at ingest | **merged across observations, retracted, bitemporal** |
| Destination | lake index | event + entity store | **graph** |
| Volume relationship | reduced, still per-event | one record per event | **14,882 observations → 1 fact** |
| Privacy role | none | none | **the boundary itself** |

The row that matters is **lifecycle**. Interflow records accumulate: one event, one record, enriched along the way. Security Facts *collapse*: thousands of observations merge into one fact whose `observation_count` increments and whose `last_seen` moves. That difference is the 200,000:1 reduction, and it is why our output is a graph rather than a lake.

---

## 4. The engine chain

Where their processing actually happens, in order:

```
  ┌────────────────────────────────────────────────────────────┐
  │ AT THE SENSOR                                              │
  │   traffic + application filters                            │
  │   metadata extraction (packets/events → metadata)          │
  │   Interflow construction                                   │
  │   → this is the volume-reduction stage                     │
  └───────────────────────┬────────────────────────────────────┘
                          │  Interflow
  ┌───────────────────────▼────────────────────────────────────┐
  │ AT THE DATA PROCESSOR                                      │
  │                                                            │
  │  RECEIVERS ──┐                                             │
  │              ├──► INTERFLOW NORMALIZATION ENGINE           │
  │  CONNECTORS ─┘     standardizes formats and schemas        │
  │                    from disparate sources                  │
  │                          │                                 │
  │                          ▼                                 │
  │                    ENRICHMENT (geo, TI, context)           │
  │                          │                                 │
  │                          ▼                                 │
  │                    DATA LAKE INDICES                       │
  │                          │                                 │
  │                          ▼                                 │
  │                    ML DETECTION ENGINES                    │
  │                    examine lake records, detect anomalies, │
  │                    generate ALERTS mapped to kill-chain    │
  │                    stages, tactics and techniques          │
  │                          │                                 │
  │                          ▼                                 │
  │                    ALERTS INDEX                            │
  │                          │                                 │
  │                          ▼                                 │
  │                    ML CORRELATION ENGINE                   │
  │                    "correlates disparate alerts into a     │
  │                     coalesced CASE" with dynamic severity  │
  │                          │                                 │
  │                          ▼                                 │
  │                    CASES → UI → PLAYBOOKS → RESPONSE       │
  └────────────────────────────────────────────────────────────┘
```

### 4.1 The two ML stages are distinct

Worth separating, because they are often conflated:

```
  ML STAGE 1 — DETECTION
    input   individual Interflow records in the lake
    asks    "is this anomalous?"
    output  an ALERT, tagged with kill-chain stage, tactic, technique

  ML STAGE 2 — CORRELATION
    input   many alerts
    asks    "do these belong to the same attack?"
    output  a CASE, with a dynamic severity score

  Multi-ML applies these SEPARATELY PER TENANT, so one customer's
  baseline never contaminates another's. ✔
```

Their correlation is **temporal and behavioural** — these alerts happened near each other and share artifacts, therefore they are one incident.

Ours is **structural** — these entities are connected by permissions, therefore a path exists. No time window involved.

---

## 5. So how do they "work on things" — the walkthrough

An end-to-end trace, to make the mechanics concrete.

```
  T0   A Windows server runs an unusual PowerShell command.

  T1   WINDOWS SERVER SENSOR observes the event.
       Extracts metadata: process, command, user, parent process,
       host. Applies filters. Constructs an Interflow record.
       Sends it to the DP.

  T2   RECEIVER on the DP accepts it.

  T3   INTERFLOW NORMALIZATION ENGINE standardizes it into the
       common schema — the same shape a Fortinet syslog line or an
       Okta API response would become.

  T4   ENRICHMENT adds geo for any external IP, threat-intel tags,
       and any known context.

  T5   Record lands in a DATA LAKE INDEX.

  T6   ML DETECTION examines it against learned baselines for this
       tenant. It is anomalous → generates an ALERT, tagged to a
       kill-chain stage.

  T7   Meanwhile, a firewall CONNECTOR pulled a log showing an
       outbound connection from the same host, and an EDR connector
       pulled a detection. Both became Interflow, both became alerts.

  T8   ML CORRELATION notices the three alerts share the host
       artifact and fall in a time window. It coalesces them into
       one CASE with a computed severity.

  T9   Analyst opens the case in the UI. Investigates across the
       lake. Runs a PLAYBOOK: a firewall connector's RESPOND
       function pushes a block.
```

**That is the whole machine.** Sensors and connectors produce a common record; the lake stores it; ML finds anomalies; ML groups anomalies; humans and playbooks act.

It is a well-built, coherent XDR. Every piece serves the goal of *detecting an attack in progress and responding to it.*

---

## 6. The mechanical difference from Overlook

Now the comparison is exact rather than hand-waving.

```
   STELLAR CYBER                      OVERLOOK
   ─────────────────────────────      ─────────────────────────────
   INPUT   events, alerts, traffic    configuration, permissions,
                                      policies, relationships

   UNIT    the Interflow record       the Security Fact
           one event → one record     N observations → one fact

   STORE   data lake, indices         graph, adjacency + closure
           optimised for SEARCH       optimised for TRAVERSAL

   ENGINE  ML anomaly detection       permission closure
           ML alert correlation       escalation matching
                                      reverse BFS path finding

   TIME    time windows are central   time is a LIFETIME
           "these happened together"  "this has existed 41 days"

   OUTPUT  alerts → cases             edges → paths → chokepoints

   ASKS    "is something happening?"  "what is possible?"

   ACTS    contain the incident       remove the permission
```

### 6.1 Why they cannot become us by adding connectors

The instinct — *"they have all the connectors, so they could just do what we do"* — does not hold, for four mechanical reasons:

```
  1. THEIR STORE IS WRONG FOR IT
     A lake with indices answers "find records matching X."
     Attack paths need "traverse from X to Y through N hops."
     Recursive traversal over an index-based lake is not a feature
     you add; it is a different storage engine.

  2. THEIR RECORD IS WRONG FOR IT
     Interflow is an event. A permission is not an event. There is
     no "this happened" for "svc-devops-ai CAN assume DevOpsAdmin" —
     it is a standing condition derived from evaluating policy
     documents, and it has no timestamp in the event sense.

  3. THEY HAVE NO PERMISSION CLOSURE ENGINE
     Their cloud connectors are CloudTrail, CloudWatch, Event Hub,
     Cloud Audit — all LOG streams (08 §3.1). Reading policy
     documents and evaluating precedence, boundaries, SCPs and
     conditions is a different engine that does not exist in an XDR.

  4. THEY HAVE NO ENTITY RESOLUTION FOR THIS PURPOSE
     They correlate on shared ARTIFACTS within a time window —
     same IP, same host, same user string. We must resolve one
     person across five identifier systems into one canonical node
     that persists for years (01 §8). Different problem, different
     engine.
```

Adding a GitHub connector to Stellar Cyber gives them GitHub *events*. It does not give them "this repository's OIDC trust lets any workflow assume a production role." That sentence requires reading a trust policy, evaluating a condition, and creating a durable edge — three things their machine does not do.

---

## 7. What we should copy — mechanically

```
  M1  ONE ARTIFACT, FEATURES BY CONFIGURATION
      They folded Network Sensor and Security Sensor into one
      Modular Sensor with toggles. We reached the same conclusion
      independently (10 §7: one binary, role flags). Confirmed.

  M2  A NAMED, DOCUMENTED RECORD FORMAT WITH A DICTIONARY
      "Interflow" is a product asset, not just an internal schema.
      It has a published field dictionary users write hunts against.
      Our Security Fact schema should be named, documented and
      published the same way — and for us it doubles as the privacy
      proof (01 §5.6).

  M3  EXTENSIBLE RECORDS THAT DO NOT BREAK ON NEW SOURCES
      Interflow is explicitly designed so new sources extend rather
      than break it. Our forward-compatibility rule — unknown fields
      preserved, not rejected (01 §5.6) — is the same instinct.
      Keep it.

  M4  ENRICHMENT AS A PIPELINE STAGE, NOT A QUERY-TIME JOIN
      They enrich records as they travel. We do the same at stage 5
      (04 §13). Cheaper than joining at query time and it survives
      the source going away.

  M5  RECEIVERS vs CONNECTORS AS A FIRST-CLASS DISTINCTION
      Passive listeners and active pullers are different components
      with different failure modes. Our four ingress classes
      (04 §2) are the more precise version of the same idea.

  M6  FILTER AT THE HIGHEST-VOLUME POINT
      Their sensors filter before sending. Our flow aggregation at
      receive (04 §6.2) is the same principle. Whoever sees the
      volume first should reduce it.
```

## 7.1 What we must not copy

```
  X1  CONNECTORS RUNNING CENTRALLY
      For them the DP is customer-side, so it is fine. For us,
      connectors MUST run on the appliance — they read policy
      documents and credentials, which is precisely the data that
      may not travel.

  X2  A DATA LAKE AS THE PRIMARY STORE
      Storing every record is their business model and their cost
      base. Ours is to discard events and keep relationships
      (01 §1.2). A lake would be the single most expensive wrong
      turn available to us.

  X3  ML AS THE PRIMARY DETECTION MECHANISM
      Their ML answers "is this anomalous?" Our findings are
      structural and deterministic: a permission either creates a
      path or it does not. Deterministic findings are explainable,
      reproducible and arguable with a cloud engineer. Do not
      reach for ML where closure gives a provable answer.

  X4  BUILDING CONNECTORS PER CUSTOMER REQUEST
      Already rejected (08 §2.3).
```

---

## 8. "They have everything" — the honest answer

They do. For their machine.

```
  WHAT THEY HAVE, COMPLETE
    ✓ 86+ connectors across 13 categories
    ✓ sensors for host, network, cloud, container
    ✓ a normalization engine and a documented record format
    ✓ a data lake with per-tenant indices and retention
    ✓ ML detection and ML correlation
    ✓ case management, playbooks, response actions
    ✓ multi-tenancy from inception
    ✓ three deployment topologies

  WHAT THEY DO NOT HAVE — and it is not a backlog
    ✗ a permission closure engine
    ✗ an escalation primitive catalog
    ✗ a graph store with transitive closure
    ✗ attack path computation or chokepoint analysis
    ✗ entity resolution across identifier systems, persisted
    ✗ configuration-API connectors (theirs are log streams)
    ✗ any connector for code, Kubernetes, data platforms, AI
      or business context (08 §3)
    ✗ a privacy boundary — their answer is "self-host the DP"
```

**The correct way to hold this:** Stellar Cyber finished building an XDR. We are not behind them on that race, because we are not in it. Every component they have that we lack is a component we decided not to build (`01 §1.4`), and every component we need that they lack is one an XDR has no reason to build.

Where their completeness *should* worry us is operational, not architectural: they have a mature multi-tenant platform, three deployment topologies, and a working MSSP business. That is the operational maturity we will have to reach if the model in `09` scales past a handful of customers. It just is not the same as being ahead on the product.

---

## 9. What this changes

```
  Nothing architecturally — every mechanic examined here confirms a
  decision already taken in docs 01, 04, 09 and 10.

  Two things to adopt:
    A1  Name and publish the Security Fact schema as a product asset
        with a field dictionary, the way Interflow is. (M2)
    A2  State explicitly in 10 §2 that connectors run ON the
        appliance and why — the contrast with Stellar Cyber's
        DP-resident connectors makes the reasoning clearer than
        asserting it alone. (X1)

  One thing to watch:
    W1  Their operational maturity — multi-tenancy, topologies,
        MSSP tooling — is real and took years. Our 09 model needs
        the fleet plane earlier than feels necessary (09 §7).
```

---

*End of document.*

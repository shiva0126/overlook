# Overlook — Ingestion and Sources

**Version:** 0.1
**Date:** 2026-08-17
**Companion to:** `04-data-flow-to-security-facts.md` (the ten pipeline stages), `engines/01`–`02` (mechanics), `collectors/00` (collector anatomy)
**Status:** Architecture. No implementation.

---

## What this document is, and is not

`04` specifies the **pipeline** — ten stages from receive to signed fact. `engines/02` specifies the **mechanics** of receiving. `collectors/00` specifies how a **collector** behaves.

None of them answers the question that comes before all of that: **what are our sources actually worth, and how does data physically get in?**

```
  THIS DOCUMENT       what we ingest · what it is worth · how it
                      enters · what ingestion guarantees

  NOT THIS DOCUMENT   the ten stages (04) · receive internals
                      (engines/02) · per-collector specs (collectors/)
```

---

# PART I — WHAT OUR SOURCES ARE

## 1. "Main source" has three answers

Asked what Overlook's main source is, there are three defensible answers and they are completely different sources. Conflating them is the trap in `08 §3.1`.

```
  BY VOLUME              firewall syslog + NetFlow
                         ~2.1 TB/day
                         → ~90 edges/day
                         → 95% of arriving bytes, 0.003% of the graph

  BY VALUE               cloud IAM configuration
                         ~18 GB per full enumeration
                         → ~310,000 edges
                         → the permission closure, and therefore
                           every attack path

  BY DIFFERENTIATION     the Overlook Agent
                         ~102 MB/day across 8,500 endpoints
                         → MCP configs, local models, IDE assistants
                         → the only source in the catalog that no
                           competitor collects at all
```

**If the question is "which source do we build the product around," the answer is cloud IAM configuration.** Everything else annotates the graph that IAM creates. Without it, E7 and E8 idle and there are no paths — the point made concretely in `collectors/03 §5`.

## 2. Value density — the ranking that matters

Edges produced per megabyte ingested. Config sources measured per full enumeration; stream sources per day.

```
  SOURCE                          INGESTED    EDGES      EDGES/MB
  ────────────────────────────    ────────    ───────    ────────
  firewall rulebase (config)         40 MB     14,000     350.0
  Active Directory (full sweep)     2.8 GB    416,000     148.6
  Entra ID (full)                   400 MB     40,000     100.0
  cloud IAM (config, all clouds)     18 GB    310,000      17.2
  Overlook Agent                    102 MB      2,040      20.0
  Kubernetes RBAC                    60 MB        900      15.0
  GitHub / repos                    180 MB      2,100      11.7
  DLP classification                1.1 GB      4,100       3.7
  CrowdStrike / EDR                  40 GB     11,600       0.3
  VMware / databases / shares        12 GB      3,400       0.3
  ────────────────────────────    ────────    ───────    ────────
  NetFlow                           900 GB         50    0.00006
  firewall traffic syslog          1.24 TB         40    0.00003
```

```
  THE SPREAD IS ROUGHLY 10,000,000 : 1

  A firewall's RULEBASE and its TRAFFIC LOG come from the same
  device, through the same connector, and differ by seven orders
  of magnitude in value per byte.

  That single fact is why our connector catalog looks nothing like
  an XDR's, and why "supports FortiGate" is a meaningless claim
  without saying which collector.
```

### 2.1 Why the low-density sources are still collected

Not everything is judged on density.

```
  NETWORK TELEMETRY (flow, traffic logs)
    produces OBSERVED reachability, to be compared against
    CONFIGURED reachability from the rulebase. The DISAGREEMENT is
    the value:
      configured, never observed  → an unused rule, attack surface
                                    with no business justification
      observed, not configured    → a shadowed rule or a bypass
    Neither is visible from config alone.

  EDR / DLP / SCANNERS
    overlays. They create few nodes and attach the properties that
    make everything else scoreable — PROTECTS edges, data
    classification, vulnerability properties. Without DLP,
    prod-payments-db is "a database" rather than a crown jewel,
    and the path engine has nothing to compute toward.

  AUDIT TRAILS
    low density, but they carry the USED state for CIEM and the
    change attribution that turns "this changed at 14:22" into
    "changed at 14:22 by priya.s via Terraform run 4471".
```

## 3. What we deliberately do not ingest

```
  ✕ FULL PROCESS TREES from our own agent
      8,500 hosts × continuous capture ≈ 2.4 BILLION records/day.
      Taken from the EDR API instead — better coverage, an already
      approved kernel driver, no new agent risk. (01 §12.1)

  ✕ RAW EVENTS RETAINED LONG-TERM
      we read logs, extract relationships, and discard. Storing
      them is a SIEM's business model and its cost base. (01 §1.2)

  ✕ SECRET VALUES
      presence, type and location only. Reading a credential to
      verify it makes us the risk we are reporting.
      (collectors/03 §4)

  ✕ PROMPT AND RESPONSE CONTENT
      metadata by default. Inspection is opt-in, local, and the
      content is discarded after classification. (01 §26.1)

  ✕ PACKET CAPTURE
      that is an NDR. Flow metadata is sufficient for reachability.

  ✕ ANYTHING WE CAN INGEST AS A RESULT
      vulnerability scanning, data classification, EDR telemetry —
      where the customer already runs a tool, consume its output
      rather than duplicating a control they paid for.
```

---

# PART II — HOW DATA PHYSICALLY ENTERS

## 4. The four ingress classes

They differ in one property — **recoverability** — and that single property determines the durability contract, the failure behaviour and whether the source can ever drive retraction.

```
  PULL      the appliance calls out and fetches
            connector API polling · LDAP · database queries
            complete objects, self-describing, RE-FETCHABLE
            volume LOW · value density VERY HIGH
            if lost: fetch again. Costs API calls, loses nothing.

  PUSH      the source calls in with an event
            webhooks, event grids
            single events, NOT re-fetchable, replayable by an attacker
            volume MEDIUM · value density HIGH
            if lost: GONE FOREVER

  STREAM    the source fires continuously
            syslog, NetFlow, IPFIX, sFlow
            fragmentary, lossy transport, no re-fetch
            volume VERY HIGH · value density VERY LOW
            if lost: gone, but individually near-worthless

  AGENT     our own software reports in
            Overlook Agent, AI Gateway
            batched, buffered AT SOURCE, re-deliverable
            volume MEDIUM · value density HIGH
            if lost: the agent re-sends from its own buffer
```

## 5. The durability contract

```
  PULL      CURSOR, no journal fsync required
            advance the cursor only after the batch is safely
            downstream. Crash mid-fetch → refetch.

  PUSH      JOURNAL + FSYNC BEFORE RETURNING 200
            returning 200 before the record is durable is the
            classic data-loss bug — the sender will never send it
            again. Plus replay protection: signature, delivery-ID
            seen-set, timestamp window.

  STREAM    AGGREGATE IN MEMORY AT RECEIVE, journal the aggregate.
            Per-record fsync at 4.1 billion records/day is not
            physically possible and would be pointless — the
            individual record is not what we keep.

  AGENT     journal + fsync, THEN ack. The agent prunes only on ack.
            Lightest contract, because retry lives at the source.
```

### 5.1 The number this produces

```
  2.2 TB/day    arrives
    187 MB/day  journaled
    4.2 MB/day  gets the expensive fsync treatment
     12 MB/day  leaves the environment (Mode 2)

  89% of arriving volume never touches disk in any form.
  The fsync budget is spent precisely where data is unrecoverable,
  and nowhere else.
```

## 6. Physical topology

What listens, what dials out, and what credential each needs.

```
  ┌─ INBOUND — the appliance LISTENS ────────────────────────────┐
  │                                                              │
  │  syslog/udp:514      legacy. Lossy. Recommend migration.     │
  │  syslog/tcp:6514     TLS. Preferred.                         │
  │  netflow/udp:2055    NetFlow v9 / IPFIX / sFlow              │
  │  https:443           webhook receiver, per-source path,       │
  │                      HMAC or mTLS verified                    │
  │  mtls:8443           agent gateway. Agent-initiated.          │
  │  https:8443          Controller UI + resolve API              │
  │                      (customer network only)                  │
  │                                                              │
  │  AUTH: source IP allow-list · TLS client certs · HMAC        │
  └──────────────────────────────────────────────────────────────┘

  ┌─ OUTBOUND — the appliance DIALS OUT ─────────────────────────┐
  │                                                              │
  │  cloud APIs           IRSA / Managed Identity / WIF          │
  │                       → NO STORED CREDENTIAL                  │
  │  LDAPS:636            dedicated read-only service account     │
  │  vendor APIs          token or app credential, from the       │
  │                       credential broker, 5-minute TTL         │
  │  databases            read-only account, catalog views only   │
  │  mtls:443 → console   Mode 2 only. Outbound ONLY.            │
  │                                                              │
  │  NO INBOUND FROM OVERLOOK, EVER. The console never connects  │
  │  to the appliance. No firewall rule is required for us.      │
  └──────────────────────────────────────────────────────────────┘
```

**"You do not need to open a port for us" is a procurement feature.** It removes a firewall change request and a security-architecture review that can take months in an organisation like Meridian.

## 7. Two regimes, one pipeline

```
  INITIAL LOAD — first 72 hours       STEADY STATE — every day after
  ─────────────────────────────       ─────────────────────────────
  full enumeration of everything      delta collection on cursors
  AD: 24,000 objects, 720,000 ACEs    AD: uSNChanged delta, ~400
  AWS: 21,000 policies                AWS: ~40 changed policies
  closure computed from scratch       incremental closure on change
  resolution from scratch             resolution for new entities only
  ~14 hours of processing             ~8 minutes
  review queue fills                  2-6 items/day
  ~180,000 facts emitted              ~2,900 facts/day
```

Sizing must satisfy both: **the initial load defines peak memory, the steady state defines latency.** A design tuned only on the second is surprised by the first.

---

# PART III — WHAT INGESTION GUARANTEES

## 8. The contract

```
  ✓ nothing is acknowledged before it is durable   (PUSH, AGENT)
  ✓ nothing is processed before it is acknowledged
  ✓ provenance travels with every record
  ✓ no record is ever silently discarded — parse failures are
    QUARANTINED with a retained sample
  ✓ shedding under pressure is by declared priority class, never
    uniform
  ✓ replay produces byte-identical output given identical content
  ✓ a coverage window is emitted ONLY on a complete enumeration
  ✓ a stream NEVER emits a coverage window, and therefore can only
    add to the graph, never retract
```

## 9. Priority under pressure

Backpressure propagates to receive, which sheds **by class, never uniformly**.

```
  P0  agent telemetry, AI gateway facts
  P1  cloud audit trails
  P2  identity events
  P3  firewall / flow
  P4  application logs
  P5  everything else

  Losing 100% of verbose printer syslog is far better than losing
  10% of everything.
```

## 10. The journal's second job

The one that matters more than durability.

```
  Because the privacy architecture forbids asking a customer to
  send us their logs (03 §11.5), the ingest journal is the ONLY
  reproduction mechanism.

    fix the content → replay the journal locally, scoped by source
    and time window → diff against what was originally produced
    → apply

  This is why journal replay is a first-class Controller feature
  (05 §23) rather than an internal tool, and why PULL sources are
  journaled at all despite being re-fetchable.
```

---

# PART IV — HOW THIS DIFFERS

## 11. Against Stellar Cyber and Chronicle

Now that both are studied (`06`, `11`), the contrast is precise.

| | Stellar Cyber | Google SecOps | Overlook |
|---|---|---|---|
| Where connectors run | **inside the Data Processor** | Google cloud | **on the appliance, always** |
| Where normalization happens | at the sensor (push), at the DP (pull) | Google cloud | one place, after the journal |
| Customer-side component | sensor — filters and normalizes | forwarder — **ships raw** | Edge Node — **full analysis** |
| Unit produced | Interflow record, **one per event** | UDM event, one per event | Security Fact, **one per relationship** |
| Lifecycle | accumulates, enriched as it travels | written once at ingest | **collapses** — N observations → 1 fact |
| Destination | data lake, indices | event + entity store | graph |
| Reduction | filtering, ~10:1 | none | abstraction, **200,000:1** |
| Can vendor read it | n/a (self-hosted) | **yes** | **no, by construction** |

### 11.1 The row that matters

```
  INTERFLOW / UDM     one record per event, enriched along the way
  SECURITY FACT       14,882 observations of one relationship
                      collapse into ONE fact whose observation_count
                      increments and whose last_seen moves

  That is the 200,000:1 reduction, and it is why our output is a
  graph rather than a lake.
```

### 11.2 And why our connectors cannot run centrally

Stellar Cyber's connectors run on the Data Processor, which is fine for them — the DP is customer-side.

**Ours must run on the appliance**, because they read policy documents, ACLs, trust policies and credential metadata. That is precisely the data that may not travel. A connector running anywhere else would defeat the architecture it exists inside.

---

# PART V — EXAMPLE: MERIDIAN, ONE HOUR

```
  09:00-10:00, EDGE-DC1

  STREAM — the flood
    4 firewalls → syslog/tcp-tls:6514
      129 million events this hour ≈ 52 GB
      → E1 aggregator, 4 windows of 15 minutes
      → 5,000 aggregate records journaled
      → the 129 million individual events were NEVER WRITTEN TO
        DISK. They existed in memory for at most 15 minutes.

    6 core switches → netflow/udp:2055
      171 million records ≈ 37 GB
      → 7,500 aggregates journaled

    combined: 89 GB arrived, ~3 MB journaled

  AGENT — the quiet path
    ~2,100 of 8,500 endpoints reported (4h cadence, jittered)
    ~4.2 MB
    → journaled + fsync, THEN acked; each agent prunes on ack
    → at 09:14 the appliance is briefly loaded and returns a
      slow-down hint. Agents extend their batch interval rather
      than dropping. Nothing is lost.

  PUSH — GitHub webhooks
    41 deliveries
      HMAC verified BEFORE anything else
      2 duplicate delivery IDs from a GitHub retry
        → replay guard returns 200, does not journal twice
      39 journaled + fsync, then 200
      one fsync takes 340 ms under load; we hold the connection
      and ack late rather than acking early

  PULL — connector fetches
    AWS IAM delta across 41 accounts, ~180 MB
      → journaled WITHOUT fsync (re-fetchable)
      → cursors advanced only after the batch reached E4 safely
      → at 09:47 a worker crashes mid-account. The cursor was
        never advanced, so the next cycle refetches from the last
        good point. Cost: 40 seconds of API calls.

  HOUR TOTAL
    arrived     ~89.2 GB
    journaled   ~187 MB
    fsync'd     ~4.2 MB
```

**89 GB arrived and 4.2 MB got the expensive durability treatment.** That ratio is the ingestion design in one number.

---

## Open questions

```
  Q1  UDP syslog loss estimation — we display estimated loss from
      sequence gaps and receiver drop counters. Is that estimate
      good enough to put a number on, or should it be a
      qualitative warning?

  Q2  Aggregation window is 15 minutes. Shorter means better
      temporal fidelity and more records; longer means more
      reduction and coarser last_seen. Never validated against
      real data.

  Q3  Do we ingest a customer's existing message bus (Kafka /
      Event Hubs / Pub-Sub) as a PULL or a STREAM class? It is
      re-readable by offset, which makes it PULL-like, but
      continuous, which makes it STREAM-like. The durability
      contract differs.

  Q4  Initial load takes ~14 hours. Should it be resumable across
      an appliance restart, or is restarting from scratch
      acceptable given it happens once?
```

---

*End of document.*

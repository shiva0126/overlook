# Overlook — The Edge Collector, Component by Component

**Version:** 2.0
**Date:** 2026-08-18
**Parent:** `../LLD-edge-collector-v1.0.md`
**Status:** Architecture. No implementation.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary and takes
> precedence over this document and over
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1`, which it supersedes
> for collector internals. Where this series differs, it is recorded as an
> escalation in [13-escalations.md](13-escalations.md), not resolved here.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. What this series is

The LLD (88 sections) specifies the collector. This series explains **how each
component works and how it fails** — the reasoning underneath the specification,
at the level someone writing the code needs.

```
  LLD §1–88               the specification. Authoritative.
  THIS SERIES             mechanics, budgets, failure modes, worked examples
  13-escalations.md       the four places we think the LLD needs a decision
```

Nothing here overrides the LLD. Where this series proposes something the LLD
does not state, it is marked **PROPOSED** and carries its reasoning.

---

## 2. The three planes

LLD §4 separates the collector into three planes. This is the most useful
structural idea in the document, because the planes have different volumes,
different availability requirements and different security postures.

```
  DATA PLANE          high volume, must never block, must never lose
                      connector → gateway → NATS → parse → normalize →
                      enrich → facts → privacy → forward
                      LLD §4.1 · docs 01–09

  CONTROL PLANE       low volume, human-facing, must be available even
                      when the data plane is degraded
                      local API · connector manager · vault · certs ·
                      health · parser manager · updater
                      LLD §4.2 · doc 10

  RESPONSE PLANE      rare, high consequence, must be provably authorized
                      signed command → validate → execute → audit
                      LLD §4.3 · doc 11
```

**A failure in one plane must not disable the others.** If the data plane is
saturated, the operator must still be able to reach the UI to see why. If the
control plane is restarting, collection must continue. If the response plane is
compromised, it must not become a path into collection.

---

## 3. Process architecture — a modular monolith

LLD §5 is explicit, and it is the right call for V1:

```
  Linux
  ├── overlook-collector      ONE Go binary, all modules (LLD §6)
  ├── nats-server             JetStream
  └── overlook-updater        signed upgrades

  "Do not create separate operating-system processes for every module
   during the initial versions."                            — LLD §5
```

```
  WHAT THIS BUYS
    one binary to build, sign, ship and roll back
    in-process channel handoffs instead of broker round trips
    one config file, one log set, one health endpoint
    SQLite becomes correct — one writer, no contention (LLD §41)
    a support engineer can reason about it at 3 a.m.

  WHAT IT COSTS, AND MUST BE MITIGATED IN CODE
    a panic in any module kills the whole collector.
      → recover() at every worker-pool boundary
      → a parser panic dead-letters the record (LLD §20) and the
        worker continues. It does not propagate.
    one module can starve another for CPU.
      → the worker-pool caps in LLD §17 are the mechanism; §4 below
        is the budget they should be set from.
    no independent restart.
      → hot-reload paths matter more, not less. See 03 §4.
```

---

## 4. The resource budget

LLD §71 sets the sizing. It does not divide it, and worker pools (LLD §17) have
to be configured from *something*. This is that something.

**Edge Large — 12 vCPU / 64 GB / 1 TB, at 10,000 EPS.**

### 4.1 CPU

```
  MODULE                        vCPU   LLD REF
  ─────────────────────────────────────────────────────────
  Ingestion Manager              1.5   §6, §12
  nats-server                    1.0   §14–16
  Parser Workers                 3.5   §17  min 2 max 16
  Normalization Workers          1.0   §17  min 2 max 12
  Enrichment Workers             1.0   §17  min 2 max 8
  Security Fact Workers          1.0   §17  min 2 max 8
  Privacy Workers                0.25  §26
  Forwarder Workers              0.5   §17  min 2 max 8
  API Server + Control Plane     0.25  §46
  Agent Gateway                  0.25  §55
  ─────────────────────────────────────────────────────────
  ALLOCATED                     10.25
  OS + RESERVE                   1.75   burst headroom
```

**Parsing is the largest allocation because it is the only stage that touches
every byte of every event.** Every stage after it operates on less data. If
parsing is not the biggest number, something upstream is doing work it should
not.

### 4.2 Memory

```
  MODULE                         RAM   HOLDS
  ─────────────────────────────────────────────────────────
  Ingestion Manager             2 GB   connection buffers, TLS
  nats-server                   6 GB   in-flight + stream indexes
  Parser Workers                8 GB   parser registry, batches
  Normalization Workers         4 GB   schema maps, in-flight
  Enrichment Workers           14 GB   THE CACHES — see §4.4
  Security Fact Workers        12 GB   aggregation + dedup windows
  Privacy Workers               1 GB   policy, key material
  Forwarder Workers             2 GB   batch buffers
  Control + Agent Gateway       1 GB
  ─────────────────────────────────────────────────────────
  ALLOCATED                    50 GB
  OS + page cache              14 GB
```

Enrichment caches and fact windows are 52% of memory and both are *windows over
recent data*. **Both must be bounded by configuration, not by available RAM**, or
the collector's memory becomes a function of the customer's estate size and
LLD §71 stops being a sizing table.

### 4.3 Disk — and the number that governs everything

```
  ALLOCATION                    SIZE   LLD REF
  ─────────────────────────────────────────────────────────
  OS + binaries + logs          40 GB  §66, §70 (logs 30 d)
  NATS OVERLOOK_RAW            420 GB  §15, §69 queue.max_disk_gb
  NATS OVERLOOK_FORWARD         60 GB  §15, §70 (facts 72 h)
  Dead letter                   30 GB  §70 (7 days)
  SQLite state                  40 GB  §41
  Encrypted spool              200 GB  §29, §35
  ─────────────────────────────────────────────────────────
  ALLOCATED                    790 GB
  FREE / RESERVE               234 GB  a hard ceiling needs real slack
```

```
  RAW RETENTION IS AN SLA, NOT A SETTING.

   5,000 EPS × ~1 KB  =  5 MB/s  =  18 GB/hour
  10,000 EPS × ~1 KB  = 10 MB/s  =  36 GB/hour

  420 GB of RAW therefore buys

    at  5,000 EPS   ~23 hours   ← LLD §69 raw_hours: 24 is reachable
    at 10,000 EPS   ~11.6 hours ← it is NOT. See escalation ESC-2.

  STATE IT AS A COMMITMENT:
  "this collector tolerates an N-hour processing outage with zero loss."
```

**⚠ This allocation assumes ESC-1 is accepted.** LLD §15/§16 also persist
PARSED, NORMALIZED and ENRICHED to disk, which at 6 hours adds ~842 GB and does
not fit. See [13-escalations.md](13-escalations.md).

### 4.4 The pressure levels these numbers feed

LLD §37 defines four levels. The thresholds are only meaningful against a budget.

```
  GREEN    queue < 50% · CPU < 70% · disk < 70%      normal
  YELLOW   queue 50–75%                              scale workers up
  ORANGE   queue 75–90%                              throttle low priority
  RED      queue > 90%                               protect P0/P1 only

  and the priority ladder that governs shedding (LLD §38)

  P0  critical incident / response telemetry    NEVER discard
  P1  security alerts                           NEVER discard
  P2  configuration / posture findings          buffer
  P3  inventory / informational                 throttle
  P4  debug / verbose                           sample or discard
```

Every ingress class maps onto this differently, and the mapping is where data is
actually lost or saved — see `01 §4`.

---

## 5. The pipeline, and where each document sits

```
  CUSTOMER ENVIRONMENT
  ┌────────────────────────────────────────────────────────────────┐
  │                    overlook-collector                          │
  │                                                                │
  │   CONNECTORS   AWS · Azure · GCP · FortiGate · EDR · GitHub ·  │
  │                DB · Syslog · REST · Webhook · Agent   (LLD §12)│
  │                            │                                   │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ INGESTION GATEWAY                          doc 01      │  │
  │   │ auth · validation · rate limit · flow control          │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ NATS JETSTREAM  OVERLOOK_RAW               doc 02      │  │
  │   │ file-backed · fsync before ack                         │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ PARSER ENGINE                              doc 03      │  │
  │   │ detect · select · parse · dead-letter                  │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ NORMALIZATION ENGINE                       doc 04      │  │
  │   │ vendor fields → the common schema (LLD §21)            │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ ENRICHMENT ENGINE                          doc 05      │  │
  │   │ asset · identity · cloud · network · app · threat      │  │
  │   │ ⚠ LAST STAGE THAT REQUIRES PLAINTEXT                   │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ SECURITY FACT ENGINE                       doc 06      │  │
  │   │ facts · entities · relationships (LLD §23–25)          │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ PRIVACY ENGINE                             doc 07      │  │
  │   │ remove payload · remove secrets · minimize (LLD §26–27)│  │
  │   │ ⚠ NOTHING PAST HERE HOLDS PLAINTEXT                    │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            ▼                                   │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ FORWARDER               batch → zstd → AES-GCM  doc 08 │  │
  │   └────────────────────────┬───────────────────────────────┘  │
  │                            │ mTLS                              │
  │   SQLite · spool · dead letter                 doc 09         │
  │   control plane: API · UI · vault · certs      doc 10         │
  │   response plane: agent gateway · commands     doc 11         │
  └────────────────────────────┼───────────────────────────────────┘
                               ▼
                       OVERLOOK SAAS                    doc 12
```

---

## 6. The documents

| # | Doc | LLD sections |
|---|---|---|
| 01 | [Ingestion Gateway](01-ingestion-gateway.md) | §12, §13, §36–39 |
| 02 | [Durable Event Buffer](02-durable-event-buffer.md) | §14–16, §69, §70 |
| 03 | [Parser Engine](03-parser-engine.md) | §18–20 |
| 04 | [Normalization Engine](04-normalization-engine.md) | §21 |
| 05 | [Enrichment Engine](05-enrichment-engine.md) | §22, §40 |
| 06 | [Security Fact Engine](06-security-fact-engine.md) | §23–25, §81 |
| 07 | [Privacy Engine](07-privacy-engine.md) | §26, §27 |
| 08 | [Forwarder](08-forwarder.md) | §28–35 |
| 09 | [Local State and Storage](09-local-state-and-storage.md) | §41–45, §70 |
| 10 | [Control Plane](10-control-plane.md) | §46–54, §63–69, §74 |
| 11 | [Response Plane](11-response-plane.md) | §55–59 |
| 12 | [The SaaS Side](12-saas-side.md) | §76–81 |
| 13 | [Escalations](13-escalations.md) | — |

---

## 7. The reduction cascade

Why the collector exists at all, rather than shipping logs to a cloud lake. This
is LLD §88 expressed as arithmetic.

```
  ONE EDGE LARGE COLLECTOR AT 10,000 EPS

  STAGE                          VOLUME/DAY     SURVIVING
  ────────────────────────────────────────────────────────
  raw at the gateway                864 GB        100%
  after parse (structured)        1,037 GB        120%   ← grows
  after normalization             1,037 GB        120%
  after noise filter                260 GB         30%
  after dedup (LLD §40)              90 GB         10.4%
  facts + entities + relations      2.1 GB          0.24%
  after aggregation                 180 MB          0.021%
  compressed on the wire             45 MB          0.005%

  864 GB  →  45 MB     ≈  19,000 : 1
```

**Parsing makes the data bigger.** Structured JSON with named fields is ~20%
larger than the raw line. Any sizing estimate assuming parsing reduces volume is
wrong, and it is wrong at exactly the stage where memory is tightest. This is
also the arithmetic behind ESC-1.

**The real drop is fact extraction, not compression.** Ten thousand
authentication events describe *one* relationship. Compression contributes only
the final 4:1.

---

## 8. Meridian, the recurring example

Continuing `../12-end-to-end-deployment-story.md`.

```
  MERIDIAN FINANCIAL — 4 COLLECTORS
  (four, because of LLD §71's ceiling — scale out, not up)

  COL-mer-01   Large    datacentre   FortiGate · Palo Alto · NSX
                                     ~11,000 EPS   mostly syslog
  COL-mer-02   Large    datacentre   CrowdStrike · FortiEDR ·
                                     Scalefusion   ~9,000 EPS
  COL-mer-03   Large    DMZ          AWS · GCP · GitHub · DLP
                                     ~14,000 EPS   mostly API poll
  COL-mer-04   Medium   branch/OT    syslog · SNMP   ~2,000 EPS

  TOTAL ~36,000 EPS · 2.9M entities · 2.9M relationships
  47 crown jewels · 12,000 identities · 2,100 service accounts
```

---

*Next: [Ingestion Gateway](01-ingestion-gateway.md)*

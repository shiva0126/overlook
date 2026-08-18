# Overlook — The Edge Collector, Service by Service

**Version:** 1.0
**Date:** 2026-08-18
**Parent:** `Overlook_Edge_Collector_Engineering_Handoff_v1.1` §6.1
**Status:** Architecture. No implementation.

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> It specifies the seven services of handoff §6.1, as elaborated in the
> service architecture of 2026-08-18. Where this series adds something the
> handoff does not state, it is marked **PROPOSED** and carries the reason.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Open escalations: `../01-system-design.md` §41.

---

## 1. Why this series exists

The handoff answers *how to run the work* — twenty phases, stop/go gates,
evidence, ceilings. The rest of `docs/` answers *how it works and why*. Neither
answers **what exactly to build**, and that is the document an engineer sits
down with.

This series is that layer for the collector. One document per service, each
specifying inputs, outputs, mechanics, resource budget, and the ways it fails.

```
  LAYER 1   how to run the work        → the handoff
  LAYER 2   how it works and why       → the rest of docs/
  LAYER 3   what exactly to build      → THIS SERIES
```

---

## 2. Reconciling the three diagrams

The service architecture arrived as three diagrams that disagree in three
places. They are reconciled here, and the reasoning is recorded because the
disagreements are meaningful rather than cosmetic.

| Point | Diagram 1 | Diagram 2 | Diagram 3 | Reconciled |
|---|---|---|---|---|
| Local Analytics | branch off Enrich | inline, before Detection | absent | **Branch, fed from post-parse** |
| Redis / Local Store | shown | absent | absent | **Present, roles separated (§08)** |
| Detection | absent | a stage | absent | **Not a stage — see §5** |
| Batch order | Compress→Encrypt | Compress→Encrypt | Compress→Encrypt→Batch | **Batch→Compress→Encrypt (§07)** |
| Privacy position | after facts | after facts | after facts | **After facts, before persistence** |

### 2.1 The reconciled pipeline

```
  CUSTOMER ENVIRONMENT
  ┌──────────────────────────────────────────────────────────────────┐
  │                     OVERLOOK EDGE COLLECTOR                      │
  │                                                                  │
  │   CONNECTORS                                                     │
  │   AWS · Azure · GCP · FortiGate · CrowdStrike · FortiEDR ·       │
  │   Scalefusion · GitHub · NSX · DLP · SIEM · DB · SNMP           │
  │        │         │          │          │                         │
  │      PULL      PUSH      STREAM      AGENT    ← 4 ingress classes │
  │        └─────────┴──────────┴──────────┘                         │
  │                          ▼                                       │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 1  INGESTION GATEWAY                                     │  │
  │   │    auth · validate · rate limit · flow control · ACK     │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  │                              ▼                                   │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 2  DURABLE EVENT BUFFER — NATS JetStream                 │  │
  │   │    IS the journal. fsync before ack. 4 streams.          │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  │                              ▼                                   │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 3  PARSER ENGINE                                         │  │
  │   │    detect · parse · validate · normalize · quarantine    │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  │                              ├───────────────► 8  LOCAL ANALYTICS│
  │                              ▼                    (reduced, 7d)  │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 4  ENRICHMENT ENGINE          ◄──────► 8  REDIS (cache)  │  │
  │   │    asset · identity · geo · threat · network · tags      │  │
  │   │    ⚠ LAST STAGE THAT REQUIRES PLAINTEXT                  │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  │                              ▼                                   │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 5  SECURITY FACT ENGINE       ◄──────► 8  LOCAL STORE    │  │
  │   │    entities · relationships · findings · observations    │  │
  │   │    merge windows · coverage windows · confidence         │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  │                              ▼                                   │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 6  PRIVACY ENGINE                                        │  │
  │   │    TOKENIZE identifiers · DROP secrets · DROP raw payload│  │
  │   │    ⚠ NOTHING PAST THIS POINT HOLDS PLAINTEXT             │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  │                              ▼                                   │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │ 7  METADATA FORWARDER                                    │  │
  │   │    batch → compress → encrypt → ship · spool on failure  │  │
  │   └──────────────────────────┬───────────────────────────────┘  │
  └──────────────────────────────┼──────────────────────────────────┘
                                 │ mTLS
  ═══════════════════════════════╪═══════════════════════════════════
                                 ▼
                          OVERLOOK SAAS  (§09)
```

### 2.2 Two lines that matter more than the boxes

```
  THE PLAINTEXT LINE       after service 4
    Enrichment is the last stage that needs real values — geo needs the
    real IP, identity enrichment needs the real username, threat intel
    needs the real hash. Privacy CANNOT move earlier than this.

  THE PERSISTENCE LINE     after service 6
    Everything that survives a restart should sit on the far side of
    privacy. Local Analytics is the one deliberate exception, and §08
    states what it is allowed to hold and why.
```

---

## 3. The seven services and three stores

| # | Service | Role | Stateful? |
|---|---|---|---|
| 1 | [Ingestion Gateway](01-ingestion-gateway.md) | Front door for four ingress classes | no |
| 2 | [Durable Event Buffer](02-durable-event-buffer.md) | NATS JetStream — the journal | **yes, on disk** |
| 3 | [Parser Engine](03-parser-engine.md) | Raw → structured, normalized | no (cache only) |
| 4 | [Enrichment Engine](04-enrichment-engine.md) | Add context from caches | no (cache only) |
| 5 | [Security Fact Engine](05-security-fact-engine.md) | Events → entities, relationships, findings | **yes, windows** |
| 6 | [Privacy Engine](06-privacy-engine.md) | Tokenize, drop, minimize | no (key only) |
| 7 | [Metadata Forwarder](07-metadata-forwarder.md) | Batch, compress, encrypt, ship | **yes, spool** |
| — | [Local state and stores](08-local-state-and-stores.md) | Redis · Local Store · Local Analytics | **yes** |
| — | [The SaaS side](09-saas-side.md) | What receives all of this | — |

**Process separation is a requirement, not a style.** Seven processes means the
parser can be restarted without dropping a syslog stream, and a parser panic on
malformed input cannot take the gateway with it. It also means each has its own
resource budget, which is the only way a hard ceiling is enforceable.

---

## 4. The resource budget

Handoff §5 sets the ceiling. It does not divide it. Without a division, the
first service to be written takes what it wants and the last one to be written
discovers there is nothing left.

**Edge L — 12 vCPU / 64 GB / 1 TB, at 10,000 EPS sustained.**

### 4.1 CPU

```
  SERVICE                  vCPU    NOTE
  ─────────────────────────────────────────────────────────────
  1  Ingestion Gateway      1.5    TLS termination, framing, validation
  2  JetStream              1.0    I/O bound; fsync coordination
  3  Parser Engine          4.0    THE HOT PATH — regex, grok, Drain
  4  Enrichment Engine      1.0    cache lookups, memory bound
  5  Security Fact Engine   1.0    window aggregation, extraction
  6  Privacy Engine         0.25   HMAC-SHA256 is nearly free
  7  Metadata Forwarder     0.5    zstd + TLS
     Redis                  0.25
     Local Store            0.25
     Local Analytics        0.25   niced, capped, may be starved
  ─────────────────────────────────────────────────────────────
     ALLOCATED             10.0
     OS + RESERVE           2.0    burst headroom, not spare capacity
```

**Parsing is a third of the machine and that is correct.** It is the only stage
that touches every byte of every event. Every other stage operates on
progressively less data. If parsing is not the largest allocation, something
upstream is doing work it should not.

### 4.2 Memory

```
  SERVICE                   RAM    HOLDS
  ─────────────────────────────────────────────────────────────
  1  Ingestion Gateway      2 GB   connection buffers, TLS sessions
  2  JetStream              4 GB   in-flight messages + stream index
  3  Parser Engine          8 GB   compiled parsers, LILAC cache, batches
  4  Enrichment Engine      4 GB   working set
  5  Security Fact Engine  12 GB   MERGE AND COVERAGE WINDOWS ← the big one
  6  Privacy Engine         1 GB   token cache, sealed key material
  7  Metadata Forwarder     2 GB   outbound batch buffers
     Redis                 12 GB   asset · identity · geo · TI caches
     Local Store            4 GB   Postgres shared buffers
     Local Analytics        6 GB   DuckDB query memory, HARD CAPPED
  ─────────────────────────────────────────────────────────────
     ALLOCATED             55 GB
     OS + page cache        9 GB
```

The Fact Engine and Redis together are 37% of memory, and both are windows over
recent data. **Both must be bounded by policy rather than by available RAM**, or
the collector's memory use becomes a function of the customer's estate size and
the ceiling stops being a ceiling.

### 4.3 Disk — and the number that governs everything

```
  ALLOCATION                    SIZE    NOTE
  ─────────────────────────────────────────────────────────────
  OS + binaries + logs          40 GB
  JetStream retention          200 GB   ← see below
  Local Store (Postgres)       100 GB
  Local Analytics (Parquet)    300 GB   7 days, reduced, not raw
  Evidence (handoff §25)        50 GB   14 days, hash + excerpt
  Outbound spool               100 GB   facts only — small, and post-merge
  ─────────────────────────────────────────────────────────────
  ALLOCATED                    790 GB
  FREE / RESERVE               210 GB   a hard ceiling needs real slack
```

**JetStream retention is the most consequential number in the collector.**

```
   5,000 EPS × ~1 KB  =  5 MB/s  =  18 GB/hour  =  432 GB/day
  10,000 EPS × ~1 KB  = 10 MB/s  =  36 GB/hour  =  864 GB/day

  200 GB of retention therefore buys

    at  5,000 EPS   ~11 hours
    at 10,000 EPS   ~5.5 hours
    at 20,000 EPS   ~2.8 hours   ← and you are over the Edge L target

  THAT NUMBER IS NOT A CONFIG DETAIL. IT IS AN SLA:
  "the collector survives a N-hour parser outage with zero data loss."
```

Retention has to be expressed in **hours**, not days, because JetStream buffers
**raw** events — the highest-volume form the data ever takes. A default of
"7 days" would need 6 TB and the ceiling is 1 TB. See `02 §5`.

---

## 5. What the collector does NOT do

Two exclusions, and the reason each matters.

```
  ✕  ATTACK PATHS · RISK SCORING · THE GRAPH · CORRELATION
     handoff §3.2 — SaaS. Not a preference; permission closure and
     path traversal need a whole-estate view no single collector has.

  ✕  DETECTION
     Diagram 2 lists "Detection" as a stage. It is not carried into
     the reconciled pipeline, for two reasons:

       1  BUDGET     rules plus tuning plus possible ML does not fit
                     the 1.0 vCPU the Fact Engine has, and there is
                     nowhere to take it from.
       2  POSITION   detection is Stellar Cyber's ground and they have
                     a detection research team. Overlook is exposure.
                     ../07-competitive-landscape.md is the long form.

     ⚠ NEEDS A DECISION. If "Findings" in the Security Fact Engine
       means EXPOSURE findings — a misconfigured trust policy, an
       over-broad OIDC subject, an unrotated key — then there is no
       disagreement and this note is moot. If it means DETECTIONS,
       it is a scope change with a resource cost and belongs in an
       ADR before Phase 1.
```

---

## 6. The reduction cascade

Everything about the collector's economics falls out of this table. It is the
reason the collector exists at all, rather than shipping logs to a cloud lake.

```
  ONE EDGE L COLLECTOR AT 10,000 EPS

  STAGE                          VOLUME/DAY     SURVIVING
  ────────────────────────────────────────────────────────
  raw at the gateway                864 GB        100%
  after parse (structured)        1,037 GB        120%   ← grows
  after noise filter                260 GB         30%
  after dedup                        90 GB         10.4%
  facts extracted                   2.1 GB          0.24%  ← the drop
  after merge windows               180 MB          0.021%
  compressed on the wire             45 MB          0.005%

  864 GB  →  45 MB     ≈  19,000 : 1
```

Two observations that are easy to miss:

**Parsing makes the data bigger.** Structured JSON with normalized field names
is ~20% larger than the raw line. Every sizing estimate that assumes parsing
reduces volume is wrong, and it is wrong at the exact stage where memory is
tightest.

**The real drop is fact extraction, not compression.** Going from events to
entities-and-relationships is a 40:1 reduction on its own, because ten thousand
authentication events describe *one* relationship. Compression only contributes
the last 4:1. A design that ships events and compresses them hard is not the
same architecture with a worse constant — it is three orders of magnitude
different.

---

## 7. Meridian, as the series will use it

Continuing `../12-end-to-end-deployment-story.md`.

```
  MERIDIAN FINANCIAL — 4 COLLECTORS

  COL-mer-01   Edge L    datacentre    FortiGate · Palo Alto · NSX
                                       ~11,000 EPS   mostly STREAM
  COL-mer-02   Edge L    datacentre    CrowdStrike · FortiEDR ·
                                       Scalefusion    ~9,000 EPS
  COL-mer-03   Edge L    DMZ           AWS · GCP · GitHub · DLP
                                       ~14,000 EPS   mostly PULL
  COL-mer-04   Edge M    branch/OT     syslog, SNMP    ~2,000 EPS

  TOTAL ~36,000 EPS · 2.9M entities · 2.9M live edges
  47 crown jewels · 12,000 identities · 2,100 service accounts
```

The 12/64/1 TB ceiling is why there are four and not one. `COL-mer-03` carries
the most EPS but the least raw volume, because PULL sources arrive already
structured — which is the whole argument of `../08-connector-benchmark-and-alignment.md`
restated in deployment terms.

---

## 8. Reading order

The series is written in dependency order and each document ends by linking to
the next. If you are reading one document only, read
[Security Fact Engine](05-security-fact-engine.md) — it is where the product
actually happens.

---

*Next: [Ingestion Gateway](01-ingestion-gateway.md)*

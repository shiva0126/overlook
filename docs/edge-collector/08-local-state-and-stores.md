# 8 — Local State and Stores

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies Redis, Local Store and Local Analytics from the service
> architecture. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget: **0.75 vCPU · 22 GB RAM · 450 GB disk** across all three.

---

## 1. Purpose

Three stores appear in the service architecture with no stated boundary between
them. Without one, state lands wherever it is convenient, and the collector
acquires the failure mode where losing a cache loses something irreplaceable.

The boundary is durability:

```
  REDIS             DERIVED. Rebuildable from elsewhere.
                    Lose it → degraded quality, self-healing.

  LOCAL STORE       AUTHORITATIVE. Exists nowhere else.
                    Lose it → real, permanent loss.

  LOCAL ANALYTICS   DIAGNOSTIC. Reduced, time-bounded, disposable.
                    Lose it → lose local troubleshooting only.
```

Anything that cannot be rebuilt goes in the Local Store. Anything that can, must
not.

---

## 2. Redis — the enrichment cache

```
  BUDGET   0.25 vCPU · 12 GB RAM · no disk persistence
  HOLDS    the enrichment working set (04 §4.1)
  LOSING IT  enrichment misses across the board, records marked
             enrichment.missing, background warmers refill from
             the Local Store. Facts keep flowing at lower quality
             for a few minutes.
```

```
  CONFIGURATION THAT IS NOT OPTIONAL

    maxmemory              12gb
    maxmemory-policy       allkeys-lru
    save                   ""          ← NO RDB SNAPSHOTS
    appendonly             no          ← NO AOF

  Persistence is disabled deliberately. Redis holds nothing worth
  recovering, and an RDB fork at 12 GB stalls the process and
  writes 12 GB to a disk the retention budget has already spent.

  maxmemory is the ceiling being enforced. Without it, Redis grows
  to fill available RAM, and the collector's memory use becomes a
  function of the customer's estate size rather than of its
  configuration — at which point 00 §4.2 is fiction.
```

**Redis must never hold the token map.** It is the most sensitive object in the
collector (`06 §7`), and a cache with LRU eviction and no persistence is exactly
the wrong home: entries would silently disappear, de-tokenization would return
partial results, and a memory dump would expose the estate.

**Consider whether Redis is needed at all.** An in-process cache in the
enrichment workers would be faster, would remove a process, and would free
0.25 vCPU. Redis earns its place only if enrichment scales to multiple worker
processes that must share a cache, or if warm cache survival across an
enrichment restart is worth the complexity. Worth an ADR before Phase 1 rather
than an assumption.

---

## 3. Local Store — the authoritative state

```
  BUDGET   0.25 vCPU · 4 GB RAM · 100 GB disk
  ENGINE   PostgreSQL — the collector already needs relational
           queries for the entity registry, and SQLite's
           single-writer model conflicts with seven processes
```

### 3.1 What it holds, and what losing each costs

```
  ┌─ IRREPLACEABLE ───────────────────────────────────────────────┐
  │                                                               │
  │  TOKEN MAP           token ↔ plaintext, encrypted at rest     │
  │    LOSING IT: every fact ever shipped becomes permanently     │
  │    unreadable. The SaaS graph survives and is anonymous       │
  │    forever. THIS IS THE MOST IMPORTANT 20 GB IN THE PRODUCT.  │
  │                                                               │
  │  CONNECTOR CURSORS   per-connector fetch position             │
  │    LOSING IT: re-fetch from the beginning (duplicates, and    │
  │    API quota) or skip forward (a permanent gap). Neither      │
  │    is acceptable; both are avoidable.                         │
  │                                                               │
  │  COVERAGE WINDOWS    open and historical, per connector       │
  │    LOSING IT: SaaS cannot reason about what was and was not   │
  │    observed. Retraction safety is gone. (05 §6)               │
  │                                                               │
  │  TOKENIZATION KEY    sealed, or a reference to a sealed store │
  │    LOSING IT: same as losing the token map, plus no new       │
  │    facts can join the old ones.                               │
  └───────────────────────────────────────────────────────────────┘

  ┌─ EXPENSIVE TO REBUILD ────────────────────────────────────────┐
  │  ENTITY REGISTRY     local entities, last-emitted state       │
  │    used to decide whether an entity has CHANGED and needs     │
  │    re-emission. Losing it re-emits everything once — a        │
  │    volume event, not a loss event.                            │
  │                                                               │
  │  RELATIONSHIP STATE  what is currently asserted               │
  │    same: rebuild by re-asserting, SaaS deduplicates.          │
  │                                                               │
  │  MERGE WINDOW CHECKPOINTS   every 30 s (05 §8)                │
  │                                                               │
  │  CACHE WARM SET      top keys by access, for 04 §4.3          │
  └───────────────────────────────────────────────────────────────┘

  ┌─ OPERATIONAL ─────────────────────────────────────────────────┐
  │  connector configuration and credentials (sealed)             │
  │  parser bundle version and load history                       │
  │  evidence index (handoff §25) — 14 days, hash + excerpt       │
  │  the de-tokenization audit log (06 §6)                        │
  │  ADR-relevant config change history                           │
  └───────────────────────────────────────────────────────────────┘
```

### 3.2 Sizing

```
  token map          2.9M entities × ~180 B, indexed        ~20 GB
  entity registry    2.9M × ~400 B                          ~14 GB
  relationship state 2.9M × ~250 B                          ~11 GB
  coverage windows   90 days × 40 connectors                 ~2 GB
  evidence index     14 days                                ~12 GB
  audit + config     90 days                                 ~3 GB
  indexes, WAL, bloat                                       ~28 GB
                                                          ────────
                                                            ~90 GB
  ALLOCATED                                                 100 GB
```

**The token map is 22% of the Local Store and 100% of the product's readability.**
It should be backed up on a schedule, encrypted, to storage *inside the customer's
environment*, and that backup should be tested — because a restore that has never
been exercised is a plan, not a capability.

### 3.3 The backup that must never leave

```
  BACK UP          the token map, the tokenization key reference,
                   connector cursors, coverage windows
  DESTINATION      customer-controlled storage, inside their
                   environment
  ENCRYPTION       at rest, with a key the customer holds

  ⚠ NEVER to Overlook SaaS, never to an Overlook-controlled
    bucket, never through an Overlook-operated backup service.

    A backup of the token map in Overlook's possession is the
    entire privacy architecture undone by an operational
    convenience, and it would be discovered in the first serious
    security review a customer runs.
```

---

## 4. Local Analytics — bounded, reduced, diagnostic

Escalation **E6**. The pre-handoff design specified a 30-day Parquet dataset over
parsed events with DuckDB on top. Against a 1 TB ceiling that does not fit, and
this is the resolution.

### 4.1 What it is for

```
  IN SCOPE — questions answerable only where the raw data is

    "why did CON-fortigate-dc-02's parse rate drop at 02:11?"
    "what does the unparseable 3 EPS actually look like?"
    "which source is responsible for the cardinality spike?"
    "show me the events behind this one relationship"
    "what changed in the traffic profile after the upgrade?"

  OUT OF SCOPE — SaaS does these, with the whole estate in view

    attack paths · risk scoring · correlation · the graph ·
    exposure metrics · anything in ../analytics/
```

**This is a troubleshooting tool for the person operating the collector.** It is
not a local version of the product, and the moment it starts answering product
questions it will need retention, and retention is what does not fit.

### 4.2 The reduced projection

```
  IT DOES NOT STORE PARSED EVENTS. It stores a projection.

  KEPT
    event_time (second granularity) · connector_id · event.action ·
    parser + version · parse outcome · source.ip, destination.ip
    (TOKENIZED) · ports · protocol · bytes · outcome ·
    enrichment coverage flags · a 200-byte raw excerpt ON PARSE
    FAILURE ONLY

  DROPPED
    everything else, including all free text and all successfully
    parsed raw lines

  SIZE
    parsed event                 ~1,330 B
    reduced projection             ~110 B
    Parquet + zstd, columnar        ~14 B effective

    10,000 EPS × 14 B  =  12 GB/day
    7 days             =  84 GB      ← against a 300 GB allocation
    30 days            = 360 GB      ← does not fit alongside
                                       everything else
```

```
  RETENTION IS 7 DAYS, NOT 30.

  7 days covers the incident-response window that matters — the
  02:11 format change in 03 §9.2 was found within eight hours, and
  a week of margin is generous for that class of problem.

  30 days would answer trend questions. Trend questions belong to
  SaaS, which has ClickHouse, the whole estate, and no ceiling.
```

### 4.3 It runs on the far side of privacy

```
  Identifiers in the analytics dataset are TOKENIZED, using the
  same key and the same tokens as the fact stream.

  WHY, when the data never leaves the collector anyway:

    1  the operator can join an analytics row to a fact in SaaS
       by token, directly — which is the main thing they want
    2  a compromise of the collector's disk does not hand over a
       plaintext record of everything that happened
    3  it removes the awkward question of whether a 7-day store
       of customer identifiers on an appliance is in scope for
       the customer's own data-retention policy

  The operator de-tokenizes at query time through the same path
  as the browser (06 §6), against the same audit log.
```

### 4.4 Resource discipline

```
  DuckDB memory       6 GB HARD CAP, enforced by configuration
  CPU                 0.25 vCPU, niced BELOW every other service
  I/O                 lowest priority; the buffer's fsync wins (02 §7)
  Concurrency         one query at a time, 60 s timeout
  Availability        MAY BE STARVED. Under load the collector's
                      job is to collect; a slow analytics query is
                      the correct sacrifice, and the UI should say
                      "deprioritised under load" rather than hang.
```

---

## 5. Considerations

**Three stores is two more than ideal, and each must justify itself.** Redis is
a cache that may not need to be a separate process (§2). Local Analytics is a
convenience whose absence would cost troubleshooting time, not correctness. Only
the Local Store is unambiguously required. If the collector needs to shed
complexity to meet a gate, that is the order to shed it in.

**Postgres on the collector is an appliance component, not a database
deployment.** No superuser access, no ad-hoc connections, no extensions beyond
what the collector ships. It should be as invisible as the filesystem, and its
configuration is part of the image rather than a customer-tunable surface.

**Every store needs a bound expressed in configuration, not in hope.** Redis has
`maxmemory`. Postgres has a disk allocation and a retention policy per table.
DuckDB has a memory cap and a 7-day partition drop. A hard ceiling that depends
on nothing growing unexpectedly is not a ceiling.

**Migrations must be forward-only and must not require downtime.** A schema
migration that stops the collector drops STREAM traffic for its duration
(`01 §9.2`). Additive columns, background backfills, no blocking rewrites of the
token map.

**The evidence index is not the evidence.** Handoff §25 requires evidence per
gate. Storing full artifacts for 14 days at Meridian volumes does not fit; the
index stores hashes, excerpts and pointers, and the full artifact lives wherever
the customer's own retention already keeps it. This is escalation **E7** and this
is the proposed resolution.

---

## 6. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Token map in Redis | Silent LRU eviction; partial de-tokenization forever | Local Store only, §2 |
| Token map lost, no backup | Every fact ever shipped becomes permanently unreadable | Encrypted customer-side backup, tested restore |
| Token map backup sent to SaaS | The entire privacy architecture voided | Never leaves the environment, §3.3 |
| Redis without `maxmemory` | Grows to fill RAM; the 64 GB ceiling is fiction | `maxmemory` + `allkeys-lru` |
| Redis persistence enabled | 12 GB fork stall, 12 GB of unbudgeted disk | `save ""`, `appendonly no` |
| Cursors lost | Duplicate refetch or a permanent gap | Local Store, backed up |
| Local Analytics at 30 days | 360 GB; the ceiling breached | 7 days on a reduced projection, §4.2 |
| Local Analytics stores parsed events | ~95× the disk | Projection only, ~14 B/event |
| Analytics competes for I/O | Buffer fsync latency spikes, STREAM drops | Lowest I/O priority, may be starved |
| Blocking migration | STREAM data lost for the migration's duration | Forward-only, additive, online |
| Plaintext in the analytics store | A disk compromise yields 7 days of the estate in clear | Tokenized at rest, §4.3 |

---

## 7. Example: Meridian

### 7.1 Actual footprint on COL-mer-01, day 40

```
  REDIS
    used_memory                    9.8 GB / 12 GB
    keys                           4.1 M
    hit rate                       97.4%
    evictions/day                  ~180 K   (healthy LRU churn)

  LOCAL STORE (Postgres)
    token map                      18.2 GB   2.84 M entries
    entity registry                13.1 GB
    relationship state             10.4 GB
    coverage windows                1.7 GB   40 days retained
    evidence index                 11.9 GB   14 days
    audit + config                  2.1 GB
    indexes / WAL                  26.8 GB
                                  ─────────
                                   84.2 GB / 100 GB

  LOCAL ANALYTICS (Parquet)
    7 days × ~11.8 GB              82.6 GB / 300 GB
    ⚠ 218 GB UNUSED

  TOTAL                           176.6 GB of a 450 GB allocation
```

### 7.2 The 218 GB question

```
  Local Analytics is using 28% of its allocation. The obvious move
  is to extend retention to 25 days and use it.

  DON'T. Give it back to JetStream.

  RAW_STREAM at 4 h was the constraint that cost 3 h 20 m of
  full-fidelity FortiGate data during the 03 §9.2 incident. The
  investigation took 7 h 20 m; the buffer held 4.

  REALLOCATED
    Local Analytics   300 GB → 120 GB    (7 days + headroom)
    JetStream         200 GB → 380 GB    RAW_STREAM 4 h → 9 h

  WHAT THAT BUYS
    a 9-hour window to detect, diagnose, fix and replay before
    data is unrecoverable. The same incident would have been
    fully recovered with 2 hours to spare.

  WHAT IT COSTS
    18 days of local troubleshooting history that nobody has
    ever asked for past day 3.

  Retention on a buffer that prevents permanent loss is worth
  more than retention on a store that only saves someone a
  question. This is the sort of reallocation the fleet should
  be reviewed for after the first sixty days in production.
```

### 7.3 A token map restore

```
  DRILL, week 6. COL-mer-02's disk is deliberately destroyed.

  BEFORE
    2.84 M tokens issued, all facts shipped and in the SaaS graph

  RESTORE
    09:00  new collector provisioned from the image
    09:04  token map restored from the customer's encrypted
           backup, 18.2 GB, taken 03:00, 6 h old
    09:11  tokenization key unsealed from Meridian's HSM
    09:12  connector cursors restored — 6 h stale, so 6 h of
           PULL data will be refetched. SaaS deduplicates.
    09:14  ⚠ 6 hours of tokens issued since the backup are
           MISSING FROM THE MAP.
           ~2,100 entities discovered between 03:00 and 09:00
           are in the SaaS graph, correctly joined, and CANNOT
           BE DE-TOKENIZED.
    09:15  they are re-discovered on the next full enumeration
           and re-tokenized — TO THE SAME TOKENS, because the
           HMAC is deterministic and the key is unchanged.
    10:40  full enumeration cycle completes across all
           connectors. The map is whole. Nothing is lost.

  THE PROPERTY THAT SAVED IT
    determinism. A random token map would have made those 2,100
    entities permanently unreadable, and a random-token design
    is what you get if nobody writes this down.

  THE LESSON FOR THE RUNBOOK
    back up the token map HOURLY, not daily. It is 18 GB and it
    is append-mostly; an incremental costs almost nothing, and
    the window it closes is the only window in which loss is
    possible at all.
```

---

*Next: [The SaaS side](09-saas-side.md)*

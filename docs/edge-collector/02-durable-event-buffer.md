# 2 — The Durable Event Buffer (NATS JetStream)

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 2. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **1.0 vCPU · 4 GB RAM · 200 GB disk.**

---

## 1. Purpose

Hold every accepted record until something downstream has finished with it, and
survive a crash without losing any of it.

This service is what makes the PUSH acknowledgement honest. Everything else in
the collector can be restarted, redeployed or killed; the buffer is the reason
that costs nothing.

---

## 2. Position

```
  INPUTS
    enveloped records from the Ingestion Gateway (01 §6)

  OUTPUTS
    ordered delivery to Parser Engine workers (03)
    depth and lag telemetry → the gateway's flow control (01 §4)

  CONSUMED BY
    03 parser engine, as a queue group
    the Controller, for buffer health
    replay tooling, for reprocessing after a parser fix
```

---

## 3. Why a broker, and why this one

An earlier position in this doc set rejected Kafka. That reasoning still holds
and does not apply to JetStream, which is worth stating explicitly because the
two look like the same decision.

```
  WHY KAFKA WAS REJECTED
    · a JVM, a broker cluster, ZooKeeper or KRaft, and an operational
      discipline of its own — inside a 12 vCPU appliance
    · partition management as a customer-visible concern
    · a second durable store alongside the journal we already needed

  WHY JETSTREAM IS DIFFERENT
    · one Go binary, embeddable, ~20 MB, no external dependency
    · it IS the journal — not a queue in front of one
    · file-backed streams with per-message fsync when asked
    · consumer groups, replay, and per-consumer acknowledgement
      are given rather than written
    · backpressure signals available as first-class stream state
```

**The decisive point is the second one.** We were always going to write a
durable, fsync-on-write, replayable, ordered log with per-consumer cursors and
crash recovery. That is what JetStream is. Writing it ourselves would have been
several thousand lines of the most dangerous code in the product, and the
correctness bugs would surface as silent data loss under crash, which is the
hardest class of bug to find.

### 3.1 The rule that keeps it honest

```
  THERE IS EXACTLY ONE DURABLE WRITE ON THE INGRESS PATH.

  JetStream IS the journal. There is no separate journal behind it.

  If both existed, every record would be fsynced twice, disk write
  bandwidth would halve, and the 1 TB ceiling would be consumed by
  two copies of the same data.
```

---

## 4. Stream design

### 4.1 Four streams, one per ingress class

```
  STREAM         SUBJECTS                      RETENTION   STORAGE
  ───────────────────────────────────────────────────────────────
  RAW_PULL       raw.pull.<connector_id>       6 h         file
  RAW_PUSH       raw.push.<connector_id>       24 h        file
  RAW_STREAM     raw.stream.<connector_id>     4 h         file
  RAW_AGENT      raw.agent.<connector_id>      12 h        file
```

**Why per class and not per connector.** A stream per connector means 40–100
streams on a collector, each with its own file set, index and memory overhead,
and a management surface that grows with the catalog. A single stream for
everything means one retention policy for classes with very different loss
models. Four streams, with the connector in the *subject*, gives per-class
retention and still allows per-connector filtering and replay.

**Why the retentions differ.**

```
  RAW_PUSH gets 24 h — the longest — because PUSH data is
    IRRECOVERABLE. Once acked, nobody else has a copy. It earns
    the most disk.

  RAW_PULL gets 6 h — the shortest that is useful — because the
    source API still holds it. Lose it and refetch.

  RAW_STREAM gets 4 h despite being the highest volume, precisely
    BECAUSE it is the highest volume. At 10,000 EPS of syslog, 4 h
    is already 144 GB. It cannot have more.

  RAW_AGENT gets 12 h; the agent's own buffer is the second copy.
```

### 4.2 Acknowledgement policy

```
  PUBLISH SIDE  (gateway → JetStream)

    RAW_PUSH     sync publish, fsync before publish-ack
                 ⚠ this is the contract in 01 §3.1. Nothing else
                   in the collector depends on a durability
                   guarantee this strict.

    RAW_AGENT    sync publish, fsync before publish-ack

    RAW_PULL     async publish, batched fsync every 100 ms
                 acceptable: the cursor is not advanced until the
                 publish-ack arrives, so a crash refetches

    RAW_STREAM   async publish, batched fsync every 100 ms
                 acceptable: the data is already best-effort by
                 the nature of UDP

  CONSUME SIDE  (JetStream → parser)

    explicit ack, ack-wait 30 s, max-deliver 3
    after 3 failed deliveries → the quarantine subject (03 §6)
```

The split matters for throughput. Per-message fsync costs roughly 0.5–2 ms on
NVMe; at 10,000 EPS that is unaffordable. Batching it for the two classes that
can tolerate batching is what makes the budget work, and paying it in full for
the two that cannot is what makes the guarantee real.

---

## 5. Retention is an SLA, not a setting

```
   5,000 EPS × ~1 KB  =   5 MB/s  =  18 GB/hour
  10,000 EPS × ~1 KB  =  10 MB/s  =  36 GB/hour
  20,000 EPS × ~1 KB  =  20 MB/s  =  72 GB/hour

  200 GB TOTAL ACROSS FOUR STREAMS BUYS

    at  5,000 EPS    ~11 hours of total downstream outage
    at 10,000 EPS    ~5.5 hours
    at 20,000 EPS    ~2.8 hours   ← and Edge L is not rated for this

  STATE IT AS A COMMITMENT:
  "COL-mer-01 tolerates a 5 hour parser outage with zero loss on
   PULL, PUSH and AGENT."
```

Two consequences worth pinning down before Phase 1:

**Retention is in hours because the buffer holds raw.** Raw is the largest form
the data ever takes — larger than parsed (which is 20% bigger still, but never
persists here), and four orders of magnitude larger than the facts that
eventually ship. A "7 day" default would need 6 TB.

**Retention must be per-stream byte-limited, not only time-limited.** A time
limit alone means an unexpected traffic spike fills the disk before the time
window expires. Both limits, with the byte limit as the hard one.

```
  RAW_STREAM   max_age 4h   max_bytes 144 GB   discard old
  RAW_PUSH     max_age 24h  max_bytes  30 GB   discard NEW ← see below
  RAW_PULL     max_age 6h   max_bytes  16 GB   discard old
  RAW_AGENT    max_age 12h  max_bytes  10 GB   discard old
```

### 5.1 `discard: new` on RAW_PUSH is deliberate

```
  discard OLD    when full, delete the oldest messages to make room.
                 The publisher never knows. Data is lost silently.

  discard NEW    when full, REFUSE THE PUBLISH. The publish fails,
                 the gateway does not ack, the sender retries.

  For PUSH, silent loss is exactly the failure the fsync-before-ack
  contract exists to prevent. Refusing loudly is the whole point:
  it converts a data loss into a backpressure event (01 §4.1).

  For the other three, discard old is correct — refetch, or it was
  best-effort to begin with.
```

---

## 6. Consumers and replay

### 6.1 Parser workers as a queue group

```
  one durable consumer per stream, shared by N parser workers

    RAW_STREAM  →  consumer PARSE_STREAM   →  8 workers
    RAW_PULL    →  consumer PARSE_PULL     →  2 workers
    RAW_PUSH    →  consumer PARSE_PUSH     →  2 workers
    RAW_AGENT   →  consumer PARSE_AGENT    →  2 workers

  Workers are stateless and interchangeable. Scaling the parser is
  adding workers to the group; JetStream distributes and tracks
  acknowledgement per message.
```

**Ordering is per subject, not per stream.** Events from one connector arrive at
the parser in order. Events from different connectors interleave arbitrarily,
which is correct and is why every downstream stage keys on `event_time` rather
than arrival order.

### 6.2 Replay is the reason this is worth 200 GB

```
  A parser bug produces wrong facts for CON-fortigate-dc-01 for
  three hours. Without a buffer, the only remedy is to wait for the
  data to recur.

  WITH IT
    1  fix the parser, ship the content bundle
    2  create an ephemeral consumer over
         raw.stream.CON-fortigate-dc-01
       from a start time three hours ago
    3  reprocess into a shadow fact stream
    4  diff shadow against what was emitted
    5  emit corrections to SaaS as fact revisions

  This is only possible within the retention window. It is the
  strongest practical argument for buying the 200 GB, and it should
  be the first thing tested in the Phase gate for this service —
  a replay path that has never been exercised does not work.
```

---

## 7. Considerations

**Depth is the collector's single best health signal.** It is a leading
indicator: it rises before latency does, before facts go stale, before anything
is lost. The Controller's collector health page should lead with buffer depth
per stream, not with CPU.

**Memory storage is never correct here.** JetStream offers in-memory streams,
they are much faster, and using one would silently void the durability
guarantee this service exists to provide. File storage on all four, asserted at
startup, failing loudly if misconfigured.

**Do not put the buffer on the same spindle as Local Analytics.** Analytics
runs bursty sequential reads; the buffer needs consistent low-latency fsync.
Where the platform allows it, separate volumes. Where it does not, analytics
gets an I/O priority below the buffer's.

**One collector, one JetStream, no clustering.** JetStream clusters and it would
be tempting to cluster across a customer's collectors. Do not. Collectors are
independent by design (`../09-deployment-and-tenancy-model.md`); clustering them
creates a shared failure domain and a network dependency between sites for no
benefit that SaaS does not already provide.

**Right-size `max_bytes` per deployment, not per edition.** Two Edge L
collectors with the same specification can carry very different mixes. `COL-mer-03`
is 97% PULL and needs almost nothing in RAW_STREAM; `COL-mer-01` is the inverse.
A single default across the fleet wastes disk on one and starves the other.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| A second journal behind JetStream | Double fsync, half the write bandwidth, double disk | One durable write on the path — §3.1 |
| Memory-backed stream | Durability guarantee silently void | File storage asserted at startup |
| `discard: old` on RAW_PUSH | Acked data deleted silently under pressure | `discard: new` — §5.1 |
| Time-only retention | Disk full during a traffic spike | Byte limit is the hard limit |
| Retention set in days | 6 TB required, 1 TB available | Retention in hours, sized from EPS |
| Replay never tested | The 200 GB buys nothing when needed | Replay exercised in the service's gate |
| One stream per connector | 40–100 streams, unmanageable overhead | Four streams, connector in the subject |
| Clustering across collectors | Shared failure domain, cross-site dependency | Independent per collector |
| Buffer shares a spindle with analytics | fsync latency spikes, throughput collapses | Separate volume or I/O priority |

---

## 9. Example: Meridian

### 9.1 Sizing COL-mer-01 versus COL-mer-03

```
  COL-mer-01     10,900 EPS · 97% STREAM · avg 1.1 KB

    RAW_STREAM   4 h    max_bytes  168 GB    ← almost all of it
    RAW_PUSH    24 h    max_bytes   14 GB
    RAW_PULL     6 h    max_bytes    4 GB
    RAW_AGENT    —      not deployed
                        ─────────────────
                                   186 GB

  COL-mer-03     14,200 EPS · 92% PULL · avg 2.8 KB (structured JSON)

    RAW_PULL     6 h    max_bytes  110 GB    ← inverted
    RAW_PUSH    24 h    max_bytes   48 GB    ← GitHub webhooks
    RAW_STREAM   4 h    max_bytes    6 GB
    RAW_AGENT   12 h    max_bytes   20 GB
                        ─────────────────
                                   184 GB

  SAME EDITION. SAME DISK. OPPOSITE ALLOCATION.
  A fleet-wide default would have starved one of them.
```

Note that `COL-mer-03` carries 30% more EPS but similar bytes, because PULL
sources arrive as structured JSON — fewer, larger, already-parseable records.
This is the connector value-density argument
(`../08-connector-benchmark-and-alignment.md`) appearing as a disk allocation.

### 9.2 A replay, after a parser defect

```
  2026-08-18 09:14   a FortiGate parser update ships. It misreads the
                     `dstintf` field on VDOM-scoped traffic logs, so
                     egress connections are attributed to the wrong
                     interface.

  2026-08-18 12:40   an analyst notices CONNECTS_TO edges pointing
                     at the wrong network segment.

  IMPACT WINDOW      3 h 26 m · CON-fortigate-dc-01 and -02
                     ~76 M events · ~84 GB, still inside the 4 h
                     RAW_STREAM window WITH 34 MINUTES TO SPARE

  RECOVERY
    12:41  parser rolled back to the previous bundle
    12:52  fixed bundle validated against the quarantine corpus
    13:04  ephemeral consumer created:
             subjects  raw.stream.CON-fortigate-dc-0{1,2}
             start     2026-08-18T09:14:00Z
             deliver   to a shadow fact stream
    13:31  replay complete — 76 M events reprocessed in 27 minutes
    13:38  diff: 1,847 CONNECTS_TO relationships wrong
    13:44  corrections emitted to SaaS as fact revisions

  THIRTY-FOUR MINUTES OF MARGIN.

  Had RAW_STREAM been sized at 3 hours instead of 4, the data would
  have aged out during the investigation and 1,847 wrong edges would
  have stayed in the graph until they naturally re-observed —
  which, for a quarterly batch job's traffic, is ninety days.
```

---

*Next: [Parser Engine](03-parser-engine.md)*

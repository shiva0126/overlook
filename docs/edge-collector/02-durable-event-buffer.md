# 2 — The Durable Event Buffer (NATS JetStream)

**Series:** [The Edge Collector](00-index.md) · **LLD:** §14–16, §69, §70, §85

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. **This document
> carries the collector's largest open escalation** — LLD §15/§16 persist
> each event six times, which does not fit the disk in LLD §71. See
> [ESC-1](13-escalations.md).
> Budget: **1.0 vCPU · 6 GB RAM · 480 GB disk** (`00 §4`).

---

## 1. Purpose

Hold every accepted record until something downstream has finished with it, and
survive a crash without losing any of it.

This is what makes the PUSH acknowledgement honest. Everything else in the
collector can be restarted or killed; the buffer is the reason that costs
nothing.

---

## 2. Position

```
  INPUTS
    enveloped records from the Ingestion Gateway (01 §6)

  OUTPUTS
    ordered delivery to worker pools (LLD §16, §17)
    depth telemetry → flow control (01 §4), LLD §50

  CONSUMED BY
    03 parser engine, as a queue group
    08 forwarder, via OVERLOOK_FORWARD
    replay tooling, after a parser fix (§6.2)
```

---

## 3. Why JetStream and not Kafka

LLD §85 excludes Kafka from V1 and §86 selects JetStream. Both are right, and the
distinction is worth recording because they look like the same decision.

```
  WHY KAFKA IS EXCLUDED
    a JVM, a broker cluster, KRaft or ZooKeeper, and an operational
    discipline of its own — inside a 12 vCPU appliance
    partition management as a customer-visible concern
    a second durable store alongside a journal we already need

  WHY JETSTREAM IS DIFFERENT
    one Go binary, ~20 MB, no external dependency
    it IS the journal — not a queue in front of one
    file-backed streams with per-message fsync when asked
    consumer groups, replay and per-consumer acknowledgement given
    rather than written
    depth available as first-class state for LLD §37's levels
```

**The decisive point is the second.** We were always going to write a durable,
fsync-on-write, replayable, ordered log with per-consumer cursors and crash
recovery. That is what JetStream is, and hand-writing it would be several
thousand lines of the most dangerous code in the product — where the bugs
surface as silent loss under crash.

---

## 4. Streams and subjects

LLD §14 defines the subject hierarchy and §15 three streams.

```
  OVERLOOK_RAW          overlook.raw.aws
                        overlook.raw.azure
                        overlook.raw.fortigate
                        overlook.raw.syslog
                        overlook.raw.agent
                        → file · retention Limits · replication 1

  OVERLOOK_PROCESSING   overlook.parsed
                        overlook.normalized
                        overlook.enriched
                        → file  ⚠ see §5

  OVERLOOK_FORWARD      overlook.fact
                        overlook.forward
                        overlook.retry
                        → file

  overlook.deadletter   parse failures (LLD §20, doc 03 §6)
```

### 4.1 Subject granularity

LLD §14 keys `overlook.raw.*` by **connector type**, not connector instance.
Extending it to `overlook.raw.<type>.<connector_id>` costs nothing and buys two
things:

```
  · per-connector replay. The incident in §6.2 replays ONE firewall,
    not every firewall on the collector.
  · per-connector filtering for diagnostics without a full scan.

  Retention policy stays at the STREAM level, so this does not
  multiply configuration. It is a subject-naming change only.
```

### 4.2 Acknowledgement policy

```
  PUBLISH SIDE   (gateway → OVERLOOK_RAW)

    PUSH, AGENT     sync publish, fsync BEFORE publish-ack
                    → this is the 01 §3.1 contract. Nothing else in
                      the collector needs a guarantee this strict.

    PULL, STREAM    async publish, batched fsync every 100 ms
                    → PULL: the checkpoint is not advanced until the
                      publish-ack arrives, so a crash refetches.
                    → STREAM: already best-effort by nature of UDP.

  CONSUME SIDE

    explicit ack · ack-wait 30 s · max-deliver 3
    after 3 failed deliveries → overlook.deadletter (LLD §20)
```

Per-message fsync costs 0.5–2 ms on NVMe; at 10,000 EPS that is unaffordable.
Batching it for the two classes that tolerate it is what makes the budget work,
and paying it in full for the two that do not is what makes the guarantee real.

---

## 5. ⚠ The six-stage persistence problem

**This is escalation [ESC-1](13-escalations.md) and it is the reason this
document's disk budget carries a caveat.**

LLD §16's chain writes each event to file-backed storage at RAW, PARSED,
NORMALIZED and ENRICHED. LLD §70 retains parsed 6–24 hours.

```
  AT LLD §71 "LARGE" — 10,000 EPS, 1 TB — AT SIX HOURS

    RAW           1.0 KB   10 MB/s    216 GB
    PARSED        1.2 KB   12 MB/s    259 GB
    NORMALIZED    1.2 KB   12 MB/s    259 GB
    ENRICHED      1.5 KB   15 MB/s    324 GB
                                     ────────
                                     1,058 GB

  THE DISK IS 1,024 GB — before OS, SQLite, dead letter, logs or spool.
  Sustained write load: 49 MB/s for a 10 MB/s ingest.
```

```
  PROPOSED — PERSIST TWICE, NOT SIX TIMES

    OVERLOOK_RAW       file-backed, fsync before ack.
                       The durability contract. Untouched.

    in-process         parsed → normalized → enriched → fact
                       Go channels between worker pools. LLD §5 makes
                       this natural — it is ONE PROCESS. No
                       serialization, no disk, no broker hop.

    OVERLOOK_FORWARD   file-backed. Facts awaiting SaaS ack, plus
                       the spool.

  CRASH SAFETY IS UNCHANGED — in-flight events replay from RAW, which
  is what RAW retention is for. LLD §16's "ACK only after successful
  processing" is preserved by moving the RAW ack to after the FORWARD
  publish.

  RECOVERS   disk 1,058 GB → 216 GB · write I/O 49 → 10 MB/s ·
             40,000 marshal ops/sec of CPU
```

The disk allocation in `00 §4.3` assumes this resolution. If the chain must
stay, `OVERLOOK_PROCESSING` should be **memory-backed** — bounded, fast, lost on
restart, and replayable from RAW.

---

## 6. Retention is an SLA, not a setting

```
   5,000 EPS × ~1 KB  =  5 MB/s  =  18 GB/hour
  10,000 EPS × ~1 KB  = 10 MB/s  =  36 GB/hour

  420 GB OF RAW BUYS
    at  5,000 EPS   ~23 hours
    at 10,000 EPS   ~11.6 hours
    at 20,000 EPS   ~5.8 hours  ← and Large is not rated for this
```

```
  ⚠ LLD §69 SAYS raw_hours: 24 AND queue.max_disk_gb: 300.

  At 10,000 EPS, 300 GB is 8.3 hours, not 24. The two are only both
  true below ~3,500 EPS — the bottom of the Medium tier.
  → escalation ESC-2.

  PROPOSED: max_disk_gb is the only configured value. The collector
  COMPUTES and DISPLAYS the resulting window —
     "RAW retention: 11.6 hours at current 10,200 EPS"
  — and alarms when it falls below a floor. LLD §35's UI already
  shows "Estimated Buffer Remaining"; this is the same computation,
  driving the config rather than merely reporting it.
```

### 6.1 Limits must be byte-based as well as time-based

```
  A time limit alone means a traffic spike fills the disk before the
  window expires. Both, with bytes as the hard limit.

  OVERLOOK_RAW       max_bytes 420 GB   discard OLD
  OVERLOOK_FORWARD   max_bytes  60 GB   discard NEW  ← see below
  deadletter         max_bytes  30 GB   max_age 7 d (LLD §70)
```

```
  discard NEW ON OVERLOOK_FORWARD IS DELIBERATE.

  discard old  → the oldest FACTS are deleted to make room. Silent
                 loss of the thing the customer is buying.
  discard new  → the publish fails, privacy workers block, back
                 pressure propagates to the gateway, and the loss
                 becomes a visible pressure event instead.

  On OVERLOOK_RAW, discard old is correct: PULL refetches and
  STREAM was best-effort to begin with.
```

---

## 7. Consumers and replay

### 7.1 Worker pools as queue groups

```
  OVERLOOK_RAW      → consumer PARSE      → LLD §17  min 2 max 16
  overlook.parsed   → consumer NORMALIZE  →          min 2 max 12
  overlook.normalized → consumer ENRICH   →          min 2 max 8
  overlook.enriched → consumer FACTS      →          min 2 max 8
  overlook.fact     → consumer PRIVACY
  overlook.forward  → consumer FORWARD    →          min 2 max 8

  (under ESC-1's resolution the middle four become in-process
   channels with the same pool shapes and the same scaling inputs)

  SCALING INPUTS (LLD §17)  queue depth · EPS · CPU · latency
```

**Ordering is per subject, not per stream.** Events from one connector reach the
parser in order; different connectors interleave arbitrarily. That is correct,
and it is why every downstream stage keys on `timestamp` rather than arrival
order.

### 7.2 Replay is why the disk is worth spending

```
  A parser defect produces wrong facts for one firewall for three
  hours. Without a buffer the only remedy is waiting for the data
  to recur — which for a quarterly batch job's traffic is 90 days.

  WITH IT
    1  fix the parser, load the bundle (03 §4)
    2  ephemeral consumer over overlook.raw.fortigate.<id>
       from a start time three hours ago
    3  reprocess into a shadow fact stream
    4  diff shadow against what was emitted
    5  emit corrections to SaaS as fact revisions

  Only possible inside the retention window. This should be the
  FIRST thing exercised in this module's gate — a replay path that
  has never been run does not work.
```

---

## 8. Considerations

**Depth is the collector's best health signal.** It is a leading indicator: it
rises before latency does, before facts go stale, before anything is lost. LLD
§64's dashboard shows Queue in GB — it should show depth as a *percentage of the
configured limit*, because that is what LLD §37's levels are defined against.

**File storage on OVERLOOK_RAW, asserted at startup.** JetStream offers
memory-backed streams; they are much faster and using one silently voids the
durability guarantee this module exists to provide. Assert and fail loudly.

**Do not share a spindle with the spool.** The spool (LLD §29) writes in bursts
during an outage; RAW needs consistent low-latency fsync. Separate volumes where
the platform allows, I/O priority where it does not.

**One collector, one JetStream, no clustering.** JetStream clusters, and it would
be tempting to cluster across a customer's collectors. LLD §84 correctly puts
clustering in Phase 3. Collectors are independent by design
(`../09-deployment-and-tenancy-model.md`); clustering them creates a shared
failure domain and a cross-site network dependency for no benefit SaaS does not
already provide.

**Size per deployment, not per edition.** Two Large collectors with identical
specifications carry very different mixes — `COL-mer-03` is 92% API poll and
needs almost nothing in the syslog subjects; `COL-mer-01` is the inverse. A
fleet-wide default wastes disk on one and starves the other.

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Six-stage file persistence | 1,058 GB at 6 h against a 1 TB disk | ESC-1 — persist twice |
| `raw_hours` fixed at 24 | Silently becomes 8 h under load, and gets quoted in an incident | ESC-2 — derive and display |
| Memory-backed RAW | Durability guarantee silently void | File storage asserted at startup |
| `discard: old` on FORWARD | Facts deleted silently under pressure | `discard: new`, §6.1 |
| Time-only retention | Disk full during a spike | Byte limit is the hard limit |
| Replay never tested | The disk buys nothing when needed | Exercised in the module's gate |
| Clustering across collectors | Shared failure domain, cross-site dependency | Independent per collector; LLD §84 |
| Spool shares a spindle with RAW | fsync latency spikes; STREAM drops at the gateway | Separate volume or I/O priority |
| Depth shown in GB not % | The operator cannot see which pressure level applies | Percentage of configured limit |

---

## 10. Example: Meridian

### 10.1 Two Large collectors, opposite allocations

```
  COL-mer-01     10,900 EPS · 97% syslog · avg 1.1 KB

    overlook.raw.fortigate    max_bytes  340 GB
    overlook.raw.syslog                   40 GB
    overlook.raw.<push/pull>              40 GB
                              ─────────────────
    OVERLOOK_RAW                         420 GB → ~10.6 hours

  COL-mer-03     14,200 EPS · 92% API poll · avg 2.8 KB structured

    overlook.raw.aws          max_bytes  240 GB
    overlook.raw.azure/gcp                80 GB
    overlook.raw.agent                    60 GB
    overlook.raw.syslog                   40 GB
                              ─────────────────
    OVERLOOK_RAW                         420 GB → ~4.4 hours ⚠

  SAME EDITION. SAME DISK. COL-mer-03 GETS LESS THAN HALF THE
  RETENTION WINDOW, because API sources arrive as larger structured
  records.

  → COL-mer-03 needs either more disk or a shorter incident-response
    SLA, and the difference must be known before an incident rather
    than discovered during one.
```

### 10.2 A replay, after a parser defect

```
  09:14  a FortiGate parser update ships. It misreads `dstintf` on
         VDOM-scoped traffic logs, so egress connections are
         attributed to the wrong interface.

  12:40  an analyst notices CONNECTS_TO edges pointing at the wrong
         network segment.

  IMPACT   3 h 26 m · two connectors · ~76 M events · ~84 GB
           still inside COL-mer-01's ~10.6 h window, comfortably

  12:41  parser rolled back to the previous bundle
  12:52  fixed bundle validated against the dead-letter corpus
  13:04  ephemeral consumer, subjects
           overlook.raw.fortigate.con-fortigate-dc-0{1,2}
           start 09:14, deliver to a shadow fact stream
  13:31  replay complete — 76 M events in 27 minutes
  13:38  diff: 1,847 CONNECTS_TO relationships wrong
  13:44  corrections emitted to SaaS as fact revisions

  ON COL-mer-03, WITH ITS 4.4-HOUR WINDOW, THE SAME INCIDENT
  WOULD HAVE AGED OUT AT 13:36 — eight minutes before the diff
  completed.

  That is not a hypothetical margin. It is the argument for
  ESC-2's derived-and-displayed retention.
```

---

*Next: [Parser Engine](03-parser-engine.md)*

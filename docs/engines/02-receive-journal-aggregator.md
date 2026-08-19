# Receive, Ingest Journal, and E1 Stream Aggregator

**Series:** [Engine documentation](00-index.md) · **v1:** journal yes, aggregator no

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

Three tightly coupled things at the front door.

**Receive** terminates protocols, authenticates sources and establishes provenance. **The Ingest Journal** makes accepted data durable before anything else touches it. **E1, the Stream Aggregator**, collapses high-volume telemetry in memory before it can reach disk at all.

Together they answer one question: *what do we promise about data we have accepted?*

---

## 2. Position

```
  INPUTS
    PULL     connector fetch results        (from the worker pool)
    PUSH     webhook HTTP requests
    STREAM   syslog frames, NetFlow/IPFIX datagrams
    AGENT    mTLS batches from endpoints

  OUTPUTS
    journaled records with provenance  → E2 / E3
    aggregate records                  → E2 / E3  (stream path)
    acknowledgements                   → back to the source
    backpressure signals               → to E15 and to sources
```

---

## 3. Mechanics

### 3.1 The durability contract differs by ingress class

This is the whole design. Recoverability determines what we must promise.

```
  PULL      re-fetchable by cursor
            → NO journal fsync required. Cursor persisted only after
              the batch is safely downstream.
            → crash mid-fetch = refetch. Costs API calls, loses nothing.

  PUSH      NOT re-fetchable. Gone if dropped.
            → journal + FSYNC BEFORE returning 200.
            → returning 200 before fsync is the classic data-loss bug.

  STREAM    not re-fetchable, but individually near-worthless
            → AGGREGATE FIRST, journal the aggregate.
            → batched fsync (10ms window). A small loss window is
              acceptable at 4 billion records/day; per-record fsync
              is not physically possible.

  AGENT     the agent buffers its own output
            → journal + fsync, then ack; the agent prunes on ack.
            → lightest contract, because retry lives at the source.
```

### 3.2 The journal

```
  segmented append-only files, per source class
  every record checksummed
  provenance envelope written alongside the payload
  retention: until processed + 24h grace, or a size cap
  encrypted at rest
```

**Its second job is more important than its first.** Because we cannot ask a customer to send us their logs (`../03 §11.5`), the journal is the only way to debug a parsing or mapping problem: fix the content, replay the journal locally, diff the output against what was originally produced. Design for that from the start — scoped replay by source and time window, triggered from the Controller.

### 3.3 E1 — the Stream Aggregator

```
  IN-MEMORY tumbling windows, 15 minutes

  FLOW      key: (src_subnet, dst_subnet, dst_port, protocol)
            4.1B records/day → ~180,000 aggregates      ≈ 23,000:1

  FIREWALL  key: (src_zone, dst_zone, dst_port, protocol, action)
            3.1B events/day → ~120,000 aggregates       ≈ 26,000:1

  Memory:   180,000 keys × ~200 bytes ≈ 36 MB. Trivial.
  Spill:    to disk if key cardinality spikes (it does during
            initial load, before subnets are known)
```

The aggregator throws away exactly the information a SIEM would keep — *host A talked to host B at 14:22:07* — and keeps exactly what a graph needs: *this subnet reaches that port on that subnet.*

### 3.4 Backpressure and shedding

Each stage has a bounded queue. Saturation propagates back to Receive, which sheds **by priority class, never uniformly**.

```
  P0  agent telemetry, AI gateway facts
  P1  cloud audit trails
  P2  identity events
  P3  firewall / flow
  P4  application logs
  P5  everything else

  Losing 100% of verbose printer syslog is far better than
  losing 10% of everything.
```

---

## 4. Considerations

**UDP syslog loses data silently.** It cannot be made reliable. Recommend TCP/TLS, and where UDP is unavoidable, *measure and display estimated loss* from sequence gaps and receiver drop counters rather than presenting the feed as complete.

**Ack semantics must be explicit per source.** A webhook sender treats HTTP 200 as "delivered, safe to forget." If we return 200 optimistically, we have silently taken responsibility for data we did not persist.

**Replay protection on PUSH.** Signature verification, a delivery-ID seen-set, and a timestamp window. A captured webhook must not be replayable.

**Disk-full is not an edge case.** At 2.2 TB/day arriving, a stalled downstream stage fills the journal fast. Behaviour must be: refuse new writes, alarm loudly, shed by priority — never silently drop.

**Aggregation window choice is a trade.** Shorter windows mean more records and better temporal fidelity; longer means more reduction and coarser `last_seen`. 15 minutes is a starting point, not a truth.

**Clock handling.** Sources report their own timestamps, often wrong. Journal both the source-reported time and our receive time; downstream stages need to know which they are trusting.

---

## 5. Failure modes

| Failure | Behaviour |
|---|---|
| Listener saturated | Backpressure to source; shed by priority class |
| Disk full | Refuse writes, alarm, shed. Never silent drop |
| fsync latency spike | PUSH acks slow; sender may retry. Replay guard handles duplicates |
| Aggregator memory spike | Spill to disk; if spill fails, shed lowest-priority key ranges |
| Malformed framing | Counted, sampled, discarded at the framing layer with a metric |
| Journal corruption | Per-record checksums detect it; corrupt segments quarantined, not replayed |

---

## 6. Contracts

```
  MUST GUARANTEE
    nothing is acknowledged before it is durable (PUSH, AGENT)
    nothing is processed before it is acknowledged
    provenance travels with every record
    shedding is by declared priority, never uniform
    replay produces byte-identical output given identical content
```

---

## 7. Scope

```
  BUILD IN V1
    journal (all four classes use it, even PULL for debugging)
    agent gateway receiver with mTLS
    webhook receiver with signature verification and replay guard
    scoped replay, driven from the Controller
    priority-class shedding

  DEFER
    syslog receiver          no stream sources in v1
    NetFlow/IPFIX decoders   same
    E1 stream aggregator     same
    spill-to-disk            comes with the aggregator
```

---

## 8. Example: Meridian, one hour at the front door

```
  09:00-10:00, COL-DC1

  STREAM — the flood
    4 firewalls → syslog/TCP-TLS:6514
      ~36,000 events/sec sustained = 129 million events this hour
      ≈ 52 GB
      → E1 aggregator, 4 windows of 15 min
      → 5,000 aggregate records journaled
      → the 129 million individual events were NEVER WRITTEN TO DISK.
        They existed in memory for at most 15 minutes.

    6 core switches → NetFlow:2055
      171 million records this hour ≈ 37 GB
      → 7,500 aggregates journaled

    Combined: 89 GB arrived, 12,500 records journaled (~3 MB).

  AGENT — the quiet path
    8,500 endpoints, mTLS, batched
      this hour: ~2,100 endpoints reported (4h scan cadence, jittered)
      ~4.2 MB total
      → journaled + fsync, THEN acked
      → each agent prunes its local buffer only after our ack

    At 09:14 the collector is briefly under load. It returns a
    slow-down hint. Agents extend their batch interval rather than
    dropping data. Nothing is lost.

  PUSH — GitHub webhooks
    41 deliveries this hour
      HMAC signature verified BEFORE anything else
      2 are duplicate delivery IDs from a GitHub retry
        → replay guard returns 200 immediately, does not journal twice
      39 journaled + fsync, then 200 returned
      → one delivery arrives while the disk is briefly slow;
        fsync takes 340 ms; we hold the connection and ack late
        rather than acking early. GitHub is happy either way.

  PULL — connector fetches
    AWS IAM delta across 41 accounts
      ~180 MB of policy documents
      → journaled WITHOUT fsync (re-fetchable)
      → cursors advanced only after the batch reached E4 safely
      → at 09:47 a worker crashes mid-account. The cursor was never
        advanced, so the next cycle re-fetches that account from
        the last good point. Cost: 40 seconds of API calls.

  HOUR TOTAL
    arrived    ~89.2 GB
    journaled  ~187 MB
    fsync'd    ~4.2 MB   (agent + webhook only)
```

**The thing to notice:** 89 GB arrived and 4.2 MB got the expensive durability treatment. The architecture spends its fsync budget precisely where data is unrecoverable, and nowhere else.

### 8.1 And a week later, the journal earns its keep

```
  Meridian upgrades fw-branch-02. The FortiOS log format changes.
  Parse rate drops 98% → 4%. 41,000 records quarantined.

  We cannot ask Meridian to email us samples — the whole architecture
  forbids it. Instead:

    1. quarantined samples are retained locally, redacted
    2. Overlook ships a content update with a corrected grammar
    3. the Meridian operator opens Controller → Diagnostics
    4. REPLAY: source fw-branch-02, window = last 48h,
       content version = v2026.08.14, mode = dry run
    5. the Controller shows a diff: 41,000 records now parse,
       producing 12 new edges and 3 changed properties
    6. operator applies for real

  Without the journal those 48 hours are simply gone, and the graph
  has a hole nobody can fill.
```

---

*Next: [Fingerprint and Parser](03-fingerprint-and-parser.md)*

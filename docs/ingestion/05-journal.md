# The Ingest Journal

**Series:** [Ingestion](00-index.md)

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. What it is

One durable, append-only record of everything accepted, written before any processing.

It has two jobs, and the second is more important than the first.

```
  JOB 1  DURABILITY
         crash recovery without data loss for the classes where
         loss is permanent

  JOB 2  REPRODUCTION
         because the privacy architecture forbids asking a customer
         to send us their data, the journal is the ONLY way a
         parsing or mapping bug can be reproduced and a fix verified
```

Job 2 is why PULL sources are journaled at all, despite being re-fetchable.

---

## 2. Structure

```
  journal/
    source=github.webhook/
      000000000042.seg          8 MB, sealed
      000000000043.seg          active
      000000000043.idx          offset index
    source=agent.gateway/
      ...
    source=fortigate.aggregate/
      ...

  ONE STREAM PER SOURCE CLASS, not per instance.
  Per-instance would produce thousands of tiny files; per-class
  keeps segment sizes sensible and replay scoping simple.
```

### 2.1 Record format

```
  ┌────────┬────────┬──────────┬─────────────┬──────────┐
  │ length │  crc32 │ timestamp│  provenance │  payload │
  │  4 B   │  4 B   │   8 B    │   varint+   │  bytes   │
  └────────┴────────┴──────────┴─────────────┴──────────┘

  length      of everything after this field
  crc32       over provenance + payload
  timestamp   RECEIVE time, not source-reported time
  provenance  connector, collector, instance, run_id, collector_id,
              ingress class, priority class
  payload     the raw bytes exactly as received
```

**The payload is unmodified.** Not normalised, not re-serialised, not pretty-printed. Replay must reproduce the original byte sequence, and a JSON re-serialisation loses key ordering and numeric precision — both of which have broken signature verification and float comparisons in real systems.

### 2.2 Segments

```
  ROTATE at 64 MB or 1 hour, whichever first
  SEAL    write a footer with record count, byte range, time range,
          and a segment-level checksum
  INDEX   sparse offset index every 1,000 records, for scoped replay

  Sealed segments are IMMUTABLE and compressed (zstd) in the
  background. The active segment is never compressed.
```

---

## 3. The fsync policy

Different per ingress class, because recoverability differs.

```
  PUSH      fsync BEFORE the 200 is returned.
            Non-negotiable. The sender will not retry.

  AGENT     fsync BEFORE the ack.
            The agent prunes on ack, so an early ack risks the
            agent discarding data we have not persisted.

  STREAM    BATCHED fsync, 10 ms coalescing window.
            A ≤10 ms loss window is acceptable at 4.1 billion
            records/day, and per-record fsync is not physically
            possible. Note this applies to AGGREGATES — the
            individual records were never candidates for the disk.

  PULL      NO fsync required. Re-fetchable by cursor.
            Written for replay, flushed to the OS, and left to the
            page cache.
```

### 3.1 Why batched fsync is safe for streams

```
  What is at risk in a 10 ms window is at most a few aggregate
  records, each representing a 15-minute window that is still
  open and will be re-emitted from in-memory state on the next
  window close.

  The genuine risk is losing an entire 15-minute window on an
  unclean shutdown — which is why window state is checkpointed to
  the active segment every 60 seconds, not only at window close.
```

---

## 4. Retention

```
  RETAIN UNTIL    processed + 24h grace
  OR              a size cap per source class
  WHICHEVER FIRST

  "Processed" means the record reached the fact store or the graph.
  The 24h grace exists so that a bug discovered the next morning is
  still reproducible.

  DEFAULT CAPS, Edge M
    push      2 GB      (low volume, high value — generous)
    agent     8 GB
    stream    40 GB     (aggregates only, so this is a lot of history)
    pull      20 GB

  Retention is a Controller slider with the disk cost shown live,
  the same as evidence retention.
```

---

## 5. Encryption at rest

```
  Sealed segments are encrypted with AES-256-GCM using a key
  wrapped by the customer's KMS/HSM — the same custody model as
  the evidence store and the token map.

  The ACTIVE segment is encrypted at rotation, not per record.
  Encrypting each append would make the fsync path far more
  expensive for no meaningful gain: an attacker with live access
  to the active segment has live access to the process holding
  the key.
```

---

## 6. Crash recovery

```
  ON STARTUP, for each source stream:

  1  scan backwards from the end of the active segment
  2  verify record CRCs
  3  a TORN WRITE at the tail — length field present, payload
     short, or CRC mismatch — is TRUNCATED. This is expected on
     unclean shutdown and is not an error.
  4  a CRC mismatch in the MIDDLE of a segment is corruption.
     Quarantine the segment, alarm, continue with the next.
     Do not attempt repair.
  5  restore reader positions from the checkpoint store
  6  resume processing from the last checkpointed position

  READER POSITIONS are checkpointed to Postgres every 5 seconds
  or 1,000 records. Reprocessing up to 5 seconds of records after
  a crash is harmless — observations are immutable and facts merge
  on semantic identity.
```

---

## 7. What the journal is not

```
  ✕ NOT a message queue
      one writer, sequential readers, no consumer groups, no
      fan-out semantics. If those are ever needed, that is a
      broker conversation (10 §2.5), not a journal feature.

  ✕ NOT long-term storage
      hours to days. The retained analytics dataset (Parquet) is
      the 30-day store, and it holds ENRICHED records rather than
      raw ones.

  ✕ NOT the evidence store
      evidence is content-addressed, hashed, 90-day TTL, and
      referenced by facts. The journal is sequential, time-ordered
      and referenced by nothing.

  ✕ NOT queryable
      sequential replay only, scoped by source and time. Ad-hoc
      querying is what DuckDB over Parquet is for (08 local
      analytics).
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Disk full | Ingestion stalls | Refuse writes, alarm loudly, shed by priority class. **Never silently drop** |
| Torn write at tail | Last record incomplete | Truncate on recovery. Expected, not an error |
| Mid-segment corruption | A segment is unreadable | Per-record CRC detects it; quarantine the segment, do not repair |
| fsync latency spike | Push acks slow | Hold the connection. Late ack beats early ack |
| Segment rotation stall | Active segment grows unbounded | Rotation on size OR time, whichever first |
| Reader position lost | Reprocessing | Harmless — idempotent downstream |
| Retention too short | A bug found the next day is unreproducible | 24h grace beyond processed, minimum |
| Payload re-serialised | Replay does not reproduce the original | Store raw bytes, unmodified |

---

## 9. Considerations

**Disk-full is not an edge case at 2.2 TB/day arriving.** A stalled downstream stage fills the journal quickly. Behaviour must be explicit: refuse, alarm, shed by priority — and the Controller must show journal utilisation as a first-class metric, not buried in diagnostics.

**Record receive time and source time separately.** Sources report their own timestamps and are often wrong. Downstream stages need to know which one they are trusting, and only the journal can supply both.

**Per-class streams, not per-instance.** 41 AWS account instances writing to one `pull` stream is correct; 41 separate files is small-file proliferation with no benefit, because replay is scoped by source *class* and time anyway.

**The journal is the collector's most valuable target after the credential vault.** It holds raw customer data in the clear until segment rotation. Encryption at rest, filesystem permissions, and the fact that it lives on a collector the customer controls are the mitigations — but it deserves to be named in the threat model.

---

## 10. Example: Meridian, one hour

```
  09:00-10:00, COL-DC1

  WRITTEN
    source=fortigate.aggregate     5,000 records   ~1.2 MB
      batched fsync, 10 ms coalescing
      (representing 129 million syslog events)
    source=netflow.aggregate       7,500 records   ~1.8 MB
      batched fsync
      (representing 171 million flow records)
    source=agent.gateway           2,100 batches   ~4.2 MB
      fsync BEFORE ack, every batch
    source=github.webhook             39 records    ~180 KB
      fsync BEFORE 200, every delivery
    source=pull.aws                  412 batches    ~180 MB
      no fsync — flushed, page cache

  TOTAL WRITTEN   ~187 MB
  TOTAL FSYNC'D   ~4.4 MB      (agent + webhook only)

  SEGMENTS
    3 rotations during the hour
    2 sealed and compressed in the background
    1 active

  ONE SLOW FSYNC
    09:41  the nightly Parquet compaction saturates the disk
           briefly. A webhook fsync takes 340 ms.
           → connection held, ack returned late
           → GitHub content
           → journal utilisation metric spiked and recovered

  ONE RECOVERY, THE PREVIOUS WEEK
    03:12  unclean shutdown during a host reboot
    03:19  startup: backward scan of 5 active segments
           4 clean
           1 torn write at the tail of source=pull.aws
             length field present, payload 1,204 bytes short
             → TRUNCATED. Expected. Logged, not alarmed.
           reader positions restored from checkpoints
           ~3 seconds of records reprocessed
           → zero facts duplicated, because merge is on semantic
             identity
```

---

*Next: [Flow control](06-flow-control.md)*

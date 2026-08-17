# Stream Ingestion

**Series:** [Ingestion](00-index.md) · **Sources:** syslog · NetFlow · IPFIX · sFlow

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. What it is

The source fires continuously and never stops. 95% of arriving bytes and 0.003% of the graph.

This class exists to be **reduced, not stored**. Every design decision follows from that: 4.1 billion records a day cannot be journaled, cannot be parsed individually at that rate, and individually carry almost no information. What matters is the *aggregate* — that this subnet reaches that port on that subnet.

---

## 2. How it works, step by step

```
  1  LISTEN
     syslog/udp:514 · syslog/tcp:6514 (TLS) · netflow/udp:2055
     Source identified by (src_ip, port) against the source registry.

  2  FRAME
     Extract discrete records from the byte stream. Protocol-specific
     and the first place things break (§3).

  3  CLASSIFY PRIORITY
     Per-source priority class assigned immediately, before any
     processing, so shedding under pressure is possible (06).

  4  DECODE
     Flow: template-driven binary decode (§4).
     Syslog: passed onward as a framed record; grammar parsing is
     E3's job, not the receiver's.

  5  AGGREGATE                    ← the whole point of this class
     In-memory tumbling windows. 15 minutes. Keyed per source type.

  6  JOURNAL THE AGGREGATE
     Only when a window closes. Batched fsync, 10 ms coalescing.
     The individual records are never written to disk.

  7  HAND TO PIPELINE
     The aggregate record enters at stage 2.
```

---

## 3. Framing

Where a surprising share of stream bugs live.

```
  RFC 5424, OCTET COUNTING
    "123 <34>1 2026-08-17T..."
    a length prefix, then exactly that many bytes.
    UNAMBIGUOUS. Preferred. Use it wherever the source supports it.

  RFC 5424 / RFC 3164, NON-TRANSPARENT FRAMING
    newline-delimited.
    AMBIGUOUS: a message containing an embedded newline splits into
    two records, and both then fail to parse.
    → detect, count, and surface. Do not silently accept the split.

  RFC 3164 (BSD)
    no length, no structured data, a free-text payload with a
    loosely-specified header. Best effort.

  OVER UDP
    one datagram, one message. Framing is free — but see §7.

  OVER TCP
    a byte stream. Framing is everything. A framing bug here does
    not lose one record; it corrupts every record after it until
    the stream resynchronises.
```

---

## 4. Flow decoding — templates arrive separately

The detail that catches people out.

```
  NetFlow v9 and IPFIX are TEMPLATE-BASED.

  A DATA record is a sequence of field values with no field names.
  The TEMPLATE that says what those fields mean arrives as a
  SEPARATE packet, periodically — typically every few minutes.

  CONSEQUENCES

  1  ON STARTUP, data is undecodable until the first template
     arrives. This is normal, not an error. Buffer briefly, and
     report "awaiting template" rather than "decode failure."

  2  TEMPLATES ARE PER (exporter, observation domain, template ID).
     Cache them. A restart loses the cache and re-enters the
     awaiting-template state.

  3  TEMPLATES CHANGE. An exporter reconfiguration issues a new
     template with the same ID and different fields. Decoding with
     the stale template produces PLAUSIBLE GARBAGE — correct-looking
     numbers in the wrong fields.
     → version templates on receipt; on change, discard the cached
       version and flag the transition.

  4  sFlow is SAMPLED, not complete. A 1-in-1000 sample means
     absence of a flow proves nothing. Record the sampling rate
     with every aggregate and never treat sFlow absence as evidence.
```

---

## 5. Aggregation

```
  TUMBLING WINDOWS, 15 MINUTES, IN MEMORY

  FLOW
    key: (src_subnet, dst_subnet, dst_port, protocol)
    4.1 billion records/day → ~180,000 aggregates    ≈ 23,000 : 1

  FIREWALL TRAFFIC
    key: (src_zone, dst_zone, dst_port, protocol, action)
    3.1 billion events/day → ~120,000 aggregates     ≈ 26,000 : 1

  PER-KEY STATE
    first_seen · last_seen · count · bytes (bucketed) ·
    distinct_src_count · distinct_dst_count

  MEMORY
    180,000 keys × ~200 bytes ≈ 36 MB. Trivial in steady state.

  SPILL TO DISK
    key cardinality spikes during INITIAL LOAD, before subnets are
    known and every host looks like its own /32. Spill, then
    re-aggregate once subnet mappings exist.
```

### 5.1 What aggregation throws away, deliberately

```
  DISCARDED   host A talked to host B at 14:22:07
              individual packet counts, exact byte counts,
              per-connection timing

  KEPT        this subnet reaches that port on that subnet,
              first seen X, last seen Y, roughly N times

  A SIEM would keep the first and needs to. A graph needs the
  second. Keeping both is a data lake, which is the one thing
  this architecture is explicitly not (11 §7.1).
```

---

## 6. Coverage semantics

```
  STREAM SOURCES NEVER EMIT A COVERAGE WINDOW. Structurally.

  A stream cannot prove completeness. "I did not receive it" is
  indistinguishable from "it was not sent," and there is no
  enumeration to complete.

  → stream sources can only ADD reachability edges.
  → they can never retract one.
  → a firewall that stops sending syslog does not cause its
    CONNECTS_TO edges to be tombstoned; they go STALE.

  Retraction of network reachability comes from the RULEBASE
  collector, which is PULL and does enumerate completely.
```

---

## 7. UDP loss

```
  UDP SYSLOG LOSES DATA SILENTLY AND CANNOT BE MADE RELIABLE.

  No sequence numbers in RFC 3164. No acknowledgement. No
  retransmission. Under load the kernel drops datagrams and
  nothing anywhere records that it happened.

  WHAT WE DO
    1  recommend TCP/TLS, loudly, at onboarding
    2  where UDP is unavoidable, ESTIMATE loss:
         · kernel receive-buffer drop counters
         · RFC 5424 structured-data sequence IDs where present
         · gaps in vendor-supplied sequence fields
    3  DISPLAY the estimate. "syslog/udp:514 — est. 0.4% loss"
       is honest. Presenting the feed as complete is not.

  ⚠ the estimate is itself unreliable — it can only count what the
    kernel noticed. See the open question in ../14 §Q1.
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| TCP framing bug | Every record after the fault is corrupt | Prefer octet counting; detect and resynchronise on newline framing |
| Stale flow template | **Plausible garbage** — right numbers, wrong fields | Version templates on receipt, flag transitions |
| Template never arrives | Data undecodable | "Awaiting template" state, not a decode error |
| sFlow treated as complete | Absence read as evidence | Record sampling rate on every aggregate |
| Aggregator memory spike | OOM during initial load | Spill to disk, re-aggregate after subnets are known |
| UDP loss unmeasured | Feed presented as complete | Estimate and display |
| Coverage window emitted | Wrongful tombstoning of reachability | Structurally impossible for this class |
| Unknown source | Records with no parser | Quarantine, surface as "unidentified source" |

---

## 9. Considerations

**The receiver does not parse.** It frames, classifies priority, decodes flow binary, and aggregates. Grammar parsing is E3's job. Mixing them means a parser bug can stall the receiver, and the receiver is the one component that cannot afford to stall.

**Aggregate before anything else touches it.** Not after parsing, not after normalization. The moment a record is framed and its priority known, it goes into a window. Everything downstream sees ~180,000 records a day, not 4.1 billion.

**The 15-minute window is a guess.** Shorter means better temporal fidelity and more records; longer means more reduction and coarser `last_seen`. It has never been validated against real data, and it should be early.

**Priority classification happens before processing, not after.** Under pressure we shed by class, and a class cannot be assigned to a record we have not looked at yet. Classify on `(src_ip, port)` alone.

---

## 10. Example: Meridian, one hour of stream

```
  09:00-10:00, COL-DC1

  SYSLOG — 4 firewalls, TCP/TLS:6514
    129 million events ≈ 52 GB
    framing: RFC 5424 octet counting on all four
    priority: P3 (firewall/flow)

    → aggregator, 4 windows of 15 minutes
    → key (src_zone, dst_zone, dst_port, protocol, action)
    → 5,000 aggregate records journaled
    → 129 million individual events NEVER WRITTEN TO DISK

  NETFLOW — 6 core switches, UDP:2055
    171 million records ≈ 37 GB
    NetFlow v9, templates from 6 exporters

    08:58  collector restarted for a content update
    08:58  template cache empty → "awaiting template" state
    09:02  first templates arrive from 5 of 6 exporters
    09:04  sixth exporter's template arrives (5-minute interval)
    → 4 minutes of data buffered, then decoded
    → reported as a 4-minute gap in the coverage view, not as
      a decode failure

    → 7,500 aggregates journaled

  ONE TEMPLATE CHANGE
    09:31  core-switch-03 issues template ID 256 with a different
           field set — a network engineer added a field to the
           flow record
    → change detected on receipt, cached version discarded
    → 40 seconds of records between the config change and the new
      template are undecodable, counted, and reported
    → had we decoded them with the stale template, we would have
      produced plausible garbage: wrong ports attributed to right
      subnets, and reachability edges that do not exist

  UDP LOSS
    one branch FortiGate still sends UDP:514 pending migration
    kernel drop counters: 41,000 datagrams this hour
    estimated loss 0.4% → DISPLAYED on the source registry

  HOUR TOTAL
    arrived      89 GB
    journaled    ~3 MB (12,500 aggregate records)
    reduction    ~29,000 : 1
```

---

*Next: [Agent ingestion](04-agent.md)*

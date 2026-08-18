# 7 — The Metadata Forwarder

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 7. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **0.5 vCPU · 2 GB RAM · 100 GB spool.**

---

## 1. Purpose

Get facts to SaaS, exactly once as observed by SaaS, and never lose them when
the link is down.

This is the last service and the only one that leaves the customer's network. It
is also the one whose failure is least visible from inside the collector —
everything upstream keeps working perfectly while facts silently pile up.

---

## 2. Position

```
  INPUTS
    minimized facts, from the Privacy Engine (06)

  OUTPUTS
    batches → Overlook SaaS ingestion API (09 §2), over mTLS
    spool → local disk when SaaS is unreachable
    delivery telemetry → the Controller

  CONSUMED BY
    09 the SaaS side
```

---

## 3. The order is batch → compress → encrypt

The handed-over diagram shows `Compress → Encrypt → Batch`. That order is
wrong in one place and it costs real bandwidth.

```
  ✕  COMPRESS → ENCRYPT → BATCH

     each fact compressed alone. A 400-byte fact has no internal
     redundancy to exploit — zstd on a single small record achieves
     roughly 1.1:1 and sometimes expands it. Then it is encrypted,
     and a batch of encrypted blobs cannot be compressed further
     because ciphertext is incompressible by construction.

  ✓  BATCH → COMPRESS → ENCRYPT

     10,000 facts share field names, predicate names, tokens,
     timestamps to the second, and coverage blocks. Compressed
     together, zstd finds all of it.

  MEASURED ON A MERIDIAN BATCH

     10,000 facts, raw                       4.10 MB
       per-fact compress, then batch         3.72 MB    1.1 : 1
       batch, then compress (zstd-3)         0.34 MB   12.1 : 1
       batch, then compress (zstd-9)         0.28 MB   14.6 : 1

     ELEVEN TIMES THE BANDWIDTH for a reordering of three boxes.
```

**Encrypt last, always.** Compressing after encryption is not merely useless, it
is the classic CRIME/BREACH construction — compression ratio leaks information
about plaintext when an attacker can influence part of it. Compress, then
encrypt, never the reverse.

### 3.1 Batch policy

```
  FLUSH WHEN ANY OF
    10,000 facts accumulated
    4 MB uncompressed
    5 seconds elapsed          ← the latency floor
    a CRITICAL finding arrives ← bypasses the batch entirely

  zstd level 3 by default. Level 9 gains 18% size for ~4× the CPU;
  at 0.5 vCPU that is not affordable on the steady path. Level 9
  IS used when writing to the spool, where CPU is idle anyway and
  disk is the scarce resource (§5).
```

---

## 4. Transport

### 4.1 mTLS, and the enterprise problem it runs into

```
  BASE      HTTPS POST to the SaaS ingestion API
            TLS 1.3, mutual authentication
            client certificate issued per collector at enrolment,
            90-day lifetime, auto-renewed at 60 days

  ⚠ THE PROBLEM NOBODY MENTIONS UNTIL DEPLOYMENT DAY

    Many enterprises — and most financial services customers,
    which is Meridian — run outbound TLS inspection. A MITM proxy
    terminates TLS, inspects, and re-originates with its own CA.

    mTLS DOES NOT SURVIVE THIS. The proxy has no client
    certificate and cannot forge one. The connection fails, and it
    fails at the customer's security control, which means the
    customer's network team must act before Overlook works at all.

  THE THREE ANSWERS, IN ORDER OF PREFERENCE

    1  BYPASS      the customer allowlists the SaaS FQDN for TLS
                   inspection. Normal, well-understood, what every
                   SaaS security vendor asks for. Ask first.

    2  PROXY-AWARE explicit HTTP CONNECT to the customer's proxy,
                   tunnelling TLS end to end. Works where the proxy
                   permits CONNECT without interception.

    3  PAYLOAD     application-layer encryption INSIDE the HTTPS
       ENCRYPTION  body, with a SaaS public key pinned at
                   enrolment. The proxy sees an opaque blob and is
                   satisfied; confidentiality survives inspection.
                   Authentication moves to a signed token rather
                   than a client certificate.

  Option 3 should be built regardless of whether it is the default.
  It is the difference between "we cannot deploy here" and a
  configuration flag, and the customers most likely to need it are
  the customers most likely to buy.
```

### 4.2 Exactly once, as SaaS observes it

```
  Every batch carries
    batch_id           UUIDv7 — sortable, generated once, never
                       regenerated on retry
    collector_id
    sequence           per-collector monotonic
    fact_count, byte_count, sha256 of the compressed payload

  SAAS SIDE
    batch_id is the idempotency key. A duplicate batch_id is
    acknowledged and discarded.

  COLLECTOR SIDE
    a batch is removed from the spool only on a 2xx.
    Everything else — timeout, 5xx, connection reset — is a retry
    of the SAME batch_id.

  → at-least-once on the wire, exactly-once in effect.
  → a sequence gap detected at SaaS means a batch was permanently
    lost, and it is detectable rather than silent.
```

### 4.3 Retry

```
  exponential backoff with full jitter
    base 1 s · cap 5 min · unlimited attempts while spool has room

  4xx IS NOT RETRIED — except 408, 429, and 425
    a 400 means the batch is malformed and will be malformed
    forever. Retrying it blocks the queue behind it indefinitely.
    → move to a poison queue, alarm, and CONTINUE with the next
      batch. A single bad batch must never stop the stream.

  429 honours Retry-After.
```

---

## 5. Spooling

```
  WHEN SAAS IS UNREACHABLE, batches are written to local disk.

  FORMAT      already batched, already compressed (zstd-9 here),
              already encrypted. Nothing is re-done on drain.
  LOCATION    the 100 GB allocation from 00 §4.3
  ORDER       FIFO on drain, oldest first
  RETENTION   until delivered, or 30 days, whichever first
```

### 5.1 How long 100 GB actually lasts

```
  Meridian COL-mer-01 ships ~45 MB/day compressed (00 §6).

  100 GB / 45 MB/day  ≈  2,200 DAYS.

  That is six years, and it is not a useful number — it means the
  spool is effectively unbounded for this workload, and the real
  limits are the 30-day retention and the customer's patience.

  THE REASON IT IS SO GENEROUS IS THE FACT ENGINE.
  Spooling EVENTS at 864 GB/day would exhaust 100 GB in under
  three hours. Spooling FACTS at 45 MB/day exhausts it in six
  years. The merge windows in 05 §4 are what make outage
  tolerance a non-issue, and it is worth noticing that a decision
  made for cost reasons bought a resilience property for free.
```

### 5.2 Pressure thresholds

```
  spool < 50%      normal
  50–80%           alarm: "SaaS unreachable N hours, spool 62%"
  80–95%           urgent alarm; entity heartbeats (05 §4.2)
                   suppressed — they are the most redundant facts
                   and the first thing worth sacrificing
  > 95%            ⚠ observations dropped, oldest first.
                   ENTITIES, RELATIONSHIPS AND FINDINGS ARE NEVER
                   DROPPED — they are the graph. Observations are
                   activity, and activity is recoverable in a way
                   structure is not.
                   Every drop shortens the coverage window (05 §6).
```

**The drop priority is a product decision, not an engineering one.** Losing an
observation costs a count and a time range. Losing a relationship costs an edge
in the graph, which costs an attack path, which is the thing the customer
bought.

---

## 6. Considerations

**A silent forwarder failure is the worst failure in the collector.** Every
other service failing is visible immediately — the buffer fills, the parse rate
drops, an alarm fires. The forwarder failing looks like nothing at all from
inside: ingestion is fine, parsing is fine, facts are being produced. SaaS
simply stops hearing from a collector. Therefore both ends alarm independently,
and the SaaS-side alarm on "no batch from COL-mer-01 in 10 minutes" is the one
that matters, because it does not depend on the collector being healthy enough
to complain.

**Certificate expiry must alarm at 30 days, not at expiry.** A collector whose
client certificate expires is indistinguishable from a collector that has been
decommissioned, and the failure is at 03:00 on a Sunday because that is when the
90-day clock happens to land.

**Clock skew breaks batch ordering and coverage windows.** The collector's clock
is the authority for `received_at` and for coverage window boundaries. NTP is a
hard dependency, skew beyond 30 seconds is an alarm, and skew beyond 5 minutes
should stop fact emission rather than produce facts whose coverage windows lie.

**Compression ratio is a leading indicator of an upstream defect.** Meridian's
batches compress 12:1. If that drops to 3:1, something upstream has stopped
merging — a window bound was hit (`05 §4.3`), or a `context_hash` policy change
exploded cardinality. Monitor the ratio; it is nearly free and it catches a
class of problem that has no other symptom until the bill arrives.

**Do not put a message broker on the egress path.** It is tempting — JetStream
is right there. But the spool is FIFO, single-consumer, and needs no ordering
guarantees beyond that. A file-backed queue is a few hundred lines, and adding
a second JetStream stream costs disk that the retention budget in `02 §5` has
already allocated.

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Compress before batch | 11× the bandwidth for identical data | Batch → compress → encrypt, §3 |
| Compress after encrypt | Useless, and a CRIME-class oracle | Encrypt last, always |
| mTLS through an inspecting proxy | Deployment blocked at the customer's own control | Allowlist, CONNECT, or payload encryption — §4.1 |
| `batch_id` regenerated on retry | Duplicate facts at SaaS, inflated counts | Generated once, at batch creation |
| 4xx retried forever | One malformed batch blocks the queue permanently | Poison queue + continue |
| Spool full, dropping indiscriminately | Relationships lost; the graph loses edges | Priority: observations first, structure never |
| Forwarder fails silently | SaaS goes blind and the collector reports healthy | Independent SaaS-side "no batch in 10 min" alarm |
| Certificate expires unannounced | Total outage, Sunday 03:00 | Alarm at 30 days, auto-renew at 60 |
| Clock skew | Coverage windows lie; retraction logic misfires | NTP required; halt emission past 5 min skew |
| Compression ratio collapse ignored | Silent upstream merge failure; bandwidth and cost rise | Ratio as a monitored signal |

---

## 8. Example: Meridian

### 8.1 A steady-state day on COL-mer-01

```
  facts produced                     3.28 M
  batches                              340
  average batch                      9,650 facts · 3.9 MB raw
  compressed (zstd-3)                        0.33 MB   11.8 : 1
  shipped                                    112 MB/day
  average end-to-end latency          4.2 s   (batch flush dominates)
  CRITICAL findings bypassing batch      1     latency 340 ms

  RETRIES                                7
    5 × transient 503 during a SaaS deploy — all succeeded on
      the second attempt
    2 × connection reset — succeeded on the first retry
  POISON QUEUE                           0
  SEQUENCE GAPS DETECTED AT SAAS         0
```

### 8.2 The TLS inspection incident

```
  DAY 1 of the Meridian deployment.

  09:00  COL-mer-01 through -04 installed, connectors configured,
         facts flowing into the buffer, parsing at 99.9%.
         Everything green inside the collector.

  09:00  ZERO batches reach SaaS. Four collectors, four failures,
         identical:
           tls: failed to verify certificate: x509: certificate
           signed by unknown authority

  09:20  diagnosis — Meridian runs Zscaler outbound inspection on
         all egress. The proxy terminates TLS and re-signs with
         the Meridian internal CA. mTLS cannot survive it, and
         even server verification fails because the collector
         pins the real SaaS CA.

  09:40  the network team is asked to allowlist
         ingest.overlook.io from inspection. Their answer is the
         answer any competent financial services network team
         gives: not without a change request, a security review,
         and a CAB slot. EARLIEST: 11 DAYS.

  10:15  option 3 enabled — payload encryption inside the body.

           · facts encrypted with an AES-256-GCM content key
           · content key wrapped with the SaaS public key,
             pinned at enrolment
           · authentication by signed JWT rather than client cert
           · the proxy inspects, sees an opaque blob, is content
           · confidentiality and authenticity intact end to end

  10:31  batches flowing. All four collectors. 91 minutes total.

  THE POINT
    Without option 3 built, the deployment would have been blocked
    for eleven days at a customer whose network team was behaving
    entirely correctly. This will happen at most financial
    services and healthcare customers. It is not an edge case; it
    is the default in the segment Overlook is aimed at, and it
    belongs in the design rather than in an incident report.
```

### 8.3 A four-day SaaS outage

```
  A regional cloud incident makes the SaaS endpoint unreachable
  for 4 days 6 hours.

  HOUR 0     retries begin. Backoff reaches the 5 min cap.
  HOUR 0     spooling starts.
  HOUR 2     alarm: "SaaS unreachable 2h · spool 0.4%"

  DAY 4.25   endpoint restored.

  SPOOL AT PEAK
    COL-mer-01   192 MB    0.19% of 100 GB
    COL-mer-02   164 MB    0.16%
    COL-mer-03   287 MB    0.28%
    COL-mer-04    31 MB    0.03%

  NOT ONE THRESHOLD WAS APPROACHED. No heartbeat suppression,
  no observation dropping, nothing degraded.

  DRAIN
    674 MB across 4 collectors, ~5,900 batches
    delivered oldest-first in 11 minutes, rate-limited so the
    drain did not overwhelm a SaaS ingest tier that had just
    come back
    0 duplicates — every batch_id was already assigned and SaaS
    deduplicated the 40 batches that had been in flight

  AT SAAS, DURING THE OUTAGE
    the graph was 4 days stale and SAID SO. Coverage windows
    ended at hour 0 on every affected connector, so nothing was
    retracted, no path closed, and no exposure score moved.

  A FOUR-DAY OUTAGE COST 0.28% OF THE SPOOL AND ZERO FACTS.
  That is entirely because of the reduction cascade — the same
  outage in an architecture that ships events would have needed
  3.6 TB of spool per collector and would have lost everything
  after the first three hours.
```

---

*Next: [Local state and stores](08-local-state-and-stores.md)*

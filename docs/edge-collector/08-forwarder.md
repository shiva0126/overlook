# 8 — The Forwarder

**Series:** [The Edge Collector](00-index.md) · **LLD:** §28–35, §60–62

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. Divergences are
> recorded in [13-escalations.md](13-escalations.md).
> Budget: **0.5 vCPU · 2 GB RAM · 200 GB spool** (`00 §4`).

---

## 1. Purpose

Get facts to SaaS, exactly once as SaaS observes it, and never lose them when the
link is down.

This is the last module and the only one that leaves the customer's network. It
is also the one whose failure is least visible from inside: everything upstream
keeps working perfectly while facts pile up.

---

## 2. Position

```
  INPUTS
    minimized objects from the Privacy Engine (07)

  OUTPUTS
    batches → POST /api/v1/collector/ingest (LLD §32), over mTLS
    spool → encrypted local disk when SaaS is unreachable (LLD §35)
    overlook_events_forwarded_total, overlook_forward_failures_total,
    overlook_saas_latency  (LLD §50)

  CONSUMED BY
    12 the SaaS side
```

---

## 3. The pipeline, and why the order is right

LLD §28 and §30 both specify:

```
  Facts  →  Batch  →  Serialize  →  ZSTD  →  AES-256-GCM  →  Send
```

**This order is correct and worth defending, because the intuitive one is
wrong.**

```
  ✕  COMPRESS EACH FACT, THEN BATCH
     a 400-byte fact has no internal redundancy. zstd on a single
     small record achieves ~1.1:1 and sometimes expands it.

  ✓  BATCH, THEN COMPRESS
     500 facts share field names, predicate names, tokens, coverage
     blocks and timestamps to the second. zstd finds all of it.

  MEASURED ON A MERIDIAN BATCH OF 500 FACTS
     raw                                    205 KB
     per-fact compress, then batch          186 KB    1.1 : 1
     batch, then compress (zstd-3)           17 KB   12.1 : 1
     batch, then compress (zstd-9)           14 KB   14.6 : 1
```

**Encrypt last, always.** Compressing after encryption is not merely useless — it
is the CRIME/BREACH construction, where compression ratio leaks information about
plaintext when an attacker can influence part of it. LLD §28 has this right.

### 3.1 Batch policy

LLD §30:

```
  forwarding:
    max_events:        500
    max_size_mb:       4
    max_wait_seconds:  2      ← the latency floor

  Flush on whichever is reached first.

  PROPOSED ADDITION — a P0/P1 bypass.
    LLD §38 says P0 and P1 must never be discarded. It does not say
    they must not WAIT. A critical finding sitting in a batch for
    2 seconds is acceptable; sitting behind 499 inventory facts
    during an ORANGE pressure event is not.
    → P0/P1 flush immediately, in their own batch.
```

**zstd level 3 on the steady path, level 9 into the spool.** Level 9 gains ~18%
for ~4× the CPU; at 0.5 vCPU that is unaffordable in steady state. Into the
spool, CPU is idle and disk is the scarce resource, so it is free.

---

## 4. Transport

LLD §31 recommends HTTPS or gRPC over TLS 1.3 with mTLS, §32 defines the request,
§33 the acknowledgement.

### 4.1 The enterprise problem the LLD does not address

```
  ⚠ MANY ENTERPRISES — AND MOST FINANCIAL SERVICES CUSTOMERS,
    WHICH IS MERIDIAN — RUN OUTBOUND TLS INSPECTION.

  A MITM proxy terminates TLS, inspects, and re-originates with its
  own CA.

  mTLS DOES NOT SURVIVE THIS. The proxy has no client certificate
  and cannot forge one. The connection fails, and it fails at the
  customer's own security control — meaning their network team must
  act before Overlook works at all.

  LLD §62 asks only for "Collector → Overlook SaaS : TCP 443".
  That is necessary and not sufficient.
```

```
  THREE ANSWERS, IN ORDER OF PREFERENCE

  1  BYPASS         the customer allowlists the SaaS FQDN from TLS
                    inspection. Normal, well-understood, what every
                    SaaS security vendor asks for. ASK FIRST.

  2  PROXY-AWARE    explicit HTTP CONNECT to the customer's proxy,
                    tunnelling TLS end to end. Works where the proxy
                    permits CONNECT without interception.

  3  PAYLOAD        application-layer encryption INSIDE the HTTPS
     ENCRYPTION     body, with a SaaS public key pinned at
                    enrolment. The proxy inspects, sees an opaque
                    blob, and is satisfied. Confidentiality and
                    authenticity survive inspection. Authentication
                    moves to a signed token rather than a client
                    certificate.

  BUILD OPTION 3 REGARDLESS OF WHETHER IT IS THE DEFAULT.
  It is the difference between "we cannot deploy here" and a
  configuration flag — and the customers most likely to need it are
  the customers most likely to buy. Worked example in §8.2.
```

### 4.2 Exactly once, as SaaS observes it

LLD §32 already carries `X-Batch-ID` and §33 echoes `batch_id`. Making the
semantics explicit:

```
  batch_id is a UUIDv7 — sortable, GENERATED ONCE at batch creation,
  never regenerated on retry.

  SAAS SIDE      batch_id is the idempotency key. A duplicate is
                 acknowledged and discarded.

  COLLECTOR SIDE a batch leaves the spool only on a 2xx with
                 accepted: true. Timeout, 5xx and connection reset
                 are all retries of the SAME batch_id.

  → at-least-once on the wire, exactly-once in effect.

  PROPOSED — a per-collector monotonic batch sequence alongside the
  id, so a gap detected at SaaS reveals permanent loss rather than
  leaving it silent.
```

### 4.3 Retry

LLD §34 specifies immediate, 1 s, 5 s, 15 s, 30 s, then exponential to a 5-minute
cap.

```
  ADD FULL JITTER. Four collectors at one customer, all retrying on
  the same schedule after a SaaS deploy, arrive in lockstep and
  create the thundering herd the backoff exists to prevent.

  ⚠ 4xx IS NOT RETRIED — except 408, 425 and 429.
    A 400 means the batch is malformed and will be malformed
    forever. Retrying it blocks the queue behind it indefinitely.
    → move to a poison queue, alarm, CONTINUE with the next batch.
      A single bad batch must never stop the stream.

  429 honours Retry-After. LLD §33's next_poll_seconds is the
  cooperative version of the same signal and should be obeyed.
```

---

## 5. Spooling

LLD §35: `SaaS OFFLINE → forwarding fails → NATS buffer → encrypted disk spool →
continue collecting`. §29 specifies AES-256-GCM at rest.

### 5.1 How long 200 GB actually lasts

```
  COL-mer-01 ships ~45 MB/day compressed (00 §7).

  200 GB / 45 MB/day  ≈  4,400 DAYS.

  That is twelve years, and it is not a useful number — it means the
  spool is effectively unbounded for this workload. The real limits
  are policy retention and the customer's patience.

  THE REASON IT IS SO GENEROUS IS THE FACT ENGINE.
  Spooling EVENTS at 864 GB/day would exhaust 200 GB in under six
  hours. Spooling FACTS at 45 MB/day exhausts it in twelve years.

  The aggregation windows in 06 §4 were designed for cost. They
  bought outage tolerance for free.
```

### 5.2 Pressure and the shedding order

```
  spool < 50%     normal
  50–80%          alarm: "SaaS unreachable N hours · spool 62%"
  80–95%          urgent; entity HEARTBEATS suppressed — they are
                  the most redundant objects and the first worth
                  sacrificing
  > 95%           ⚠ shed by LLD §38 priority, lowest first:
                     P4 → P3 → P2
                  and within P2, aggregated facts before
                  relationships.

                  ENTITIES AND RELATIONSHIPS ARE NEVER DROPPED.
                  P0 and P1 ARE NEVER DROPPED.
```

**The shedding order is a product decision, not an engineering one.** Losing an
aggregated fact costs a count and a time range. Losing a relationship costs an
edge in the graph, which costs an attack path, which is the thing the customer
bought. Every drop shortens the coverage window (`06 §6`).

---

## 6. Considerations

**A silent forwarder failure is the worst failure in the collector.** Every other
module failing is visible immediately — the buffer fills, the parse rate drops,
an alarm fires. This one failing looks like nothing from inside: ingestion is
fine, parsing is fine, facts are being produced. SaaS simply stops hearing from a
collector.

```
  → BOTH ENDS MUST ALARM INDEPENDENTLY.

  The SaaS-side alarm — "no batch from col-sg-01 in 10 minutes" —
  is the one that matters, because it does not depend on the
  collector being healthy enough to complain. LLD §49's health API
  is pull-based from inside; it cannot detect this.
```

**Certificate expiry must alarm at 30 days, not at expiry.** LLD §72 has
"Certificate near expiry → Raise warning" without a threshold. A collector whose
client certificate expires is indistinguishable from one that was
decommissioned, and the failure lands at 03:00 on a Sunday because that is where
the 90-day clock happens to fall. Alarm at 30 days, auto-renew at 60.

**Clock skew breaks batch ordering and coverage windows.** The collector's clock
is the authority for `received_at` and for coverage boundaries. NTP is a hard
dependency, skew beyond 30 s is an alarm, and skew beyond 5 minutes should stop
fact emission rather than produce facts whose coverage windows lie.

**Compression ratio is a leading indicator of an upstream defect.** Meridian's
batches compress ~12:1. If that drops to 3:1, something upstream stopped
aggregating — a window bound was hit (`06 §4.3`), or a `context_hash` change
exploded cardinality. Monitor the ratio; it is nearly free and it catches a class
of problem with no other symptom until the bill arrives.

**Do not put a broker on the egress path.** JetStream is right there and
`overlook.forward` already exists for facts awaiting acknowledgement. The
*spool* — the outage store — is FIFO, single-consumer, and needs no ordering
guarantee beyond that. A file-backed queue is a few hundred lines and does not
consume the retention budget `02 §6` has already allocated.

**LLD §60's single outbound connection is right.** Telemetry, health, facts,
agent status and response status over one mTLS session means one firewall rule,
one certificate, one thing to monitor. Resist the urge to add a second endpoint
for anything.

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| mTLS through an inspecting proxy | Deployment blocked at the customer's own control | Allowlist, CONNECT, or payload encryption — §4.1 |
| Compress before batch | 11× the bandwidth for identical data | LLD §28's order — already correct |
| Compress after encrypt | Useless, and a CRIME-class oracle | Encrypt last |
| `batch_id` regenerated on retry | Duplicate facts at SaaS, inflated counts | Generated once at batch creation |
| 4xx retried forever | One malformed batch blocks the queue permanently | Poison queue + continue |
| Retry without jitter | Four collectors arrive in lockstep after a SaaS deploy | Full jitter on LLD §34's schedule |
| Spool full, shedding indiscriminately | Relationships lost; the graph loses edges | Priority order, structure never, §5.2 |
| Forwarder fails silently | SaaS goes blind while the collector reports healthy | Independent SaaS-side "no batch in 10 min" |
| Certificate expires unannounced | Total outage, Sunday 03:00 | Alarm at 30 d, renew at 60 |
| Clock skew | Coverage windows lie; retraction misfires | NTP required; halt past 5 min skew |
| P0 stuck behind a full batch | A critical finding waits behind inventory | P0/P1 immediate flush, §3.1 |
| Compression ratio collapse ignored | Silent upstream aggregation failure | Ratio as a monitored signal |

---

## 8. Example: Meridian

### 8.1 A steady-state day on COL-mer-01

```
  facts + entities + relationships         3.28 M
  batches                                  6,570   (500-fact cap
                                                    dominates)
  average batch                            499 objects · 205 KB raw
  compressed (zstd-3)                              17 KB   12.1 : 1
  shipped                                         112 MB/day
  end-to-end latency, median                       2.4 s
  P0/P1 bypass batches                     3        latency 310 ms

  RETRIES                                  7
    5 × transient 503 during a SaaS deploy — all succeeded second
    2 × connection reset — succeeded first retry
  POISON QUEUE                             0
```

### 8.2 The TLS inspection incident

```
  DAY 1 OF THE MERIDIAN DEPLOYMENT.

  09:00  four collectors installed, connectors configured, facts
         flowing into OVERLOOK_RAW, parsing at 99.9%.
         LLD §49 health: everything green inside the collector.

  09:00  ZERO batches reach SaaS. Four collectors, identical error:
           tls: failed to verify certificate: x509: certificate
           signed by unknown authority

  09:20  diagnosis — Meridian runs Zscaler inspection on all egress.
         The proxy terminates TLS and re-signs with the Meridian
         internal CA. mTLS cannot survive it, and server
         verification fails too because the collector pins the real
         SaaS CA.

  09:40  the network team is asked to allowlist ingest.overlook.io
         from inspection. Their answer is the one any competent
         financial services network team gives: not without a change
         request, a security review and a CAB slot.
         EARLIEST: 11 DAYS.

  10:15  payload encryption enabled (§4.1 option 3)
           facts encrypted with an AES-256-GCM content key
           content key wrapped with the SaaS public key, pinned at
             enrolment
           authentication by signed JWT rather than client cert
           the proxy inspects, sees an opaque blob, is content

  10:31  batches flowing. All four collectors. 91 MINUTES TOTAL.

  WITHOUT OPTION 3 BUILT, THE DEPLOYMENT WOULD HAVE BEEN BLOCKED
  ELEVEN DAYS — at a customer whose network team was behaving
  entirely correctly.

  This is not an edge case. It is the default in the segment
  Overlook is aimed at, and it belongs in the design rather than in
  an incident report.
```

### 8.3 A four-day SaaS outage

```
  A regional cloud incident makes the endpoint unreachable for
  4 days 6 hours.

  HOUR 0     retries begin, backoff reaches the 5 min cap (LLD §34)
  HOUR 0     spooling starts (LLD §35)
  HOUR 2     alarm: "SaaS unreachable 2h · spool 0.2%"

  SPOOL AT PEAK
    COL-mer-01   192 MB   0.10% of 200 GB
    COL-mer-02   164 MB   0.08%
    COL-mer-03   287 MB   0.14%
    COL-mer-04    31 MB   0.02%

  NOT ONE THRESHOLD APPROACHED. No heartbeat suppression, no
  shedding, nothing degraded.

  DRAIN
    674 MB, ~13,500 batches, oldest first, in 11 minutes
    rate-limited so the drain did not overwhelm a SaaS ingest tier
    that had just come back
    0 duplicates — every batch_id was already assigned, and SaaS
    deduplicated the 40 that had been in flight

  AT SAAS, DURING THE OUTAGE
    the graph was 4 days stale and SAID SO. Coverage windows ended
    at hour 0 on every connector, so nothing was retracted, no path
    closed, and no exposure score moved.

  A FOUR-DAY OUTAGE COST 0.14% OF THE SPOOL AND ZERO FACTS.

  Entirely because of the reduction cascade. The same outage in an
  architecture that ships events would have needed 3.6 TB of spool
  per collector and would have lost everything after six hours.
```

---

*Next: [Local State and Storage](09-local-state-and-storage.md)*

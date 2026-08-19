# Flow Control

**Series:** [Ingestion](00-index.md)

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. What it is

What happens when more arrives than can be processed.

Two separate mechanisms that are often confused:

```
  INBOUND FLOW CONTROL   this document
    the collector is saturated. Backpressure propagates to receive,
    which sheds by priority class.

  OUTBOUND RATE GOVERNANCE   ../engines/01 §3.4
    the SOURCE would be saturated. Token buckets limit how fast we
    call someone else's API.

  Opposite directions, opposite concerns. Conflating them produces
  a system that throttles its own collectors when its own disk is
  slow, which fixes nothing.
```

---

## 2. How backpressure propagates

```
  Every stage has a BOUNDED queue.

  RECEIVE → journal → identify → parse → normalize → enrich
          → resolve → derive

  When a queue fills, the upstream stage blocks.
  Blocking propagates backwards to RECEIVE, which is the only
  stage that can do anything about it — because it is the only
  one touching the outside world.

  RECEIVE's options, in order:
    PULL     stop dispatching new collector runs (E15 backs off)
    PUSH     return 503 with Retry-After
    AGENT    return a pacing hint
    STREAM   SHED — the only class with no way to say "wait"
```

**Streams are why shedding exists.** A syslog sender does not care that we are busy. It has no ack, no retry, and no interest in our queue depth. The only available action is to drop, and the only question is *what*.

---

## 3. Priority classes

```
  P0  agent telemetry · AI gateway facts
      irreplaceable, tiny, and the differentiated layer

  P1  cloud audit trails
      the USED state and change attribution

  P2  identity events
      authentication and directory changes

  P3  firewall and flow
      high volume, low density

  P4  application logs

  P5  everything else
      unidentified sources, verbose debug, printers
```

**Shed lowest-first, and never uniformly.** Losing 100% of verbose printer syslog is far better than losing 10% of everything, because 10% of everything means every source is now unreliable and none of them can be reasoned about.

### 3.1 Classification happens before processing

```
  A record's priority must be known BEFORE anything expensive
  touches it, or shedding cannot help — we would have already
  paid the cost we are trying to avoid.

  Classification uses only (src_ip, port) for streams, and the
  declared source for pull, push and agent. No parsing, no
  inspection.
```

---

## 4. Shedding policy

```
  TRIGGER              journal write latency, queue depth, or
                       disk utilisation crossing a threshold

  ACTION, escalating
    level 1  stop dispatching new PULL runs
             → the cheapest intervention, and often sufficient.
               Pull is re-fetchable, so nothing is lost.
    level 2  503 to PUSH, pacing hints to AGENT
    level 3  shed P5 streams entirely
    level 4  shed P4
    level 5  shed P3 by SAMPLING, not by dropping wholesale
             → a 1-in-10 sample of flow still produces usable
               aggregates with a recorded sampling rate
    NEVER    shed P0, P1 or P2

  RECOVERY
    hysteresis on every threshold. A system that sheds at 80% and
    recovers at 79% oscillates. Shed at 80%, recover at 60%.
```

### 4.1 Sampling beats dropping for streams

```
  Wholesale dropping of flow for 10 minutes creates a hole:
  reachability edges simply do not exist for that window, and
  nothing records why.

  Sampling 1-in-10 and RECORDING the sampling rate on every
  aggregate produces aggregates that are still correct in shape,
  with a known and stated confidence penalty.

  → sampled aggregates carry sampling_rate, and downstream
    confidence is reduced accordingly.
```

---

## 5. Disk pressure

The failure this whole mechanism exists to prevent.

```
  JOURNAL UTILISATION THRESHOLDS

  60%   warn in the Controller
  75%   stop dispatching new PULL runs
  85%   503 to push, pacing to agents, shed P5
  92%   shed P4, sample P3
  97%   REFUSE ALL WRITES. Alarm at maximum severity.
        Continue serving the Controller and the resolve API.

  AT NO POINT do we silently drop a record we accepted.
  Refusing to accept is honest. Accepting and discarding is not.
```

**The 97% behaviour is deliberate.** A collector that stops ingesting but keeps answering resolve queries is degraded; one that fills its disk and stops entirely is an outage that also takes down every analyst's investigation.

---

## 6. Worker pool isolation

Flow control at the CPU level rather than the queue level.

```
  POOL A   API / IO bound          most collectors
           concurrency 4 × vCPU, capped at 64
  POOL B   CPU bound               parsing, closure, classification
           concurrency vCPU − 2
  POOL C   SCANNER                 DSPM crawling, local analytics
           concurrency 4, HARD CAP, cannot borrow
  POOL D   REALTIME                resolve API, response, agent gateway
           RESERVED, never yielded

  Pool D's reservation is what keeps the analyst-facing latency
  budget (150 ms p95) intact while the collector is under ingest
  pressure. Pool C's hard cap is what stops a classification crawl
  starving identity collection — the single most common way an
  collector like this falls over.
```

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Uniform shedding | Every source becomes unreliable | Shed by priority class, lowest first |
| No hysteresis | Oscillation between shedding and normal | Shed at 80%, recover at 60% |
| Priority classified after parsing | The cost we are avoiding is already paid | Classify on `(src_ip, port)` alone |
| Silent drop on disk-full | Data accepted and discarded | Refuse to accept; alarm |
| Scanner pool borrows capacity | Classification starves identity collection | Hard cap, no borrowing |
| Realtime pool yields | Resolve API latency blown during ingest spikes | Reserved capacity |
| Sustained 503 to a webhook source | The source disables the webhook | Backpressure must be rare; alert on recurrence |
| Flow dropped wholesale | Unexplained holes in reachability | Sample and record the rate instead |

---

## 8. Considerations

**Backpressure should be rare and short.** If it is routine, the collector is undersized and the answer is a bigger profile, not better shedding. Persistent shedding must be surfaced as a sizing recommendation in the Controller, not absorbed silently.

**Shedding must be visible in coverage, not only in logs.** If P3 flow was sampled for two hours, the reachability edges from that window carry lower confidence and the coverage view should say so. Otherwise an analyst reads a gap as an absence.

**Pull backoff is the cheapest lever and should be used first.** Pull data is re-fetchable; deferring a collector run costs API calls and nothing else. Reaching for stream shedding before pausing pull is the wrong order.

**Do not shed P0.** The agent layer is 102 MB/day. There is no pressure scenario where dropping it helps, and it is the one thing nobody else collects.

---

## 9. Example: Meridian, a pressure event

```
  02:00  nightly full enumeration begins. AD sweep (24,000 objects,
         720,000 ACEs) plus 41 AWS account enumerations plus the
         Parquet daily compaction, all overlapping.

  02:14  journal write latency rises. Queue depth on the parse
         stage crosses its threshold. Journal utilisation 76%.

  02:14  LEVEL 1 — E15 stops dispatching new PULL runs.
         In-flight runs continue. 8 queued collector runs deferred.
         → nothing lost. Pull is re-fetchable.

  02:19  utilisation 86%. Still climbing — the AD ACL enumeration
         is the bulk of it and cannot be paused mid-sweep without
         losing its coverage window.

  02:19  LEVEL 2 — 503 with Retry-After to 2 webhook deliveries.
                   Pacing hints to agents: extend to 900s.
         LEVEL 3 — P5 shed. The unidentified source at
                   10.4.9.22:514 (4,200 rec/h) stops being
                   accepted. Recorded.

  02:31  AD sweep completes. Coverage window emitted.
         Utilisation falls to 71%.

  02:36  utilisation 58% — below the 60% recovery threshold.
         Hysteresis satisfied.
         → pull dispatch resumes, the 8 deferred runs execute
         → agents return to normal cadence
         → P5 acceptance resumes
         → the 2 webhook deliveries were retried by GitHub at
           02:22 and 02:24, both accepted

  WHAT WAS ACTUALLY LOST
    ~1,300 records from the unidentified P5 source, over 17 minutes.
    Recorded as a shed event with the count and the class.
    Nothing from P0, P1 or P2. Nothing from push or agent.

  WHAT THE CONTROLLER SHOWED THE NEXT MORNING
    ⚠ SHEDDING EVENT · 02:19-02:36 · 17 minutes
      P5 shed: 1,300 records from 10.4.9.22:514
      Cause: nightly full enumeration overlapped Parquet compaction
      RECOMMENDATION: stagger the compaction window, or Edge L
      is approaching its ceiling on this workload
```

That last line matters. Shedding worked, and it was reported as a **sizing signal** rather than absorbed as normal operation.

---

*Next: [Journal replay](07-replay.md)*

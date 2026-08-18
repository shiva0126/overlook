# 5 — The Security Fact Engine

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 5. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **1.0 vCPU · 12 GB RAM** — the largest
> memory allocation of the seven, because the windows live here.

---

## 1. Purpose

Turn a stream of events into a small number of durable statements about what
exists, what is connected to what, what is wrong, and what was seen.

**This is where the product happens.** Everything before it is plumbing that
several vendors have built. Everything after it is transport. The 19,000:1
reduction in `00 §6` is almost entirely this service, and so is the difference
between Overlook and a log shipper.

---

## 2. Position

```
  INPUTS
    enriched records, from the Enrichment Engine (04)
    entity registry and open windows, from Local Store (08 §3)

  OUTPUTS
    Security Facts → Privacy Engine (06)
      · ENTITY        something exists
      · RELATIONSHIP  something connects to something
      · FINDING       something is wrong
      · OBSERVATION   something was seen, N times, between T1 and T2
    coverage windows → carried on every fact
    fact telemetry → the Controller

  CONSUMED BY
    06 privacy engine, then 07 forwarder, then SaaS (09)
```

---

## 3. The four fact types

The handoff names these four. Defining them precisely is the most valuable thing
this document does, because the boundary between them decides volume, and volume
decides whether the architecture works.

```
  ENTITY          A THING THAT EXISTS.
                  Identity · asset · network · datastore · repository ·
                  cloud resource · AI agent · policy · certificate
                  EMITTED   on discovery, on change, on heartbeat
                  VOLUME    bounded by the estate. Meridian: 2.9M total,
                            a few thousand changes a day.

  RELATIONSHIP    A DIRECTED EDGE BETWEEN TWO ENTITIES.
                  CAN_ASSUME · AUTHENTICATES_TO · CAN_READ · ROUTES_TO ·
                  MEMBER_OF · RUNS_AS · CONNECTS_TO · PROTECTS · INVOKES
                  EMITTED   on discovery, on change, on removal
                  VOLUME    bounded by the estate. Meridian: 2.9M live.

  FINDING         AN ASSESSED PROBLEM WITH A SPECIFIC ENTITY OR EDGE.
                  over-broad OIDC subject · unrotated key · public
                  bucket · disabled MFA on a privileged identity ·
                  a permission granted and never used
                  EMITTED   on detection, on resolution
                  VOLUME    hundreds. Meridian: 183 open.

  OBSERVATION     A TIME-BOUNDED, COUNTED STATEMENT THAT SOMETHING
                  WAS SEEN.
                  "IDN-jsmith authenticated to AST-lt-4471,
                   47 times, 09:00–09:05"
                  EMITTED   once per merge window per tuple
                  VOLUME    THE ONLY UNBOUNDED TYPE. This is what
                            §4 exists to control.
```

### 3.1 The distinction that governs the architecture

```
  10,000 authentication events between the same identity and the
  same host in five minutes are:

    ONE RELATIONSHIP    IDN-jsmith ─AUTHENTICATES_TO→ AST-lt-4471
                        emitted ONCE, when first discovered, and
                        never again unless it changes or disappears

    ONE OBSERVATION     the same tuple, count 10,000,
                        first_seen 09:00:00, last_seen 09:04:59

    ZERO EVENTS         events do not leave the collector

  10,000 → 2.

  A design that ships one fact per event ships 864 million facts a
  day from one collector. That is a data lake with extra steps, and
  it is the cost base the handoff §2 explicitly rejects.
```

**FINDING vs DETECTION.** A finding is a statement about *configuration or
state* — this trust policy is too broad, this key has not rotated in 400 days.
It is not an alert about behaviour. Behavioural detection is out of scope
(`00 §5`), and keeping findings on the configuration side of that line is what
keeps this service inside 1.0 vCPU.

---

## 4. Merge windows

The mechanism that makes OBSERVATION volume bounded.

### 4.1 How it works

```
  KEY      (subject, predicate, object, context_hash)
  VALUE    { count, first_seen, last_seen, confidence, sources[] }

  for each enriched record:
    derive the tuple
    if the key is open in the window:  count++, extend last_seen
    else:                              open a new window entry

  on window close:
    emit ONE observation fact per key
    persist the tuple to the Local Store as "seen"
    free the memory
```

### 4.2 Window durations, per type

```
  OBSERVATION      5 minutes, or 10,000 counts, whichever first
                   short enough that SaaS sees near-real-time
                   activity; long enough to collapse a login storm

  RELATIONSHIP     no window — emitted on CHANGE, not on interval.
                   A relationship that has not changed produces
                   nothing at all.

  ENTITY           no window on change; a HEARTBEAT every 6 hours
                   so SaaS can distinguish "unchanged" from
                   "collector stopped reporting"

  FINDING          no window — emitted on detection and resolution
```

### 4.3 Memory is the constraint, and it must be bounded by policy

```
  12 GB / ~180 bytes per window entry  ≈  70 MILLION open keys

  Meridian COL-mer-01, peak: ~1.4M open keys. Comfortable.

  BUT KEY CARDINALITY IS NOT UNDER OUR CONTROL.

  A firewall logging every (src_ip, dst_ip, dst_port) tuple across
  a /16 scanning event produces millions of unique keys in seconds.
  A misconfigured application opening a new source port per request
  does the same.

  THEREFORE, HARD BOUNDS:

    max_open_keys        20,000,000    per collector
    on 80%               shorten windows to 60 s, emit early
    on 95%               emit and close the largest cohort
    on 100%              ⚠ emit unmerged, count it, alarm

  Emitting unmerged is a volume event, not a loss event. The facts
  still ship, they are just larger and less useful. That is the
  correct degradation: never lose, degrade the reduction.
```

**`context_hash` is the cardinality control.** It captures the fields that make
two observations of the same predicate meaningfully different — a port, an
action, an outcome — and deliberately excludes fields that only add cardinality
without adding meaning, like a source ephemeral port or a request ID. What goes
into it is a per-predicate policy decision and it is the single most effective
lever on this service's memory use.

---

## 5. Noise policy — dropping before extracting

```
  Not every event contributes a fact. Firewall traffic-accept logs
  for an established, already-known connection restate a
  relationship that was asserted hours ago.

  DROP RULES, evaluated BEFORE extraction:
    · the relationship is already open in this window and the
      observation adds no new context_hash
    · the event type is on the connector's noise list
    · the event is a duplicate by (connector, sequence) — a
      retransmit

  MERIDIAN COL-mer-01:  ~70% of firewall traffic logs dropped here.

  ⚠ EVERY DROP IS COUNTED AND ATTRIBUTED.
    facts_dropped_total{connector,rule}
    A noise rule that starts dropping 99% instead of 70% is either
    a configuration error or an outage, and the counter is the
    only thing that tells them apart.
```

Where a drop predicate can be evaluated on the raw line, `03 §7` says to do it
at the parser instead — parsing and then discarding spends the collector's
most expensive budget on data that was never going to be used.

---

## 6. Coverage windows

**PROPOSED** — escalation E3. Not in the handoff. The strongest argument for
adding it is that without it the product produces confidently wrong answers, and
that it costs almost nothing.

### 6.1 The problem

```
  ABSENCE OF OBSERVATION IS NOT OBSERVATION OF ABSENCE.

  SaaS holds:  IDN-svc-batch ─CAN_ASSUME→ ROL-ghadeploy
               last_seen 2026-08-18T03:59:00Z

  It is now 09:00 and the fact has not been re-observed.

  TWO POSSIBLE WORLDS, AND THE FACT STREAM CANNOT TELL THEM APART

    A  the permission was revoked at 04:00
       → the edge should be removed, an attack path closes,
         the exposure score should improve

    B  the AWS connector broke at 04:00
       → nothing changed in the estate, and removing the edge
         invents a security improvement that did not happen
```

### 6.2 The mechanism

Every fact carries the window during which its source was known to be
collecting.

```
  coverage: {
    connector_id     CON-aws-org-01
    window_start     2026-08-18T03:00:00Z
    window_end       2026-08-18T04:00:00Z
    completeness     FULL | PARTIAL | DEGRADED
    gaps             [ {from, to, reason} ]
  }

  completeness is derived from
    · gateway drop counters            (01 §4.3)
    · parse success rate               (03 §6)
    · connector fetch success          per cycle
    · enrichment coverage              (04 §6)

  SAAS RETRACTION RULE
    remove a relationship ONLY IF a fact arrives whose coverage
    window CONTAINS the period in question and is marked FULL,
    and which does not re-assert it.

    Otherwise the relationship is held as STALE with its last
    coverage timestamp shown — not removed, not asserted as current.
```

### 6.3 Cost

```
  ~120 bytes per fact batch, not per fact — the window is a
  property of the connector and the period, so it is carried
  once per batch and referenced.

  A few counters per connector, already being collected for
  the Controller's health view.

  This is the cheapest escalation on the list and the one whose
  absence is most immediately visible to a customer.
```

---

## 7. Confidence

Every fact carries a confidence, and it is the input to
`../analytics/04 §3.2`'s `confidence_factor`.

```
  CONTRIBUTORS

    ingress class     PULL 1.00 · AGENT 0.95 · PUSH 0.90 · STREAM 0.80
                      STREAM is lower because it is unauthenticated
                      and lossy (01 §5)

    parser state      frozen 1.00 · confirmed 0.95 · proposed 0.75 ·
                      generic fallback 0.60          (03 §4.2)

    enrichment        all present 1.00 · identity missing 0.70 ·
                      asset missing 0.80             (04 §5)

    coverage          FULL 1.00 · PARTIAL 0.85 · DEGRADED 0.60

    confidence = the PRODUCT, floored at 0.30
                 below 0.30 the fact is not emitted at all
```

```
  THE FORTIGATE JSON FALLBACK FROM 03 §9.2, SCORED

    STREAM 0.80 × generic fallback 0.60 × enrichment 1.00
    × coverage PARTIAL 0.85
    = 0.41

  Emitted, and visibly untrustworthy. That is the correct outcome:
  the path stays visible so the analyst knows it exists, and the
  0.41 tells them not to act on it without checking.
```

---

## 8. Considerations

**Extraction is table-driven, not per-source code.** A rule maps
`(event.action, entity types present)` to a fact template. Adding a source adds
rules to the content bundle, not code — the same argument as `03 §4`, and for
the same reason.

**Relationship removal is never inferred from silence.** A relationship is
removed when a source *states* it is gone — a policy no longer lists the
principal, an account is absent from a full enumeration. Never because it
stopped appearing. `§6` is what makes this enforceable.

**A full enumeration is a different fact from an incremental one.** When the AWS
connector lists every role in an account, that list is authoritative for that
account at that moment and *can* justify removals. When it streams CloudTrail
events, it cannot. Facts carry which kind they came from, and only the first
kind licenses a retraction.

**The heartbeat on entities is not redundant.** Without it, SaaS cannot
distinguish "this asset is unchanged" from "this collector has been dead for
three days". Six hours is frequent enough to notice and infrequent enough to
cost nothing — Meridian's 2.9M entities heartbeating every 6 h is ~134 facts/sec
averaged, and it batches to almost nothing.

**Do not let findings drift into detections.** The pressure will be constant,
because the pipeline is right there and a rule is easy to write. Every finding
must be answerable by "what is misconfigured?" If the answer is "someone did
something suspicious", it belongs in SaaS correlation or in another product.

**Windows must survive a restart.** 1.4M open keys discarded on a restart means
1.4M observations emitted unmerged. Checkpoint the window state to the Local
Store every 30 s, and restore on startup.

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| One fact per event | 864M facts/day; the lake this architecture exists to avoid | Relationship/observation split, §3.1 |
| Unbounded window keys | OOM; the 64 GB ceiling breached | `max_open_keys` + graduated early emission, §4.3 |
| `context_hash` includes ephemeral fields | Cardinality explosion from ports and request IDs | Per-predicate policy on what enters the hash |
| Removal inferred from silence | Broken connector improves the exposure score | Coverage windows + explicit-removal-only, §6 |
| Incremental treated as authoritative | Wrong retractions from a partial view | Enumeration vs incremental carried on the fact |
| No entity heartbeat | Dead collector indistinguishable from a stable estate | 6 h heartbeat |
| Windows lost on restart | A burst of unmerged observations after every deploy | 30 s checkpoint to Local Store |
| Noise rule silently over-drops | Looks like an outage; is a config error | `facts_dropped_total{connector,rule}` |
| Findings become detections | Scope creep past 1.0 vCPU; wrong product | Configuration-only test on every finding rule |
| Confidence not carried | A JSON-fallback fact scores like a PULL fact | Product of four contributors, floored |

---

## 10. Example: Meridian

### 10.1 One five-minute window on COL-mer-01

```
  INPUT                                        3,270,000 events

  NOISE POLICY (§5)
    established-connection restatements       −2,289,000   (70%)
    connector noise lists                       −163,500
    duplicate sequence numbers                      −890
                                              ───────────
  REACHING EXTRACTION                            817,610

  EXTRACTED
    observations, pre-merge                      817,610
    unique (subject, predicate, object, ctx)      11,240
                                              ───────────
  OBSERVATION FACTS EMITTED                       11,240   (73:1)

    relationships — new                               34
    relationships — changed                            7
    relationships — removed                            2   ← both
                                                            from an
                                                            authoritative
                                                            enumeration
    entities — new                                    12
    entities — changed                                89
    findings — new                                     1
    findings — resolved                                0
                                              ───────────
  TOTAL FACTS                                     11,385

  3,270,000 EVENTS  →  11,385 FACTS     287 : 1
  and after compression on the wire (07)  ~1,100 : 1
```

### 10.2 The one finding

```
  FINDING  FND-oidc-subject-broad-001

    entity        POL-github-oidc-trust
    relationship  PIP-gha-any-meridian-repo ─CAN_ASSUME→ ROL-ghadeploy
    class         over_broad_federation_subject
    detail        the trust policy condition
                    token.actions.githubusercontent.com:sub
                      = "repo:meridian/*"
                  matches every repository in the organisation,
                  including 4 with external contributors
    confidence    PULL 1.00 × frozen 1.00 × enrichment 1.00
                  × coverage FULL 1.00  =  1.00

  This is a CONFIGURATION statement. It required no behaviour, no
  baseline, no anomaly model. It is true the moment the policy is
  read, and it is the head of the path in ../analytics/04 §9.1.

  It is also the argument for the whole product in one fact: no
  amount of log volume would have produced it, and one API call
  did.
```

### 10.3 A coverage window preventing a wrong retraction

```
  04:00  the AWS connector's credentials expire. PULL cycles begin
         failing. No AWS facts are produced.

  04:00–09:12   SaaS receives no re-assertion of 1,847 AWS
                relationships, including the CAN_ASSUME edge above.

  WITHOUT COVERAGE WINDOWS
    09:12  a staleness sweep removes relationships unseen for 5 h
           → 1,847 edges deleted
           → 6 attack paths close, including the flagship
           → exposure score 58 → 39
           → the Monday report shows a 19-point improvement
           → nothing whatsoever changed at Meridian

  WITH THEM
    every AWS fact before 04:00 carried
      coverage.window_end 04:00:00Z, completeness FULL

    no fact arrives after 04:00 with a window covering 04:00–09:12

    → the retraction rule (§6.2) does not fire
    → the 1,847 relationships are held as STALE, timestamped
    → the exposure score does not move
    → the Controller shows
        "CON-aws-org-01 · no coverage since 04:00 · credential
         expired · 1,847 relationships stale"

  09:12  credentials rotated, PULL resumes, a full enumeration
         arrives with coverage FULL for 09:12 onward
  09:13  1,844 relationships re-asserted and marked fresh
         3 genuinely gone — removed, correctly, on authority of an
         enumeration rather than on silence
```

---

*Next: [Privacy Engine](06-privacy-engine.md)*

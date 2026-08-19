# 6 — The Security Fact Engine

**Series:** [The Edge Collector](00-index.md) · **LLD:** §23, §24, §25, §77–81, §88

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. **Two
> escalations live here** — [ESC-4](13-escalations.md) (per-event facts
> and a risk score the collector cannot compute) and
> [ESC-5](13-escalations.md) (coverage windows).
> Budget: **1.0 vCPU · 12 GB RAM** (`00 §4`).

---

## 1. Purpose

Turn a stream of events into a small number of durable statements about what
exists, what is connected to what, and what is wrong.

**This is where the product happens.** Everything before it is plumbing several
vendors have built. Everything after it is transport. LLD §88 says it directly:

> The Collector should **not be treated as a log-forwarding appliance.** […] The
> central object connecting all Overlook modules should be
> `SECURITY FACT + ENTITY + RELATIONSHIP` rather than the original raw log.

The 19,000:1 reduction in `00 §7` is almost entirely this module.

---

## 2. Position

```
  INPUTS
    enriched events (05)
    the entity/relationship state, from SQLite (09 §3)

  OUTPUTS
    Security Facts (LLD §23) · Entities (§24) · Relationships (§25)
    → Privacy Engine (07)

  CONSUMED BY
    07 privacy engine, then 08 forwarder, then SaaS (12)
```

---

## 3. The three objects

LLD §88's triad. Defining the boundary between them precisely is the most
valuable thing this document does, because the boundary decides volume, and
volume decides whether the architecture works at all.

```
  ENTITY (LLD §24)      A THING THAT EXISTS.
                        identity · asset · cloud workload · network ·
                        datastore · repository · AI agent · policy
                        EMITTED  on discovery, on change, on heartbeat
                        VOLUME   bounded by the estate.
                                 Meridian: 2.9M, a few thousand
                                 changes a day.

  RELATIONSHIP (§25)    A DIRECTED EDGE BETWEEN TWO ENTITIES.
                        CAN_ASSUME · CAN_ACCESS · RUNS_ON ·
                        CONNECTED_TO · CONTAINS · AUTHENTICATES_TO ·
                        MEMBER_OF · ROUTES_TO       (LLD §81)
                        EMITTED  on discovery, on change, on removal
                        VOLUME   bounded. Meridian: 2.9M live.
                        ✓ ALREADY CARRIES first_seen, last_seen,
                          confidence — the LLD gets this right.

  SECURITY FACT (§23)   AN ASSESSED STATEMENT ABOUT SUBJECT-ACTION-
                        OBJECT, with severity and context.
                        EMITTED  per LLD §23, ONE PER EVENT
                        VOLUME   ⚠ UNBOUNDED. This is ESC-4.
```

### 3.1 The distinction that governs everything

```
  10,000 authentication events between the same identity and the
  same role in five minutes are:

    ONE RELATIONSHIP     IDN-jsmith ─CAN_ASSUME→ ROL-prodadmin
                         emitted ONCE on discovery, never again
                         unless it changes or disappears

    ONE AGGREGATED FACT  the same tuple, count 10,000,
                         first_seen 09:00:00, last_seen 09:04:59

    ZERO EVENTS          events do not leave the collector

  10,000 → 2.

  UNDER LLD §23 AS WRITTEN, IT IS 10,000 → 10,001.

  At 10,000 EPS that is 864,000,000 facts/day from one collector —
  the data lake LLD §1 and §88 exist to avoid, moved one stage
  later and given a different name.
```

### 3.2 The proposed fact shape

**ESC-4a.** The change is small, and §25 already does it correctly for
relationships:

```json
  {
    "fact_id":     "fact-892188",
    "tenant_id":   "tenant-acme",
    "collector_id":"col-sg-01",
    "fact_type":   "identity_access",
    "subject":     { "type": "identity",   "id": "…" },
    "action":      "assume_role",
    "object":      { "type": "cloud_role", "id": "…" },
    "context":     { "environment": "production", "cloud": "aws" },
    "severity":    "high",

    "count":       10000,                    // ADDED
    "first_seen":  "2026-08-18T10:31:00Z",   // ADDED
    "last_seen":   "2026-08-18T10:36:00Z",   // ADDED
    "confidence":  0.94,                     // ADDED  (§7)
    "coverage":    { … }                     // ADDED  (§6, ESC-5)

    // "risk_score": 78   ← REMOVED. See §3.3.
  }
```

### 3.3 Why `risk_score` cannot be computed here

```
  A RISK SCORE NEEDS
    crown jewel designation   SaaS — customer-declared
    start conditions          SaaS — whole-estate
    the full attack path      SaaS — spans collectors
    blast radius              SaaS — whole-graph

  The collector has NONE of these. Whatever number it emits is
  either a per-event severity mislabelled as risk, or it will
  disagree with the SaaS score for the same entity — and the
  customer will see both numbers on two screens.

  → the collector emits SEVERITY, which it can determine from the
    source and the rule, and CONFIDENCE, which ONLY it can
    determine. SaaS computes risk.

    One number, one owner.
```

---

## 4. Aggregation windows

The mechanism that makes fact volume bounded. LLD §40 already specifies a
fingerprint and a TTL cache for dedup; this is the same machinery applied over a
window rather than to exact retransmits.

### 4.1 Mechanism

```
  KEY    (fact_type, subject, action, object, context_hash)
  VALUE  { count, first_seen, last_seen, confidence, sources[] }

  per enriched event:
    derive the key
    open in the window?  count++, extend last_seen
    else                 open a new entry

  on window close:
    emit ONE fact per key
    persist the tuple to SQLite as seen
    free the memory
```

### 4.2 Durations, per object

```
  SECURITY FACT   5 minutes, or 10,000 counts, whichever first
                  short enough for near-real-time activity at SaaS;
                  long enough to collapse a login storm

  RELATIONSHIP    no window — emitted on CHANGE. A relationship that
                  has not changed produces nothing at all.

  ENTITY          no window on change; a HEARTBEAT every 6 hours so
                  SaaS can distinguish "unchanged" from "the
                  collector stopped reporting"
```

### 4.3 Memory must be bounded by policy

```
  12 GB / ~180 bytes per entry  ≈  70 MILLION open keys
  Meridian COL-mer-01 peak: ~1.4M. Comfortable.

  BUT KEY CARDINALITY IS NOT UNDER OUR CONTROL.

  A firewall logging every (src, dst, port) tuple during a /16 scan
  produces millions of unique keys in seconds. A misconfigured
  application opening a new source port per request does the same.

  HARD BOUNDS
    max_open_keys   20,000,000
    at 80%          shorten windows to 60 s, emit early
    at 95%          emit and close the largest cohort
    at 100%         ⚠ emit UNMERGED, count it, alarm

  Emitting unmerged is a VOLUME event, not a LOSS event. Facts still
  ship, larger and less useful. That is the correct degradation:
  never lose, degrade the reduction.

  ⚠ IN A MONOLITH (LLD §5) AN OOM HERE KILLS COLLECTION. This bound
    is not a nicety.
```

**`context_hash` is the cardinality control.** It captures fields that make two
facts meaningfully different — a port, an action, an outcome — and deliberately
excludes fields that add cardinality without meaning, like an ephemeral source
port or a request ID. What enters it is a per-fact-type policy decision and it is
the single most effective lever on this module's memory.

---

## 5. Noise policy

```
  Not every event contributes a fact. A firewall accept for an
  established, already-known connection restates a relationship
  asserted hours ago.

  DROP RULES, evaluated BEFORE extraction:
    · the relationship is already open in this window and the event
      adds no new context_hash
    · the event type is on the connector's noise list
    · the event is a duplicate by fingerprint (05 §6)

  MERIDIAN COL-mer-01: ~70% of firewall traffic logs dropped here.

  ⚠ EVERY DROP IS COUNTED AND ATTRIBUTED.
    overlook_facts_dropped_total{connector,rule}
    A rule that starts dropping 99% instead of 70% is either a
    configuration error or an outage, and the counter is the only
    thing that distinguishes them.
```

Where the predicate can be evaluated on the raw line, `03 §8` says do it at the
parser instead — parsing and then discarding spends the collector's most
expensive budget on data that was never going to be used.

---

## 6. Coverage windows

**ESC-5.** Not in the LLD. The argument for adding it is that without it the
product produces confidently wrong answers, and that it costs almost nothing.

```
  ABSENCE OF OBSERVATION IS NOT OBSERVATION OF ABSENCE.

  SaaS holds  IDN-svc-batch ─CAN_ASSUME→ ROL-ghadeploy
              last_seen 04:00

  It is 09:00 and it has not been re-observed. Two worlds:

    A  revoked at 04:00 → remove the edge, a path closes, the
       score improves
    B  the connector's credentials expired at 04:00 → nothing
       changed, and removing the edge INVENTS a security improvement

  LLD §72 marks the connector "degraded" LOCALLY. Nothing tells SaaS.
```

```json
  "coverage": {
    "connector_id": "con-aws-prod",
    "window_start": "2026-08-18T03:00:00Z",
    "window_end":   "2026-08-18T04:00:00Z",
    "completeness": "FULL",        // FULL | PARTIAL | DEGRADED
    "gaps": []
  }
```

```
  completeness DERIVES FROM COUNTERS THAT ALREADY EXIST
    gateway drop counters              (01 §4.3)
    overlook_parse_failures_total      (LLD §50)
    connector fetch success per cycle  (LLD §52)
    enrichment coverage                (05 §7)

  SAAS RETRACTION RULE
    remove a relationship ONLY IF a fact arrives whose coverage
    window CONTAINS the period, is marked FULL, and does not
    re-assert it. Otherwise hold as STALE with the last coverage
    timestamp shown.

  COST  ~120 bytes per BATCH, not per fact.
```

---

## 7. Confidence

LLD §25 already carries `confidence` on relationships. This is how it is
computed.

```
  ingress class    PULL 1.00 · AGENT 0.95 · PUSH 0.90 · STREAM 0.80
                   STREAM is lower because it is unauthenticated
                   and lossy (01 §5)

  parser state     frozen 1.00 · confirmed 0.95 · proposed 0.75 ·
                   generic fallback 0.60          (03 §5.2)

  enrichment       all present 1.00 · identity missing 0.70 ·
                   asset missing 0.80             (05 §5)

  coverage         FULL 1.00 · PARTIAL 0.85 · DEGRADED 0.60

  confidence = the PRODUCT, floored at 0.30.
               Below 0.30 the fact is not emitted at all.
```

```
  THE FORTIGATE JSON FALLBACK FROM 03 §10.2, SCORED

    STREAM 0.80 × generic 0.60 × enrichment 1.00 × PARTIAL 0.85
    = 0.41

  Emitted, and visibly untrustworthy. That is the right outcome:
  the path stays visible so the analyst knows it exists, and 0.41
  tells them not to act without checking.
```

---

## 8. Considerations

**Extraction is table-driven, not per-source code.** A rule maps
`(event.category, event.action, entity types present)` to a fact or relationship
template. Adding a source adds rules to the signed bundle, not code — the same
argument as `03 §5`, for the same reason.

**Relationship removal is never inferred from silence.** A relationship is
removed when a source *states* it is gone — a policy no longer lists the
principal, an account absent from a full enumeration. Never because it stopped
appearing. §6 is what makes this enforceable.

**A full enumeration is a different fact from an incremental one.** When the AWS
connector lists every role in an account, that list is authoritative for that
account at that moment and *can* justify removals. When it streams CloudTrail, it
cannot. Facts carry which they came from, and only the first licenses a
retraction.

**The entity heartbeat is not redundant.** Without it SaaS cannot distinguish
"unchanged" from "this collector has been dead three days". Six hours over
Meridian's 2.9M entities averages ~134 facts/sec and batches to almost nothing.

**Findings are configuration statements, not detections.** Every finding rule
must be answerable by *"what is misconfigured?"* If the answer is *"someone did
something suspicious"*, it belongs in SaaS correlation. The pressure to drift
will be constant because the pipeline is right there and a rule is easy to write.

**Windows must survive a restart.** 1.4M open keys discarded on restart means
1.4M facts emitted unmerged — and in a monolith every upgrade is a restart.
Checkpoint window state to SQLite every 30 s, restore on startup.

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| One fact per event | 864M facts/day; the lake the architecture exists to avoid | ESC-4a — aggregation window |
| `risk_score` emitted here | Two different risk numbers for one entity, on two screens | ESC-4b — severity + confidence only |
| Unbounded window keys | OOM; in a monolith that kills collection | `max_open_keys` + graduated early emission |
| `context_hash` includes ephemeral fields | Cardinality explosion from ports and request IDs | Per-fact-type hash policy |
| Removal inferred from silence | A broken connector improves the exposure score | ESC-5 + explicit-removal-only |
| Incremental treated as authoritative | Wrong retractions from a partial view | Enumeration flag on the fact |
| No entity heartbeat | Dead collector indistinguishable from a stable estate | 6 h heartbeat |
| Windows lost on restart | A burst of unmerged facts after every upgrade | 30 s checkpoint to SQLite |
| Noise rule silently over-drops | Looks like an outage; is a config error | `overlook_facts_dropped_total` |
| Findings drift into detections | Scope creep past 1.0 vCPU; the wrong product | Configuration-only test per rule |

---

## 10. Example: Meridian

### 10.1 One five-minute window on COL-mer-01

```
  INPUT                                        3,270,000 events

  NOISE POLICY (§5)
    established-connection restatements       −2,289,000   (70%)
    connector noise lists                       −163,500
    fingerprint duplicates (05 §6)                  −890
                                              ───────────
  REACHING EXTRACTION                            817,610

  AGGREGATED (§4)
    unique (type, subject, action, object, ctx)   11,240
                                              ───────────
  SECURITY FACTS EMITTED                          11,240   (73:1)

    relationships — new                               34
    relationships — changed                            7
    relationships — removed                            2  ← both from
                                                            an authoritative
                                                            enumeration
    entities — new                                    12
    entities — changed                                89
                                              ───────────
  TOTAL OBJECTS                                   11,384

  3,270,000 EVENTS → 11,384 OBJECTS        287 : 1
  after compression on the wire (08)     ~1,100 : 1

  UNDER LLD §23 AS WRITTEN: 817,610 facts for the same window.
  72× more, for the same information.
```

### 10.2 The finding that is the whole product

```
  FINDING  over_broad_federation_subject

    entity        POL-github-oidc-trust
    relationship  PIP-gha-any-meridian-repo ─CAN_ASSUME→ ROL-ghadeploy
    detail        the trust condition
                    token.actions.githubusercontent.com:sub
                      = "repo:meridian/*"
                  matches EVERY repository in the organisation,
                  including 4 with external contributors
    severity      high
    confidence    PULL 1.00 × frozen 1.00 × enrichment 1.00
                  × FULL 1.00  =  1.00

  This is a CONFIGURATION statement. No behaviour, no baseline, no
  anomaly model. It is true the moment the policy is read.

  It is also the argument for the whole product in one object: no
  amount of log volume would have produced it, and one API call did.
```

### 10.3 A coverage window preventing a wrong retraction

```
  04:00  the AWS connector's credentials expire. Polls fail. No AWS
         facts are produced. LLD §72: connector marked DEGRADED —
         locally.

  04:00–09:12  SaaS receives no re-assertion of 1,847 AWS
               relationships, including the CAN_ASSUME above.

  WITHOUT COVERAGE WINDOWS
    09:12  a staleness sweep removes relationships unseen for 5 h
           → 1,847 edges deleted
           → 6 attack paths close, including the flagship
           → exposure score 58 → 39
           → Monday's report shows a 19-point improvement
           → NOTHING CHANGED AT MERIDIAN

  WITH THEM
    every AWS fact before 04:00 carried
      coverage.window_end 04:00:00Z, completeness FULL
    no fact arrives after 04:00 with a window covering the gap
    → the retraction rule does not fire
    → 1,847 relationships held STALE, timestamped
    → the exposure score does not move
    → the UI shows "con-aws-prod · no coverage since 04:00 ·
       credential expired · 1,847 relationships stale"

  09:12  credentials rotated; a full enumeration arrives with
         coverage FULL from 09:12
  09:13  1,844 re-asserted and marked fresh
         3 genuinely gone — removed correctly, on the authority of
         an enumeration rather than on silence
```

---

*Next: [Privacy Engine](07-privacy-engine.md)*

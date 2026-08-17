# Local Analytics — The Retained Dataset

**Series:** [Analytics](00-index.md)

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

Everything else in this series is **exposure intelligence** — the graph, paths, scores. This is a different kind of analytics: **operational and diagnostic**, over the 30 days of enriched telemetry the collector retains locally (`../04-data-flow-to-security-facts.md §28.1`).

They answer different questions:

```
  EXPOSURE INTELLIGENCE     "what is possible?"
                            structural · the graph · leaves as facts

  LOCAL ANALYTICS           "what actually happened, and is this
                            collector working?"
                            temporal · Parquet + DuckDB · NEVER leaves
```

The second exists because of a constraint the privacy architecture creates: **we cannot ask a customer to send us their data**, so every diagnostic question must be answerable by them, locally, without a support ticket.

---

## 2. Position

```
  INPUT
    stage 5 (ENRICH) forks normalized, enriched records into
    partitioned Parquet — before resolution, before derivation

  ENGINE
    DuckDB, embedded in-process. A library, not a daemon.
    Reads the Parquet in place: the retained dataset IS the
    query store.

  SURFACES
    Controller → Local data          ad-hoc query
    Controller → Diagnostics         pre-built diagnostic queries
    Controller → Connections         per-collector cost and volume
    Controller → Coverage            trend over time
    degraded mode                    the only analytics available
                                     when the console is unreachable

  NEVER
    crosses the Privacy Gate. Nothing here is a fact.
```

---

## 3. What runs on it

Four classes of query, in descending order of how often they are used.

### 3.1 Pipeline and connector diagnostics

The primary purpose. Questions an operator asks when something looks wrong.

```
  · records received per source, per hour, against baseline
  · parse rate per source over time, with the drop annotated
  · field presence per source — which fields stopped populating
  · quarantine volume grouped by failure signature
  · API calls issued per collector, per account, per day
  · rate-limit waits and 429s per rate-limit domain
  · wall clock per collector run, and its trend
  · objects emitted per run versus the baseline
    → the query behind the SILENT state (../05 §6.1)
```

### 3.2 Contribution and coverage

```
  · which sources contributed to a given entity or edge
  · what a connector actually returned last Tuesday
  · coverage percentage per domain over 30 days
  · how a coverage gap changed after a connector was added
  · unresolved-reference counts per collector run
```

### 3.3 Investigation support

Not investigation itself — that is the graph. This is the evidence layer beneath it.

```
  · every observation that contributed to one edge
  · the raw enriched records behind a finding
  · activity for one identity or asset across all sources in a window
  · what a source reported about an entity before it changed
```

### 3.4 Cost and capacity

```
  · storage growth per source, projected against retention
  · quota consumption trend per provider
  · which collectors cost the most per edge produced
    → the cost sparkline that ranks collectors (../05 §10.1)
```

---

## 4. Pre-built versus ad-hoc

```
  PRE-BUILT     ~40 named queries covering §3.1 and §3.2
                exposed as buttons and panels, never as SQL
                these are what an operator actually uses

  AD-HOC        a SQL surface for the operator who wants it
                bounded by row limit and wall-clock timeout
                schema documented in the Controller

  Most operators will never write a query. The pre-built set is
  the product; the SQL surface is the escape hatch that stops
  "I need to know X" becoming a support ticket.
```

---

## 5. Constraints

Carried from `../04 §28.1`, restated because violating any of them is a real failure:

```
  · queries run in the SCANNER pool, never the realtime pool.
    An ad-hoc query must not contend with the resolve API's
    150 ms p95 budget.

  · DuckDB's temp_directory must be on an ENCRYPTED volume.
    Otherwise a large query spills plaintext customer data to disk,
    which is precisely what the architecture exists to prevent.

  · every query is bounded by row limit and timeout, and truncation
    is SHOWN rather than silent.

  · Parquet is written hourly per source and compacted daily.
    Small-file proliferation is the standard way this design
    degrades.

  · partition pruning is the performance model. A query without a
    source or date predicate scans everything; the query builder
    should require at least one.
```

---

## 6. Considerations

**This is not a SIEM and must not become one.** No alerting, no correlation search, no long retention tier. Thirty days, local, for diagnostics and investigation support. The moment someone asks for a saved search that emails on match, that is the boundary.

**It is not a second product surface.** It answers questions about the collector and its sources. Cross-domain investigation stays in the graph, in the console. The Controller's line (`../05 §2.1`) holds.

**Retention is a cost slider with visible consequences.** 30 days at Edge M is roughly 400 GB of Parquet after compression. The Controller should show the disk cost as the operator moves it, the same way evidence retention does.

**Schema evolution is real.** Enriched records change shape as normalizers are updated. Parquet handles added columns; removed or retyped columns break older partitions. Version the schema per partition and let DuckDB union across versions.

**It is the reason degraded mode is usable.** When the console is unreachable, this is the only analytical capability left. That is not a side effect — it is a large part of why it exists (`../05 §29`).

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Temp directory unencrypted | Plaintext spilled to disk | Explicit configuration, checked at startup |
| Query in the realtime pool | Resolve API latency budget blown | Scanner pool, enforced |
| No partition predicate | Full scan of 30 days | Query builder requires source or date |
| Small-file proliferation | Query time degrades steadily | Hourly write, daily compaction |
| Schema drift across partitions | Older partitions unreadable | Per-partition schema version, union on read |
| Silent truncation | Operator draws a wrong conclusion | Truncation shown with the limit that caused it |
| Becomes a SIEM | Scope explosion, second product | No alerting, no saved-search notification, ever |

---

## 8. Example: Meridian

### 8.1 The FortiGate parse failure, diagnosed locally

```
  06:12  attention inbox: parse rate collapse on fw-branch-02

  The operator opens Controller → Diagnostics and runs three
  PRE-BUILT queries. No SQL, no support ticket.

  1  PARSE RATE OVER TIME · source fw-branch-02 · 7 days
       98.9% steady until 03:41, then 4%
       → confirms a sharp break, not a gradual drift

  2  FIELD PRESENCE · source fw-branch-02 · before vs after 03:41
       srcintf    99.8% → 99.7%   unchanged
       dstintf    99.8% → 99.7%   unchanged
       service    99.1% →  0.2%   ← the field moved or was renamed
       app        41.0% → 98.9%   ← and something new appeared
       → not a total format change. A field rename.

  3  QUARANTINE BY FAILURE SIGNATURE · last 24h
       41,204 records · 3 distinct signatures
       signature A  38,900  "unexpected token at position 14"
       signature B   2,100
       signature C     204

  The operator now knows: FortiOS 7.6 renamed the service field,
  three failure shapes, 41k records recoverable from the journal.

  That diagnosis happened entirely on the customer's premises, by
  the customer, with zero data leaving. Which is the only way it
  could have happened.
```

### 8.2 A contribution question

```
  An analyst asks: "why does Overlook think Priya can write to
  payments-api?"

  The GRAPH answers structurally — the edge, its confidence, its
  evidence reference.

  LOCAL ANALYTICS answers evidentially:

    CONTRIBUTION · edge IDN-priya ─CAN_WRITE→ REP-payments-api

    source                 observations   first        last
    github.collaborators           41   2026-05-02   2026-08-17
    github.teams                   38   2026-05-02   2026-08-17
    ─────────────────────────────────────────────────────────────
    2 independent sources, 79 observations, no disagreement

    [ show the 3 most recent raw records ]

  The raw records are enriched, plaintext, local, and never left
  the collector.
```

### 8.3 Cost, answered without guessing

```
  Meridian's cloud team asks why AWS API usage rose.

  PRE-BUILT · API calls per collector · 30 days · provider aws

    cloudtrail          41%   ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇
    iam.policies        22%   ▇▇▇▇▇▇▇▇
    iam.roles           14%   ▇▇▇▇▇
    ec2.instances        9%   ▇▇▇
    s3.buckets           6%   ▇▇
    ... 37 others        8%

    total 2.4M calls/day · 22% of published quota
    trend: +31% since 2026-08-10
    coincides with: 6 accounts added to the org on 2026-08-09

  The answer is on the screen in one query, with the cause
  attached. The alternative is an argument between two teams,
  neither of which has the data.
```

---

*End of the analytics series. Back to the [index](00-index.md).*

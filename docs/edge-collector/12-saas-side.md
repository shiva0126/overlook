# 12 — The SaaS Side

**Series:** [The Edge Collector](00-index.md) · **LLD:** §76–81, §88

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary for the
> collector. The SaaS side has **no engineering handoff of its own**, and
> LLD §76's final four boxes are the entire specification that exists.
> This document exists because the collector's fact schema is determined
> by what this side needs — and while it has no owner,
> [ESC-3 and ESC-4](13-escalations.md) get decided by default.

---

## 1. Purpose

Receive facts from every collector of every tenant, resolve them into one graph,
compute what the collector deliberately does not, and serve it.

---

## 2. What the LLD specifies

LLD §76 ends with:

```
  Overlook SaaS
       │
       ▼
     NATS
       │
       ├── ClickHouse
       ├── Postgres
       └── Graph Processing
```

That is four boxes and no contract. Everything below is proposed, and is offered
so the collector's decisions have something to be correct against.

---

## 3. The receiving path

```
  ┌─────────────────────────────────────────────────────────────┐
  │  INGESTION API   POST /api/v1/collector/ingest   (LLD §32)  │
  │    mTLS, or signed JWT + payload encryption (08 §4.1)       │
  │    X-Batch-ID idempotency · sequence gap detection          │
  │    decrypt · decompress · validate                          │
  │    → 202 AS SOON AS DURABLE. Not after processing. (§7)     │
  │    → the LLD §33 acknowledgement body                       │
  └──────────────────────────┬──────────────────────────────────┘
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  NATS JETSTREAM       facts.<tenant>.<type>                 │
  │    retention in DAYS here, not hours — facts are four       │
  │    orders of magnitude smaller than raw, and replay across  │
  │    a schema change is worth having                          │
  └──────────────────────────┬──────────────────────────────────┘
                ┌────────────┼────────────┐
                ▼            ▼            ▼
       ┌──────────────┐ ┌──────────┐ ┌──────────────┐
       │  POSTGRES    │ │CLICKHOUSE│ │ GRAPH ENGINE │
       │  entities    │ │  facts   │ │  in-memory,  │
       │  relations   │ │  metrics │ │  materialized│
       │  findings    │ │  trends  │ │  from        │
       │  tenants     │ │  audit   │ │  Postgres    │
       │  coverage    │ │          │ │              │
       └──────────────┘ └──────────┘ └──────────────┘
                └────────────┼────────────┘
                             ▼
                   CORRELATION ENGINE
                             ▼
                   RISK / FINDINGS  (../analytics/)
                             ▼
                     OVERLOOK APIs
                             ▼
                       REACT UI
```

### 3.1 Which store gets which object

The split maps exactly onto LLD §88's triad, and it is the reason there are three
stores rather than one.

```
  ENTITY (§24)        → POSTGRES.  Bounded (2.9M), mutable,
                                   relational, needs constraints
                                   and transactions.

  RELATIONSHIP (§25)  → POSTGRES.  Bounded, mutable, bitemporal
                                   (first_seen / last_seen /
                                   removed_at).
                                   THE GRAPH IS DERIVED FROM THIS.

  FINDING             → POSTGRES.  Hundreds. Has workflow state —
                                   assigned, dispositioned,
                                   suppressed.

  SECURITY FACT (§23) → CLICKHOUSE. Append-only, never updated,
                                   queried in aggregate over time.
                                   Precisely the workload a
                                   columnar store exists for, and
                                   precisely the one Postgres is
                                   worst at.
```

```
  ⚠ PUT SECURITY FACTS IN POSTGRES AND THE DESIGN FAILS WITHIN A
    QUARTER.

  Even AGGREGATED (ESC-4a), Meridian produces ~45,000 facts per
  five-minute window across four collectors — ~13M/day, append-only,
  forever. That is a ClickHouse table and a Postgres outage.

  UNAGGREGATED, per LLD §23 as written, it is 3.4 BILLION rows a day.
```

---

## 4. The graph engine

LLD §76 says "Graph Processing" without naming one. The obvious answer is
probably wrong.

```
  MERIDIAN   2.9M entities · 2.9M relationships

  AS AN IN-MEMORY GO GRAPH
    node       ~120 B  →  350 MB
    edge        ~80 B  →  232 MB
    adjacency   ~64 B  →  186 MB
                          ───────
    per tenant             ~770 MB

  20 tenants of Meridian's size → ~15 GB. One server.
```

```
  RECOMMENDATION — MATERIALIZE IN MEMORY FROM POSTGRES

    · Postgres is the durable source of truth. The graph is a
      derived index, rebuilt on startup, updated incrementally.
    · traversal is a pointer chase in process memory —
      microseconds, not a network round trip per hop
    · the choke-point simulation in ../analytics/05 needs an
      overlay copy and a ~200 ms interactive budget. Trivial
      in-process, awkward against a database.
    · no license, no cluster, no second operational discipline
    · lost on restart, and rebuilding 2.9M edges takes seconds

  WHY NOT NEO4J
    · Enterprise licensing is a per-instance cost in a business
      whose margin depends on instance count
    · Community has no clustering and no RBAC
    · it is the right answer above ~50M edges. Meridian is 2.9M.

  REVISIT IF a single tenant passes ~50M edges.
```

---

## 5. Multi-collector entity resolution

Where four collectors become one graph. **This is the strongest argument for
[ESC-3](13-escalations.md) and it is rarely made on these grounds.**

```
  STAGE 1 — DETERMINISTIC. With tokenization it is ALL of it.

    tokens are byte-identical across collectors for the same
    (tenant, type, value). An equality join resolves:

      COL-mer-02   t_id_7QK3M9F2XB4NRWZ8
      COL-mer-03   t_id_7QK3M9F2XB4NRWZ8
      → the same node. No inference. No confidence penalty.

  STAGE 2 — CROSS-IDENTIFIER, still deterministic

    john.smith@meridian.com   → t_id_A
    jsmith (sAMAccountName)   → t_id_B
    arn:…:user/jsmith         → t_id_C

    Different strings, different tokens. Linking them requires an
    authoritative source that STATES the equivalence — Entra's
    on-premises immutable ID, an IAM SAML mapping, an Okta profile.
    The collector ships those as SAME_AS relationships; SaaS
    applies them.

    ⚠ derived from a source's assertion, NEVER from string
      similarity — which HMAC makes impossible anyway. That
      impossibility is a FEATURE: it removes the sloppy merge from
      the menu.

  STAGE 3 — GRAPH REINFORCEMENT

    two candidates sharing many neighbours are probably one entity.
    High threshold, confidence penalty propagating into path
    scoring (../analytics/04 §3.2).

    BIAS TOWARD UNDER-MERGE. Two nodes that should be one produce a
    missing path — a false negative, invisible and survivable. One
    node that should be two produces a path that does not exist —
    a false positive, which an engineer disproves in a meeting, and
    which costs the product its credibility.
```

---

## 6. What the collector left for here

```
  PERMISSION CLOSURE      needs every policy in the estate:
                          deny → SCP → resource policy → boundary →
                          session → identity. No single collector
                          has that view.

  ESCALATION PRIMITIVES   ~60 synthesized-edge patterns across
                          AWS/Azure/GCP/K8s/AD. Content, updated
                          centrally, applied to the whole graph.
                          (../02-iam-deep-dive.md)

  ATTACK PATHS            reverse BFS from crown jewels to start
                          conditions. Whole-graph. (../analytics/03)

  RISK SCORING            needs crown jewels, start conditions and
                          the full path. ESC-4b. (../analytics/04)

  CHOKE POINTS            needs the complete path set.
                          (../analytics/05)

  CSPM · DSPM · ASPM      LLD §77–80 route each of these through
                          Security Facts to SaaS. The collector
                          produces the facts; the posture judgement
                          across the estate is made here.

  CROSS-TENANT            computed on TOKENS, which is only possible
  BENCHMARKING            because no plaintext ever arrived.
```

---

## 7. Considerations

**Ack on durable, not on processed.** The ingestion API returns 202 once the
batch is in JetStream. If it waited for graph materialization, a slow correlation
pass would apply backpressure all the way to a collector's spool, and a SaaS
performance problem would become a customer-visible collection problem.

**Multi-instance on the collector side, multi-tenant here.** Each customer has
their own collectors and — under ESC-3 — their own tokenization key. SaaS holds
all tenants. That asymmetry is the correct one: the blast radius of a SaaS breach
is a set of anonymous graphs; the blast radius of a collector breach is one
customer.

**`tenant_id` is on every LLD object and must be the partition key everywhere.**
Postgres row-level security, ClickHouse partitioning, one in-memory graph per
tenant. A cross-tenant query should be structurally impossible, not prevented by
a `WHERE` clause someone might forget.

**Coverage must be first-class here or the collector's work is wasted.** If ESC-5
is accepted and SaaS's staleness sweep then ignores the windows, all of it is
discarded and the failure in `06 §10.3` happens anyway. The retraction rule
belongs in the correlation engine as an invariant with a test, not as a
convention.

**The graph must be rebuildable from Postgres alone.** It is an index. The moment
something exists only in the graph, an in-memory design becomes a durability
liability and Neo4j starts looking necessary for the wrong reason.

**Someone has to own this properly.** This document is a sketch written by the
collector side because the gap was blocking its own decisions. It is not a
substitute for a SaaS engineering handoff with phases, gates and an owner — and
until that exists, ESC-3 and ESC-4 will be settled by whichever code is written
first rather than by a decision.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Security Facts in Postgres | 13M–3.4B append-only rows/day; the database fails | ClickHouse, §3.1 |
| Ack after processing | SaaS latency becomes collector backpressure and spool growth | 202 on durable |
| Coverage windows ignored | Broken connectors improve exposure scores anyway | Retraction invariant with a test |
| Graph holds unique state | An in-memory index becomes a durability risk | Postgres is the source of truth |
| Over-eager entity merging | Paths that do not exist; credibility lost in one meeting | Bias to under-merge, §5 |
| `tenant_id` enforced by convention | A cross-tenant leak in the highest-consequence system | RLS + partitioning + per-tenant graph |
| Neo4j chosen by default | Per-instance licensing against an instance-count business | In-memory to ~50M edges, §4 |
| No SaaS handoff written | ESC-3 and ESC-4 decided by implementation order | This document, then a real one |

---

## 9. Example: Meridian

### 9.1 What four collectors become

```
  INBOUND, PER DAY
    COL-mer-01   3.28 M objects   112 MB
    COL-mer-02   2.91 M           98 MB
    COL-mer-03   4.10 M          147 MB
    COL-mer-04   0.44 M           16 MB
                ────────────────────────
                10.73 M          373 MB/day

  AFTER RESOLUTION
    entities                              2,904,118
      seen on more than one collector       412,006   (14.2%)
      resolved by TOKEN EQUALITY            411,240   (99.8%) ← stage 1
      resolved by SAME_AS assertion             744   ( 0.2%) ← stage 2
      resolved by graph reinforcement            22   (0.005%)← stage 3
    relationships                         2,887,441
      synthesized escalation edges              270
    findings open                                183

  99.8% OF CROSS-COLLECTOR RESOLUTION IS AN EQUALITY JOIN.

  Under LLD §26's masking or unkeyed hashing, ALL 412,006 would
  need stage 3 — probabilistic, confidence-penalised, biased to
  under-merge. Most would simply stay fragmented.
```

### 9.2 The path, assembled from three collectors

```
  SEC-github-token-mcp      COL-mer-02   MCP config on LT-4471
       │
       ▼                    joined by TOKEN EQUALITY
  PIP-gha-any-meridian-repo COL-mer-03   the PAT authenticating a
       │                                 workflow run
       ▼
  ROL-ghadeploy             COL-mer-03   the OIDC trust policy
       │
       ▼                    synthesized: PassRole + Lambda
  ROL-ec2app                COL-mer-03
       │
       ▼
  DST-prod-payments         COL-mer-03   criticality 95, classified
                                         by the DLP connector

  THREE COLLECTORS. ONE PATH. FIVE HOPS.

  Assembled from facts none of which contained a single plaintext
  identifier, and joined without one probabilistic merge.

  This is the product working, and it is precisely ESC-3 resolved
  in favour of keyed tokenization. Under LLD §26 as written, the
  first hop never connects to the second and this path does not
  exist.
```

---

*Series ends. [Escalations](13-escalations.md) · [Index](00-index.md).*

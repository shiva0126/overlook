# 9 — The SaaS Side

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> This document specifies the receiving half of the service
> architecture. It is **outside the collector engineer's assignment**
> (handoff §3.2) and is included because the collector's fact schema
> is determined by what this side needs, and because there is
> currently no SaaS engineering handoff at all.

---

## 1. Purpose

Receive facts from every collector of every tenant, resolve them into one graph,
compute what the collector deliberately does not, and serve it.

The reason this document exists in a collector series is stated plainly: **E1 and
E2 cannot be decided by the collector team, because they are determined by the
consumer, and the consumer has no specification.** Without one, those decisions
get made by default, by whoever writes collector code first.

---

## 2. Position

```
  INPUTS
    fact batches over mTLS from N collectors × M tenants (07)
    de-tokenization requests are NOT routed here — they go
    browser → collector, directly (06 §6)

  OUTPUTS
    the graph · attack paths · risk scores · findings · metrics
    APIs · the React UI

  CONSUMED BY
    analysts, MSSP SOC operators, customer CISOs
```

---

## 3. The receiving path

```
  ┌─────────────────────────────────────────────────────────────┐
  │  API / INGESTION                                            │
  │    mTLS or signed-JWT + payload encryption (07 §4.1)        │
  │    batch_id idempotency · sequence gap detection            │
  │    decrypt · decompress · validate against Fact Schema v1   │
  │    → 202 as soon as durable. NOT after processing.          │
  └──────────────────────────┬──────────────────────────────────┘
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  NATS JETSTREAM                                             │
  │    the same reasoning as 02 — durable, replayable           │
  │    subjects: facts.<tenant>.<type>                          │
  │    ⚠ retention here is DAYS, not hours: facts are four      │
  │      orders of magnitude smaller than raw events, and       │
  │      replay across a schema change is worth having          │
  └──────────────────────────┬──────────────────────────────────┘
                             ▼
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
  ┌───────────────┐ ┌──────────────┐ ┌──────────────────┐
  │  POSTGRES     │ │  CLICKHOUSE  │ │  GRAPH ENGINE    │
  │  entities     │ │ observations │ │  in-memory       │
  │  relationships│ │ time series  │ │  materialized    │
  │  findings     │ │ metrics      │ │  from Postgres   │
  │  tenants      │ │ trends       │ │                  │
  │  coverage     │ │ audit        │ │                  │
  └───────────────┘ └──────────────┘ └──────────────────┘
              └──────────────┼──────────────┘
                             ▼
                  CORRELATION ENGINE
                             ▼
                  RISK / FINDINGS  (../analytics/)
                             ▼
                    OVERLOOK APIs
                             ▼
                      REACT UI
```

### 3.1 Which store gets which fact type

This split is the reason there are three stores rather than one, and it maps
exactly onto the four fact types from `05 §3`.

```
  ENTITY        → Postgres.   Bounded (2.9M), mutable, relational,
                              needs constraints and transactions.

  RELATIONSHIP  → Postgres.   Bounded, mutable, bitemporal
                              (first_seen / last_seen / removed_at).
                              THE GRAPH IS DERIVED FROM THIS TABLE.

  FINDING       → Postgres.   Hundreds. Has workflow state —
                              assigned, dispositioned, suppressed.

  OBSERVATION   → ClickHouse. Unbounded, append-only, never
                              updated, queried in aggregate over
                              time. This is precisely the workload
                              a columnar store exists for, and
                              precisely the one Postgres is worst at.
```

**Put an observation in Postgres and the design fails within a quarter.** Meridian
alone produces ~11,000 observation facts per five-minute window per collector —
around 12.7M per day across four collectors, append-only, forever. That is a
ClickHouse table and a Postgres outage.

---

## 4. The graph engine

The architecture shows a `Graph Engine` beside Postgres and ClickHouse without
naming one. The choice matters and the obvious answer is probably wrong.

```
  MERIDIAN     2.9M entities · 2.9M relationships

  AS AN IN-MEMORY GO GRAPH
    node          ~120 B → 350 MB
    edge          ~80 B  → 232 MB
    adjacency     ~64 B  → 186 MB
                          ────────
    per tenant             ~770 MB

  20 tenants of Meridian's size → ~15 GB. One server.
```

```
  RECOMMENDATION — MATERIALIZE IN MEMORY FROM POSTGRES

    · Postgres is the durable source of truth. The graph is a
      derived index, rebuilt on startup and updated incrementally.
    · traversal is a pointer chase in process memory —
      microseconds, not a network round trip per hop
    · the choke-point simulation in ../analytics/05 §3.3 needs an
      overlay copy of the graph and a ~200 ms interactive budget.
      That is trivial in-process and awkward against a database.
    · no license, no cluster, no second operational discipline
    · lost on restart, and rebuilding 2.9M edges from Postgres
      takes seconds

  WHY NOT NEO4J
    · Enterprise licensing is a per-instance cost in a business
      whose margin depends on instance count
    · Community edition has no clustering and no RBAC
    · it is the right answer at 100M+ edges. Meridian is 2.9M,
      and an MSSP's largest customer is unlikely to be 30× bigger
      than its flagship one.

  REVISIT IF a single tenant exceeds ~50M edges, at which point
  memory per tenant passes ~13 GB and the calculus changes.
```

---

## 5. Multi-collector entity resolution

Escalation **E8**. Meridian has four collectors and the same entity appears on
several of them. This is where they become one node.

```
  STAGE 1 — DETERMINISTIC, and with tokenization it is ALL of it

    tokens are byte-identical across collectors for the same
    (tenant, type, value). An equality join resolves:

      COL-mer-02  t_id_7QK3M9F2XB4NRWZ8
      COL-mer-03  t_id_7QK3M9F2XB4NRWZ8
      → the same node. No inference. No confidence penalty.

    THIS IS THE SINGLE LARGEST BENEFIT OF E1 AND IT IS RARELY
    ARGUED FOR ON THESE GROUNDS. Tokenization is usually sold as
    privacy; it is at least as valuable as a resolution mechanism,
    because it converts a probabilistic merge into an equality
    check.

  STAGE 2 — CROSS-IDENTIFIER, still deterministic

    john.smith@meridian.com  → t_id_A
    jsmith (AD sAMAccountName) → t_id_B
    arn:.../user/jsmith      → t_id_C

    These are different tokens for different strings. Linking them
    requires an authoritative source that STATES the equivalence —
    Entra's on-premises immutable ID, an IAM SAML mapping, an
    Okta profile. The collector ships those statements as
    SAME_AS relationships; SaaS applies them.

    ⚠ derived from a source's own assertion, never from string
      similarity across tokens — which is impossible anyway,
      since HMAC destroys similarity. That impossibility is a
      FEATURE: it makes the sloppy merge unavailable.

  STAGE 3 — GRAPH REINFORCEMENT

    two candidate nodes sharing many neighbours are probably one
    entity. Applied with a high threshold and a confidence penalty
    that propagates into path scoring (../analytics/04 §3.2).

    BIAS TOWARD UNDER-MERGE. Two nodes that should be one produce
    a missing path — a false negative, invisible and survivable.
    One node that should be two produces a path that does not
    exist — a false positive, which an engineer will disprove in
    a meeting, and which costs the product its credibility.
```

---

## 6. What the collector deliberately left for here

```
  PERMISSION CLOSURE           needs every policy in the estate:
                               deny → SCP → resource policy →
                               boundary → session → identity.
                               No single collector has that view.
                               (escalation E4, conceded)

  ESCALATION PRIMITIVES        ~60 synthesized-edge patterns
                               across AWS/Azure/GCP/K8s/AD.
                               Content, updated centrally, applied
                               to the whole graph. (../02-iam-deep-dive.md)

  ATTACK PATH TRAVERSAL        reverse BFS from crown jewels to
                               start conditions. Whole-graph.
                               (../analytics/03)

  RISK SCORING                 needs crown jewels, start conditions
                               and the full path. (../analytics/04)

  CHOKE POINTS                 needs the complete path set.
                               (../analytics/05)

  CROSS-TENANT BENCHMARKING    computed on TOKENS, which is only
                               possible because no plaintext ever
                               arrived. (../09-deployment-and-tenancy-model.md §3)
```

---

## 7. Considerations

**Ack on durable, not on processed.** The ingestion API returns 202 once the
batch is in JetStream. If it waited for graph materialization, a slow correlation
pass would apply backpressure all the way to a collector's spool, and a SaaS
performance problem would become a customer-visible collection problem.

**Multi-instance on the collector side, multi-tenant on this side.** Each customer
has their own collectors and their own tokenization key
(`../09-deployment-and-tenancy-model.md`). SaaS holds all tenants. That asymmetry
is the correct one — the blast radius of a SaaS breach is a set of anonymous
graphs, and the blast radius of a collector breach is one customer.

**`tenant_id` is mandatory on every fact, and it is the partition key everywhere.**
Postgres row-level security, ClickHouse partitioning, one in-memory graph per
tenant. A cross-tenant query must be structurally impossible rather than
prevented by a WHERE clause someone might forget.

**Coverage must be first-class here, or the collector's work is wasted.** The
collector goes to real trouble to carry coverage windows (`05 §6`). If SaaS's
staleness sweep ignores them, every bit of that is discarded and the failure in
`05 §10.3` happens anyway. The retraction rule belongs in the correlation engine
as an invariant with a test, not as a convention.

**The graph must be rebuildable from Postgres alone.** It is an index. The moment
something exists only in the graph, an in-memory design becomes a durability
liability and Neo4j starts looking necessary for the wrong reason.

**Someone has to write this document properly.** It is a sketch, produced by the
collector team because the gap was blocking their own decisions. It is not a
substitute for a SaaS engineering handoff with its own phases, gates and owner.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Observations in Postgres | 12.7M append-only rows/day/tenant; the database fails | ClickHouse for observations, §3.1 |
| Ack after processing | SaaS latency becomes collector backpressure | 202 on durable |
| Coverage windows ignored | Broken connectors improve exposure scores anyway | Retraction invariant with a test, §7 |
| Graph holds unique state | An in-memory index becomes a durability risk | Postgres is the source of truth |
| Over-eager entity merging | Paths that do not exist; credibility lost in one meeting | Bias to under-merge, §5 |
| `tenant_id` enforced by convention | A cross-tenant leak in the highest-consequence system | RLS + partitioning + per-tenant graph |
| Neo4j chosen by default | Per-instance licensing against an instance-count business | In-memory to ~50M edges, §4 |
| No SaaS handoff written | E1/E2 decided by default by collector implementation order | This document, then a real one |

---

## 9. Example: Meridian

### 9.1 What four collectors become

```
  INBOUND, per day
    COL-mer-01   3.28 M facts   112 MB
    COL-mer-02   2.91 M facts    98 MB
    COL-mer-03   4.10 M facts   147 MB
    COL-mer-04   0.44 M facts    16 MB
                ──────────────────────
                10.73 M facts   373 MB/day

  AFTER RESOLUTION
    entities                    2,904,118
      of which seen on >1 collector    412,006   (14.2%)
      resolved by token equality       411,240   (99.8%)  ← stage 1
      resolved by SAME_AS assertion        744   ( 0.2%)  ← stage 2
      resolved by graph reinforcement       22   (0.005%) ← stage 3
    relationships               2,887,441
      synthesized escalation edges         270
    findings open                        183
    observations/day                12.7 M → ClickHouse

  99.8% OF CROSS-COLLECTOR RESOLUTION IS AN EQUALITY JOIN.
  Under a redaction design, all 412,006 would have needed
  stage 3 — probabilistic, confidence-penalised, and biased to
  under-merge, meaning most of them would simply have stayed
  fragmented.
```

### 9.2 The path, assembled from three collectors

```
  SEC-github-token-mcp        found by  COL-mer-02  (endpoint)
       │                                MCP config file on LT-4471
       ▼
  PIP-gha-any-meridian-repo   found by  COL-mer-03  (GitHub audit)
       │                                the PAT authenticating a
       │                                workflow — joined to the
       │                                above by TOKEN EQUALITY
       ▼
  ROL-ghadeploy               found by  COL-mer-03  (AWS IAM)
       │                                the OIDC trust policy
       ▼
  ROL-ec2app                  found by  COL-mer-03  (AWS IAM)
       │                                synthesized: PassRole+Lambda
       ▼
  DST-prod-payments           found by  COL-mer-03  (AWS RDS)
                              criticality 95, classified by
                              COL-mer-03's DLP connector

  THREE COLLECTORS. ONE PATH. FIVE HOPS.

  Assembled at SaaS from facts none of which contained a single
  plaintext identifier, and joined without a single probabilistic
  merge.

  This is the product working. It is also, precisely, escalation
  E1 being resolved in favour of tokenization — under masking,
  the first hop never connects to the second and this path does
  not exist.
```

---

*Series ends. Back to the [index](00-index.md).*

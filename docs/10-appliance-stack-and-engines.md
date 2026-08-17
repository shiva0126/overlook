# Overlook — Collector Technology Stack and Engine Architecture

**Version:** 0.1
**Date:** 2026-08-13
**Synthesises:** docs 01–09
**Status:** Architecture. No implementation. Library versions are pinned when build begins, not here.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. The constraints that decide everything

Before any technology choice, the five constraints that actually determine it:

```
  C1  SOLO BUILD
      One person. Every choice must minimise total work, not
      maximise elegance. Boring beats clever. (07 §5)

  C2  IT IS AN COLLECTOR
      Ships as an image into a customer's data centre or VPC.
      We control the OS. It must install without customer expertise
      and run for months untouched. Remote hands are unaffordable.

  C3  IT MUST WORK OFFLINE
      Every operational function works with no internet. (05 P6)
      No cloud dependency for core operation.

  C4  WE CANNOT SEE CUSTOMER DATA
      Debugging happens locally, by the customer or via redacted
      bundles. Nobody emails us their logs. (03 §11.5, 09 §6.2)

  C5  ONE COLLECTOR, ONE CUSTOMER
      No multi-tenancy anywhere in the stack. Physical isolation.
      (09 §2)
```

C1 and C2 pull hardest. C2 rules out anything with a heavy runtime, a complex dependency tree, or an operational burden. C1 rules out anything that needs a specialist to maintain.

---

## 2. The stack

### 2.1 Language — Go

Confirmed 2026-08-13. The reasoning, restated because it should be defensible later:

```
  ✓ Single static binary. No runtime, no interpreter, no JVM, no
    dependency hell inside a customer's air-gapped data centre.
    This is the single strongest argument for a collector.
  ✓ Cross-compiles to Linux/Windows/macOS from one machine —
    the agent needs all three.
  ✓ Concurrency model fits the workload exactly: hundreds of
    concurrent connector fetches, all IO-bound.
  ✓ First-class SDKs for every source that matters: AWS, Azure,
    GCP, Kubernetes, LDAP.
  ✓ Boring. Predictable memory, predictable performance, a small
    language one person can hold entirely in their head.

  ✗ What we give up: the Python data/ML ecosystem. Relevant only
    for the Classification Engine (DSPM), which is deferred anyway,
    and can run as a separate sidecar if it ever arrives.
```

### 2.2 Storage — Postgres, plus files

> **Handoff alignment.** This section describes **collector-local** storage
> only. The SaaS stack is fixed by the handoff: **PostgreSQL for state,
> ClickHouse for analytics** (§3.2, §4). Our earlier rejection of
> ClickHouse applied to putting a lake on the collector, and still does —
> it does not apply to SaaS, where the handoff has decided.

Confirmed 2026-08-13. But the important detail is **what does NOT go in Postgres.**

| Store | Technology | Why |
|---|---|---|
| Entity store | Postgres | relational, queried by key, needs transactions |
| **Local graph** | Postgres | adjacency + materialized closure (01 §9.4) |
| Fact store | Postgres | merge-on-write, needs upsert semantics |
| Token map | Postgres, separately encrypted | the crown jewel (04 §27) |
| Cursors / coverage | Postgres | small, transactional |
| Connector config | Postgres | small, transactional |
| **Ingest journal** | **Files** | append-only, fsync-critical, TTL'd, sequential read |
| **Outbound queue** | **Files** | append-only, ordered, segmented |
| **Evidence store** | **Files** | content-addressed, write-once, TTL'd |
| **Analytics dataset** | **Parquet + DuckDB** | columnar, compressed, queried in-process (04 §28.1) |

Putting the journal or evidence store in Postgres would be a mistake: they are append-heavy, TTL-driven, and never queried relationally. Files with segment rotation are simpler, faster, and easier to reason about under crash.

**Why Postgres and not a graph database** (01 §9.4): licensing, operational simplicity in a collector, and one person's ability to debug it at 2am. Adjacency tables plus materialized transitive closure for the two dense predicates (`CAN_ASSUME`, `MEMBER_OF`) handles our scale. The graph access layer stays behind an interface so the engine can be replaced without touching the analytics above it.

### 2.3 Libraries, by role

Named by role. Versions pinned at build start, not asserted here.

```
  POSTGRES ACCESS
    pgx            driver + connection pool. Direct, no ORM.
    sqlc           generates type-safe Go from plain SQL.
                   For a solo build this is the right trade: you
                   write SQL you can debug, and get compile-time
                   safety without an ORM's runtime surprises.
    goose          migrations. Boring, file-based, reversible.

  JOB SCHEDULING
    River          Postgres-backed job queue. We already run
                   Postgres, so this adds no new infrastructure
                   and gives durable, transactional job state —
                   exactly what connector scheduling needs.

  HTTP
    net/http       stdlib is sufficient with modern routing.
    chi            only if middleware composition gets awkward.

  OBSERVABILITY
    log/slog       stdlib structured logging.
    prometheus     metrics; the Controller reads them directly.
    OpenTelemetry  tracing — optional, valuable for pipeline
                   bottleneck analysis (05 §23).

  CONCURRENCY
    errgroup       bounded parallel work with error propagation.
    semaphore      worker pool limits per resource class.

  CRYPTO — all stdlib except one
    crypto/ed25519 fact and command signing
    crypto/hmac    deterministic tokenization
    crypto/sha256  content hashing, evidence refs
    crypto/aes     AES-256-GCM at rest
    x/crypto/hkdf  key derivation
    x/crypto/argon2 local UI password hashing
    → the entire privacy mechanism is standard library. No exotic
      cryptography, nothing to get wrong, nothing to audit deeply.

  COMPRESSION
    klauspost/compress   zstd for batches and the analytics dataset

  LOCAL ANALYTICS
    parquet-go     writes the partitioned analytics dataset
    duckdb (cgo)   EMBEDDED query engine over that Parquet.
                   A library, not a daemon — the same test the
                   message broker failed (§2.5). Zero-copy: the
                   retained dataset IS the query store.
                   ⚠ cgo dependency. It is the one place we accept
                     it, and it must not leak into the agent build,
                     which stays pure Go for cross-compilation.
                   See 04 §28.1.

  SOURCE SDKs
    aws-sdk-go-v2, azidentity + Azure SDK, google-cloud-go,
    k8s client-go, go-ldap
    → these are the bulk of the dependency tree, and they are
      unavoidable. Each connector should isolate its SDK behind
      the collector interface so a breaking SDK change touches
      one file.

  UI
    html/template + go:embed    server-rendered, embedded in binary
    HTMX                        interactivity without a build step
    → see §2.4
```

### 2.4 The UI decision — server-rendered

`05 §30 Q7` left this open. Deciding it now: **server-rendered Go templates with HTMX, embedded via `go:embed`.**

```
  WHY
    ✓ No Node toolchain, no npm tree, no build pipeline, no
      separate deploy artifact. The UI ships inside the binary.
    ✓ The Controller is a low-frequency admin tool (05 §1.1) —
      an SPA is overkill for something opened weekly.
    ✓ Works offline trivially. Nothing to fetch.
    ✓ One person maintains one language.

  WHERE IT DOES NOT APPLY
    The resolve/evidence API is not a UI. It is a tight JSON
    endpoint with reserved capacity and a p95 budget under 150ms
    (05 §28). Separate code path, same process.

  WHERE WE MIGHT REGRET IT
    Graph visualisation. Mitigated by 05 §34.2: render paths as
    linear narratives, not force-directed hairballs. If a real
    graph view is needed later, one page can be a JS island.
```

### 2.5 What we explicitly reject

```
  ✗ Kubernetes as the collector runtime
      Adds an operational dependency to every customer data centre.
      We ship a VM image, not a cluster.
  ✗ A message broker (Kafka, NATS, RabbitMQ)
      Another daemon to operate in a customer environment for
      in-process queueing we can do with channels and files.

      THE FULL REASONING, since "so many collectors" makes this
      look necessary and it is not:
        · the ingest journal is ALREADY the durable buffer, and it
          is better fitted — scoped replay with a diff against
          original output is a debugging feature a broker lacks
        · 89% of arriving volume is aggregated in memory at receive
          and never journaled. A broker would move data we have
          already decided to discard. ~187 MB/day is durably written.
        · pipeline stages are goroutine pools in ONE binary.
          Bounded channels give backpressure with no network hop
          and no serialization.
        · the many-collectors problem is a FRAMEWORK problem, not a
          transport one. It is solved by the manifest, the SDK, the
          River job queue, fair-queued worker pools and fixture
          recording. A broker solves none of those.

      WHERE A BROKER WOULD BE RIGHT, and is not yet:
        · the customer already runs one → we CONSUME from it
          (connectors/09 §3). Different from operating our own.
        · beyond Edge L, when R1/R2 span collectors (handoff §19.2)
        · MSSP console fact ingest at N customers (Mode 2)

      IF ONE IS EVER NEEDED: NATS JetStream — single Go binary,
      embeddable in-process first and split out later without
      changing client code. Not Kafka.

      DISCIPLINE THAT KEEPS THIS CHEAP: pipeline stages communicate
      through an INTERFACE, not raw channels — the same treatment
      the graph store gets, so a broker is a swap and not a rewrite.
  ✗ An ORM
      Our queries are recursive CTEs and bulk upserts. An ORM
      fights both.
  ✗ A graph database (initially)
      See 01 §9.4. Behind an interface, revisit at scale.
  ✗ Microservices
      Four processes, chosen for isolation reasons (04 §26), not
      for architectural fashion.
  ✗ An SPA framework
      See 2.4.
  ✗ A separate search engine (Elastic/OpenSearch)
      The analytics dataset is Parquet, queried by an EMBEDDED
      DuckDB — a library, not a daemon (04 §28.1). Elasticsearch
      would add a JVM, a cluster and a second full copy of the data.
```

---

## 3. The engines

The direct answer to "how many engines do we need."

### 3.1 The full inventory — 16

```
  INGESTION                              needed when
  ─────────────────────────────────      ────────────────────────────
   1  Stream Aggregator                  syslog/flow sources exist
   2  Source Fingerprint Engine          unidentified stream sources
   3  Parser Engine                      non-JSON sources (CEF/LEEF/syslog)
   4  Normalizer Engine                  always

  IDENTITY
   5  Enrichment Engine                  always
   6  Entity Resolution Engine           always  ← hardest

  DERIVATION  (04 §15 — five distinct shapes)
   7  Permission Closure Engine          any IAM source  ← hardest
   8  Escalation Matcher                 any IAM source
   9  Posture Rule Engine                findings wanted
  10  Correlation Engine                 event-sequence findings
  11  Classification Engine (DSPM)       data classification wanted

  GRAPH
  12  Graph Engine                       always

  OUTPUT
  13  Fact Builder Engine                connected mode only  ← most consequential
  14  Privacy Gate Engine                connected mode only

  CONTROL
  15  Orchestration Engine               always
  16  Response Executor                  response enabled only
```

### 3.2 What makes each an engine rather than plumbing

An engine has its own algorithm, its own state, and its own failure mode. Receivers, journals, queues and transport are plumbing — important, but they move bytes rather than making decisions.

| Engine | Algorithm | State | Fails by |
|---|---|---|---|
| Stream Aggregator | windowed tumbling aggregation | in-memory windows, spill | memory pressure |
| Fingerprint | signature match over samples | source registry cache | misidentification → wrong parser |
| Parser | declarative grammar execution | none | silent drop (must quarantine) |
| Normalizer | field mapping + canonicalization | mapping tables | timezone/enum drift |
| Enrichment | lookup joins | reads entity store | stale IP→asset binding |
| **Entity Resolution** | 3-stage deterministic → probabilistic → graph reinforcement | entity store, alias directory | **over-merge = false accusation** |
| **Permission Closure** | per-cloud policy evaluation with precedence | capability sets | wrong precedence = confidently wrong graph |
| Escalation Matcher | pattern match + precondition check | primitive catalog | false positive on unchecked precondition |
| Posture Rules | stateless predicate evaluation | rule set | noise |
| Correlation | windowed sequence/threshold | event windows | memory unbounded |
| Classification | content inspection | crawl cursors | starves other work |
| **Graph** | adjacency + transitive closure + bitemporal | the graph | closure explosion |
| **Fact Builder** | merge key + arbitration + emission policy | fact store | wrong merge key = fact explosion or collapsed reality |
| Privacy Gate | tokenize, bucket, allow-list, validate | policy, token map | **fails open = breach** |
| Orchestration | banded scheduling + quorum + rate governance | schedules, cursors | starvation, quota exhaustion |
| Response Executor | signature/nonce/TTL validation | command log | executing an unsafe action |

The four in bold are where the product lives or dies (04 §25).

---

## 4. How many engines do we actually need

Against the feasible options in `07 §5`.

### 4.1 Option C — escalation primitive engine

```
  3 ENGINES

   7  Permission Closure Engine
   8  Escalation Matcher
  12  Graph Engine (minimal — in-memory is sufficient)

  NO ingestion. NO connectors. NO resolution. NO facts. NO privacy gate.
  Input is policy documents from test fixtures.
  Output is synthesized edges plus a rationale.

  This is why Option C is the cheapest credible start: three engines,
  no collector, no persistence, no UI. And two of the three are
  required by every other option.
```

### 4.2 Option A — unmanaged AI exposure map

```
  8 ENGINES

   4  Normalizer            (light — API responses are already structured)
   5  Enrichment
   6  Entity Resolution     ← the hard one
   7  Permission Closure    ← from Option C
   8  Escalation Matcher    ← from Option C
   9  Posture Rule Engine   (the MCP/agent findings)
  12  Graph Engine          ← from Option C, now persistent
  15  Orchestration

  NOT NEEDED
   1  Stream Aggregator     — no syslog or flow sources
   2  Fingerprint           — every source is a known API
   3  Parser Engine         — every source returns JSON  ← big saving
  10  Correlation           — findings are structural, not sequential
  11  Classification        — no DSPM in scope
  13  Fact Builder          — see §5, standalone mode
  14  Privacy Gate          — see §5, standalone mode
  16  Response Executor     — read-only
```

**The Parser Engine dropping out is the largest single saving.** It is the most open-ended component in the system — declarative grammar runtime, CEF/LEEF/syslog handlers, quarantine, parse-rate monitoring, and a permanent content library (`08 T9`). None of it is needed while every source is a JSON API.

### 4.3 Option B — privacy-first identity exposure

```
  10 ENGINES = Option A's 8, plus:

  13  Fact Builder Engine
  14  Privacy Gate Engine

  Plus the transport, enrollment and de-tokenization plumbing.
  Still no parser, no stream, no correlation, no classification.
```

### 4.4 The full platform

```
  16 ENGINES. Docs 01–05 as written. Not a solo build (07 Option D).
```

---

## 5. Mode 1 — SUPERSEDED

> **⚠ The handoff removes this option.** §4 and §20 assume SaaS is always
> present: the collector uploads Security Facts, receives configuration,
> and brokers response commands from the cloud. There is no standalone
> mode in which the collector is the whole product.
>
> What survives: **the collector must keep working while SaaS is
> unreachable** (handoff §19.1) — local processing continues, the spool
> absorbs, replay follows. That is degraded operation, not a deployment
> mode.
>
> The Fact Builder and the outbound path are therefore **always
> required**, not Mode 2 additions. The section below is retained for the
> reasoning about what each engine costs, and its conclusion no longer
> applies.

## 5. The conclusion that changes v1  — SUPERSEDED, see above

Working through the engine list against `09`'s multi-instance decision surfaces something the earlier documents missed.

**If every customer gets their own dedicated collector, then for a single customer the collector can be the entire product.**

```
  MODE 1 — STANDALONE                    MODE 2 — CONNECTED
  ────────────────────────────           ──────────────────────────────
  collector holds the graph              collector builds Security Facts
  Controller UI shows findings           facts flow to the MSSP console
  everything plaintext, on-site          console holds tokens only
  no SaaS at all                         cross-customer view, fleet plane

  Engines 13 (Fact Builder) and 14       Engines 13 and 14 required
  (Privacy Gate) NOT REQUIRED

  Nothing leaves the customer's          The privacy architecture
  network. The privacy claim is          earns its keep: multiple Edges
  trivially true because there is        per customer, cross-customer
  no boundary to cross.                  benchmarking, one console.
```

**Same binary, one mode flag.** And the ordering follows naturally:

```
  Mode 1 first. It is a complete, sellable product for a single
  customer — which, under multi-instance, is every customer.

  Mode 2 when there is a second Edge Collector for one customer
  (the hybrid archetype, 09 §4 Archetype 3), or a second customer
  and a reason to see across them.
```

This is not a compromise of the architecture — it is the architecture arriving in the right order. The Security Fact contract, tokenization and de-tokenization remain the design's spine (`04 §29`) and must be **designed** now so Mode 2 is a switch rather than a rewrite. But they need not be **built** to have a working product.

It also resolves an uncomfortable tension in `07 §5`: Option B looked expensive because it front-loaded the privacy machinery. In Mode 1 that machinery is deferred without abandoning it, and the customer still gets the strongest possible version of the privacy claim — *nothing leaves at all.*

---

## 6. The data flow, consolidated

Engines named, mode boundaries marked.

```
  ┌─ SOURCES ───────────────────────────────────────────────────────┐
  │  PULL: cloud APIs, IdP, repos    PUSH: webhooks                 │
  │  STREAM: syslog, flow            AGENT: Overlook Agent          │
  └───────────────┬─────────────────────────────────────────────────┘
                  │
       ┌──────────▼──────────┐
       │  E15 ORCHESTRATION  │  bands 0-5, quorum gating, rate
       │                     │  governance, credential brokering
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │  RECEIVE  (plumbing)│──► INGEST JOURNAL (files, fsync)
       │  + E1 aggregator    │    ▲ replay source for local debugging
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │  E2 FINGERPRINT     │  stream sources only
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │  E3 PARSER          │──► QUARANTINE (never drop)
       │                     │──► EVIDENCE STORE (hashed, encrypted)
       └──────────┬──────────┘    not needed while all sources are JSON
                  │
       ┌──────────▼──────────┐
       │  E4 NORMALIZER      │  → Overlook schema, UTC, canonical enums
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │  E5 ENRICHMENT      │◄── entity store (eventually consistent)
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │  E6 ENTITY          │◄── alias directory
       │     RESOLUTION      │──► RESOLUTION REVIEW QUEUE (human)
       └──────────┬──────────┘    canonical keys assigned
                  │
                  ▼  OBSERVATION  (plaintext, entity-identified)
       ┌─────────────────────────────────────────────────┐
       │  DERIVATION                                     │
       │                                                 │
       │   E7 PERMISSION CLOSURE ──► E8 ESCALATION       │
       │      (ordered)                MATCHER           │
       │                                                 │
       │   E9 POSTURE   E10 CORRELATION   E11 CLASSIFY   │
       │      (parallel, independent)                    │
       └──────────┬──────────────────────────────────────┘
                  │  derived observations
                  ▼
       ┌─────────────────────┐
       │  E12 GRAPH ENGINE   │  entities + edges + bitemporal lifecycle
       │                     │  materialized closure for CAN_ASSUME,
       │                     │  MEMBER_OF. Coverage windows govern
       │                     │  retraction.
       └──────────┬──────────┘
                  │
       ┌──────────┴───────────────────────────────┐
       │                                          │
  MODE 1 ▼ STANDALONE                    MODE 2 ▼ CONNECTED
  ┌──────────────────┐              ┌─────────────────────┐
  │ CONTROLLER UI    │              │ E13 FACT BUILDER    │
  │ findings, paths, │              │  merge, arbitrate,  │
  │ entities — all   │              │  emission policy,   │
  │ plaintext, local │              │  retraction         │
  │ NOTHING LEAVES   │              └──────────┬──────────┘
  └──────────────────┘                         │
                                    ┌──────────▼──────────┐
                                    │ E14 PRIVACY GATE    │
                                    │  tokenize, bucket,  │
                                    │  allow-list, FAIL   │
                                    │  CLOSED on invalid  │
                                    └──────────┬──────────┘
                                               │
                                    ┌──────────▼──────────┐
                                    │ SIGN + QUEUE (files)│
                                    │ mTLS 443 outbound   │
                                    └──────────┬──────────┘
                                               ▼
                                        MSSP CONSOLE
                                        (tokens only)
```

---

## 7. Process and binary layout

Four processes (`04 §26`), one binary, one mode flag.

```
  overlook  --role=core        P4  everything except the three below
  overlook  --role=vault       P1  credential broker
  overlook  --role=worker      P2  connector execution, sandboxed
  overlook  --role=scanner     P3  DSPM crawling (deferred)

  WHY SEPARATE PROCESSES — each for one specific reason
    vault    memory isolation from connector code. A bug in a
             connector must not reach 40 sets of customer credentials.
             This is a security control, not a preference.
    worker   crash isolation and resource caps. 118 eventual
             connectors is 118 chances for a leak or hot loop.
    scanner  a classification crawl must not starve identity
             collection. Hard ceiling, lowest priority.
    core     everything else. Goroutine pools, not services.

  IPC  Unix domain sockets, length-prefixed protobuf or JSON.
       Local only, never network-exposed.

  SUPERVISION  systemd units in the collector image, with restart
               backoff and crash-loop quarantine.
```

**For Option A, only `core` and `vault` are needed at first.** Worker isolation matters at connector counts we will not have; it can be a goroutine pool inside `core` initially, behind the same interface, and split out when the count justifies it.

---

## 8. The graph in Postgres

The one place where schema shape is worth sketching, because it determines whether this works at all.

```
  entities
    token TEXT PK · type · subtype · properties JSONB
    criticality · first_seen · last_seen · removed_at
    confidence · sources TEXT[]

  edges
    from_token · to_token · predicate · properties JSONB
    weight · confidence · first_seen · last_seen · removed_at
    evidence_ref · collector_id
    PK (from_token, to_token, predicate, attr_signature)

  closure_can_assume        materialized transitive closure
    from_token · to_token · depth · min_confidence · via_path
  closure_member_of         same shape

  coverage_windows
    collector · scope · started · completed · complete BOOL · count

  INDEX STRATEGY
    btree (from_token, predicate) WHERE removed_at IS NULL
    btree (to_token, predicate)   WHERE removed_at IS NULL   ← reverse BFS
    GIN   (properties)                                        ← attribute filters
    btree (last_seen)                                         ← staleness sweeps

  THE TWO THINGS THAT MATTER
    1. Partial indexes on removed_at IS NULL. The bitemporal model
       means tombstones accumulate forever; every hot query must
       exclude them at the index level, not the predicate level.
    2. Materialized closure for the two dense predicates. Computing
       CAN_ASSUME transitively at query time is where a naive
       implementation dies. Maintain incrementally on edge change
       (02 §5.6).
```

Path finding is a recursive CTE over `edges` seeded from crown jewels, reverse-BFS, depth-bounded at 8. Everything above ~10M edges gets revisited (`01 §9.4`) — behind the interface, so it is a swap rather than a rewrite.

---

## 9. Packaging

```
  DELIVERABLE
    OVA        VMware / Hyper-V — Archetype 2 and 3 (on-prem)
    AMI        AWS — Archetype 1
    Azure/GCP images
    Container  for customers who prefer it, and for our own testing

  CONTENTS
    Debian/Ubuntu LTS minimal, read-only root where practical
    Postgres
    the overlook binary (all roles)
    systemd units
    a first-boot wizard: network, enrollment, archetype selection

  UPDATES
    binary + migrations, staged, reversible, no customer downtime
    content bundles (parsers, primitives, fingerprints) separately
    and more frequently (01 §17)
    air-gapped: signed bundle uploaded through the Controller

  SIZING (01 §10.3)
    ⚠ SUPERSEDED by the handoff ceiling (01 §10.3):
    Edge S   4 vCPU / 16 GB / 250 GB
    Edge M   8 vCPU / 32 GB / 500 GB
    Edge L  12 vCPU / 64 GB / 1 TB     ← the hard maximum
    Beyond Edge L: ADD A COLLECTOR. There is no XL.
```

---

## 10. What this means for the build

```
  THE SPINE — must exist before anything (04 §29)
    Security Fact schema · entity model + predicate enum ·
    canonical key priority rules · significant-attribute table ·
    capability/action-group mapping · connector manifest schema ·
    privacy policy schema · observation schema

    These are DESIGNED now even for Mode 1, because Mode 2 must be
    a switch and not a rewrite.

  THE ENGINES, in dependency order
    E12 Graph  →  E7 Closure  →  E8 Escalation      (Option C)
    E6 Resolution → E4 Normalizer → E5 Enrichment   (Option A adds)
    E15 Orchestration → E9 Posture
    E13 Fact Builder → E14 Privacy Gate             (Mode 2 adds)

  THE HONEST V1
    8 engines · 6 connectors · 2 processes · 1 binary · Mode 1
    Postgres, files, server-rendered UI, all crypto from stdlib.
```

---

## 11. Open decisions

```
  D1  Mode 1 first — confirmed? (§5)  This is the significant one.
  D2  Ship Postgres inside the collector image, or require the
      customer to provide one? Recommend inside — C2 says the
      collector installs without customer expertise.
  D3  sqlc vs hand-written scanning. Recommend sqlc.
  D4  River for scheduling, or a hand-rolled scheduler on Postgres?
      River is less code; a hand-rolled one is fewer dependencies.
      Recommend River unless its footprint surprises us.
  D5  Worker process split at day one, or goroutine pool until
      connector count justifies it? Recommend the latter, behind
      the same interface.
  D6  Does Mode 1 need the local path engine, or only findings and
      the entity browser? (04 Q5 is now resolved — the retained
      dataset IS queryable via DuckDB, which strengthens the case
      for a usable degraded mode)
  D7  Agent language — Go for consistency, or Rust for footprint?
      Recommend Go; the agent is thin and read-mostly (01 §12.1).
```

---

*End of document.*

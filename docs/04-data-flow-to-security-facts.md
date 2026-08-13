# Overlook — Data Flow: Connectors to Security Facts

**Version:** 0.1 (design draft)
**Date:** 2026-08-12
**Companion to:** `01-system-design.md`, `02-iam-deep-dive.md`, `03-connectors.md`
**Status:** Brainstorming / architecture. No implementation.

---

## Scope of this document

This document covers exactly one span of the system: **from the moment data enters the appliance, to the moment a signed Security Fact is queued for transmission.** Nothing after that boundary — no TrustGraph, no attack paths, no SaaS.

It answers four questions:

1. **What enters, and in what shape?**
2. **What happens to it, stage by stage?**
3. **What do we actually have to build?**
4. **What must exist before we can start building it?**

| Part | Chapters | Covers |
|---|---|---|
| I | 1–3 | The shape of the problem: reduction, ingress classes, the pipeline |
| II | 4–8 | Ingress in detail, and the durability contract |
| III | 9–18 | The ten stages, each: purpose, input, output, what we build, how it fails |
| IV | 19–24 | The Fact Builder in depth — the least specified, most important part |
| V | 25–28 | The build inventory: components, processes, stores |
| VI | 29–32 | Prerequisites: contracts, decisions, and definition of done |
| VII | 33–35 | Three end-to-end traces |

---

# PART I — THE SHAPE OF THE PROBLEM

---

## 1. What this pipeline is actually for

### 1.1 The reduction

```
   Raw ingested at the appliance      ~2.4 TB / day
   After parse + normalize            ~600 GB
   After resolution + dedup            ~40 GB
   Security Facts built               ~180 MB
   After the Privacy Gate, on the wire ~12 MB / day
```

**200,000:1.** That single number defines the entire pipeline. Every stage exists to discard information that does not change what the customer should do, while preserving — and proving — the relationships that do.

Two consequences worth holding onto:

- **This is a lossy pipeline by design.** Most stages throw work away. Anything that resists that (retaining full events, shipping raw detail upward) is fighting the architecture.
- **The value is in the last two stages.** Receive/parse/normalize are commodity engineering. Fact Build and the Privacy Gate are where the product's judgement lives, and they are the least specified parts of the existing design.

### 1.2 What a Security Fact is, restated

A Security Fact is the **only** thing that crosses the boundary out of the customer environment. It is a tokenized, signed, evidence-referenced assertion. Its full anatomy is in `01-system-design.md` §5.2; the five types are:

```
  ENTITY          this thing exists, with these properties
  RELATIONSHIP    this entity relates to that entity, this way
  PROPERTY        this attribute holds for this entity
  FINDING         this condition is worth attention
  EVENT_SUMMARY   this behaviour occurred N times in this window
```

Everything the pipeline does is in service of producing those five, correctly, with defensible confidence and a retrievable evidence trail.

---

## 2. The four ingress classes

Not all input behaves the same. Treating them uniformly is the most common way this kind of pipeline goes wrong, because their **recoverability** differs, and recoverability determines the durability contract.

```
  PULL      the appliance calls out and fetches
            connector API polling
            complete objects, self-describing, re-fetchable
            volume: LOW    value density: VERY HIGH
            if lost: just fetch again

  PUSH      the source calls in with an event
            webhooks, event grids
            single events, not re-fetchable, replayable by attacker
            volume: MEDIUM value density: HIGH
            if lost: GONE FOREVER

  STREAM    the source fires continuously, connectionless or long-lived
            syslog, NetFlow/IPFIX, sFlow
            fragmentary, lossy transport, no re-fetch
            volume: VERY HIGH  value density: LOW
            if lost: gone, but individually near-worthless

  AGENT     our own software reports in
            Overlook Agent, AI Gateway
            batched, buffered at source, re-deliverable
            volume: MEDIUM value density: HIGH
            if lost: agent re-sends from its own buffer
```

### 2.1 The design consequences

```
   PULL    needs a CURSOR, not a journal.
           Idempotent by construction. Crash mid-fetch -> refetch.
           Needs rate governance far more than durability.

   PUSH    needs DURABLE JOURNALING BEFORE ACK.
           Ack only after fsync. Needs replay protection
           (signature + nonce + timestamp window).

   STREAM  needs AGGREGATION AT RECEIVE, before anything else touches it.
           Journaling every flow record is neither possible nor useful.
           Journal the AGGREGATE, not the record.

   AGENT   needs the LIGHTEST contract, because the agent already
           buffers. Ack after journal; agent prunes on ack.
```

This is why "one ingest pipeline" is a simplification that only holds after the first two stages. Receive and Identify are class-specific. From Parse onward, everything converges.

---

## 3. The pipeline at a glance

```
                            ┌──────────────┐
   PULL ────────────────────┤              │
   (connector poll)         │              │
                            │   RECEIVE    │──► ingest journal (durable)
   PUSH ────────────────────┤      +       │
   (webhook)                │  aggregate   │
                            │  for stream  │
   STREAM ──────────────────┤              │
   (syslog / flow)          │              │
                            │              │
   AGENT ───────────────────┤              │
   (agent / gateway)        └──────┬───────┘
                                   │  raw record + provenance
                                   ▼
                            ┌──────────────┐
                            │  2 IDENTIFY  │  what is this? vendor/format/version
                            └──────┬───────┘
                                   ▼
                            ┌──────────────┐
                            │  3 PARSE     │──► quarantine on failure (never drop)
                            └──────┬───────┘
                                   ▼   typed record
                            ┌──────────────┐
                            │  4 NORMALIZE │  Overlook schema, UTC, canonical enums
                            └──────┬───────┘
                                   ▼
                            ┌──────────────┐
                            │  5 ENRICH    │◄── local entity store (lookups)
                            └──────┬───────┘
                                   ▼
                            ┌──────────────┐
                            │  6 RESOLVE   │◄── Resolution Directory
                            └──────┬───────┘    canonical keys, entity identity
                                   ▼   OBSERVATION
                            ┌──────────────┐
                            │  7 DERIVE    │  closure, posture, correlation,
                            └──────┬───────┘  escalation matching
                                   ▼   derived observations
                            ┌──────────────┐
                            │  8 FACT BUILD│◄── local fact store
                            └──────┬───────┘    merge, arbitrate, decide emission
                                   ▼   FACT (plaintext)
                            ┌──────────────┐
                            │  9 PRIVACY   │  tokenize, bucket, strip, validate
                            │    GATE      │──► quarantine on validation failure
                            └──────┬───────┘
                                   ▼   FACT (tokenized)
                            ┌──────────────┐
                            │ 10 SIGN +    │
                            │    QUEUE     │──► durable outbound queue
                            └──────────────┘

   Side outputs:
     stage 3  ──► evidence store (raw, hashed, encrypted, TTL)
     stage 5  ──► local analytics dataset (compressed, encrypted, retained)
     stage 6  ──► local entity store + token map
     stage 7  ──► local graph (entities + relationships, plaintext)
```

---

# PART II — INGRESS

---

## 4. Pull: connector polling

### 4.1 What arrives

Complete, self-describing objects from an authenticated API. The highest-value input in the system — essentially all of IAM, cloud inventory, identity, and posture arrives this way.

```jsonc
// what a collector hands to RECEIVE
{
  "provenance": {
    "connector": "aws",
    "connector_version": "2.1.0",
    "collector": "iam.roles",
    "credential_ref": "cred-aws-prod-01",
    "account_scope": "123456789012",
    "region": "ap-south-1",
    "run_id": "run_01JC8Q...",
    "fetched_at": "2026-08-12T04:00:11Z",
    "cursor": null,
    "enumeration": { "complete": true, "count": 412 }
  },
  "objects": [ /* 412 raw role objects, exactly as the API returned them */ ]
}
```

Two fields carry disproportionate weight:

- **`enumeration.complete`** — the basis of the coverage window (§24). If false, nothing downstream may tombstone.
- **`cursor`** — enables delta collection. Persisted only after the batch is durably journaled, never before.

### 4.2 What we build

```
  Connector scheduler        bands, quorum gating, jitter    (03-connectors §5)
  Worker pool                sandboxed, resource-capped
  HTTP client harness        retry, backoff, pagination, 429 handling
  Rate governor              hierarchical token buckets      (03-connectors §6)
  Credential broker          separate process, scoped handles, TTL
  Cursor store               per-collector watermarks
  Coverage tracker           enumeration completeness per collector
```

### 4.3 How it fails

```
  Auth failure       -> circuit opens after 2 attempts. NEVER retry-loop:
                        that locks the customer's service account.
  Rate limited       -> back off, reduce bucket ceiling, mark degraded
  Partial enumeration-> journal what we got, set complete=false,
                        DO NOT emit a coverage window
  Connector crash    -> supervisor restarts with backoff; 3 crashes in
                        10 minutes quarantines the connector
  Schema drift       -> objects parse but map() finds unexpected shape;
                        caught by field-presence monitoring (§11.3)
```

---

## 5. Push: webhooks

### 5.1 What arrives

Single events, unsolicited, over HTTPS. Not re-fetchable — if we drop it, it is gone.

```jsonc
{
  "provenance": {
    "source": "github",
    "delivery_id": "8f14e45f-...",      // for dedup
    "signature": "sha256=...",           // HMAC, verified before anything else
    "received_at": "2026-08-12T09:14:22.881Z"
  },
  "event_type": "repository.public",
  "payload": { /* ... */ }
}
```

### 5.2 What we build

```
  Webhook receiver       TLS termination, per-source path routing
  Signature verifier     HMAC / mTLS, per-source secrets from the broker
  Replay guard           delivery_id seen-set + timestamp window
  Durable journal writer ack ONLY after fsync
  Backpressure responder 503 with Retry-After when the journal is behind
```

### 5.3 The ack contract

```
   Receive request
     -> verify signature            (reject 401 if bad — do not journal)
     -> check replay guard          (return 200 if duplicate — idempotent)
     -> append to ingest journal
     -> fsync
     -> return 200
     -> ONLY NOW is the event ours

   Returning 200 before fsync is the classic data-loss bug.
   The source will never send it again.
```

---

## 6. Stream: syslog and flow

### 6.1 What arrives

Enormous volume, fragmentary, over a transport that may silently lose data. This is 95%+ of the byte volume and a small fraction of the value.

### 6.2 Aggregate at receive, not later

The single most important decision in the ingress design:

```
   FLOW
     Raw:  ~4.1 billion records/day
        aggregate by (src_subnet, dst_subnet, dst_port, protocol)
        in 15-minute tumbling windows, IN MEMORY, at the receiver
     Out:  ~180,000 aggregate records/day        ~23,000:1

   Journaling the aggregate, never the record.

   SYSLOG
     Cannot pre-aggregate (semantically varied), so instead:
        - per-source rate limiting at receive
        - priority classification (drop debug-class before auth-class)
        - bounded buffer with explicit shedding policy
```

### 6.3 What we build

```
  Syslog receiver      UDP + TCP + TLS, RFC3164/RFC5424 framing
  Flow receiver        NetFlow v5/v9, IPFIX, sFlow decoders + templates
  Stream aggregator    windowed, in-memory, with spill-to-disk
  Shedding policy      per-source priority classes
  Source registry      which IP:port is which source (feeds IDENTIFY)
```

### 6.4 Recommend TCP/TLS, document the UDP risk

UDP syslog loses data silently under load and cannot be made reliable. Ship a clear recommendation for TCP/TLS, and when UDP is in use, **measure and display estimated loss** (sequence gaps, receiver drop counters) rather than pretending the feed is complete.

---

## 7. Agent and gateway ingress

The easiest class, because our own software is on the other end.

```
  Transport      agent-initiated mTLS, outbound only, no listening port
  Batching       agent batches locally; buffer bounded at 24h / 200 MB
  Ack            appliance journals + fsync, then acks; agent prunes
  Ordering       per-agent sequence numbers; gaps are detectable
  Backpressure   appliance returns a slow-down hint; agent extends its
                 batch interval rather than dropping
```

The AI Gateway is the same contract with one difference: it may emit facts that are already near-final (a `PROMPT_EVENT` with classification results), because inspection happened inline. Those still traverse the full pipeline — no bypass — but they short-circuit Parse and Normalize.

---

## 8. The ingest journal

### 8.1 Why it exists

One durable, append-only record of everything accepted, before any processing. It is the difference between "we lost an hour of data during a crash" and "we replayed an hour of data after a crash."

```
  Properties
    append-only, segmented files, per-source-class
    fsync before ack for PUSH and AGENT
    batched fsync (10ms window) for STREAM — a small loss window
      is acceptable at 4 billion records/day; per-record fsync is not
    no fsync required for PULL — refetchable by cursor
    checksummed records
    retention: until processed + 24h grace, or a size cap

  Enables
    crash recovery without data loss
    reprocessing after a parser fix, WITHOUT asking the source again
    debugging a mapping bug against real data, locally
```

### 8.2 The reprocessing property is worth building for

Because we cannot ask the customer to send us their logs (the privacy architecture forbids it), the ability to **fix a parser and replay the journal locally** is the primary debugging mechanism for the whole product. Design the journal so a replay can be triggered from the local UI, scoped to a source and a time window, with output diffed against what was originally produced.

---

# PART III — THE TEN STAGES

Each stage below: purpose, input, output, what we build, how it fails.

---

## 9. Stage 1 — Receive

**Purpose:** terminate the protocol, establish provenance, get bytes durably recorded.

```
  IN   protocol-specific bytes
  OUT  journaled record + provenance envelope

  BUILD   listeners per class, TLS termination, auth verification,
          the journal writer, per-source rate limiting, the shedding policy

  FAILS   listener saturation      -> backpressure to source, shed by priority
          disk full                -> refuse new writes, alarm loudly,
                                      NEVER silently drop
          malformed framing        -> counted, sampled, discarded at framing
```

**Design rule:** nothing is acknowledged before it is durable. Nothing is processed before it is acknowledged.

---

## 10. Stage 2 — Identify

**Purpose:** determine what this data *is*, so the right parser can be selected.

For PULL and AGENT this is trivial — the connector declares it. For STREAM it is a genuine problem: a syslog line arrives from `10.4.2.17:514` and nothing says whether it is a FortiGate, a Palo Alto, or a Linux host.

```
  IN   journaled record + provenance
  OUT  record + source identity (vendor, product, format, version)

  BUILD   fingerprint engine: match first N samples against a signature set
          source registry: cache (src_ip, port) -> identity, with TTL
          manual override in the local UI for stubborn sources
          re-identification trigger when fingerprint confidence drops

  FAILS   unknown source    -> quarantine bucket, surface in UI as
                              "unidentified source, 4,200 records/hour"
          misidentification -> wrong parser, low parse rate, caught by
                              parse-rate monitoring; triggers re-identify
          version drift     -> vendor changes format in a patch release
```

Fingerprints are **content** (`01-system-design.md` §17), not code — updated without shipping a new build.

---

## 11. Stage 3 — Parse

**Purpose:** turn a bytes-or-JSON blob into a typed record.

```
  IN   record + source identity
  OUT  typed record (fields extracted, types assigned)

  BUILD   grammar-driven parser runtime (declarative, not per-vendor code)
          CEF / LEEF / RFC5424 / JSON / key-value / regex-grammar handlers
          quarantine writer with sampling
          parse-rate and field-presence monitors
          evidence store writer (raw + hash, encrypted, TTL)

  FAILS   parse failure  -> QUARANTINE, never drop. Sample retained.
          partial parse  -> emit what parsed, flag incompleteness
          format change  -> parse rate drops below baseline -> P1 to the
                            customer's Overlook operator, in the local UI
```

### 11.1 Never silently drop

The most damaging failure mode in ingestion. A parser that silently discards 40% of records produces a graph that is 40% wrong and looks perfectly healthy. Quarantine plus parse-rate alerting is the defence, and it must exist from the first parser.

### 11.2 Evidence is written here

The raw record is hashed, encrypted, and written to the evidence store with a TTL. That hash becomes `evidence.ref` on any fact derived from it. This is the only point where the raw form still exists in a retrievable shape.

### 11.3 Field-presence monitoring

```
  Baseline: src_user populated in 94% of records from this source
  Today:    3%
  ->        the vendor changed the field name.
            Parse rate is still 100%. Nothing looks broken.
            Only field-presence monitoring catches this class.
```

---

## 12. Stage 4 — Normalize

**Purpose:** map vendor-specific fields onto the Overlook schema so downstream stages never see vendor differences.

```
  IN   typed record (vendor vocabulary)
  OUT  normalized record (Overlook vocabulary)

  BUILD   field mapping runtime (declarative, per source, content-shipped)
          timestamp normalizer (timezones, epochs, formats -> UTC)
          IP/CIDR normalizer (v4/v6, embedded forms, zero-padding)
          identifier normalizers (ARN, DN, UPN, SPN, URI, resource IDs)
          case folding + trimming rules per field class
          enum canonicalizer (allow/permit/accept -> ALLOW)

  FAILS   ambiguous timezone   -> a real problem; policy per source,
                                 default to source-declared, else appliance TZ
          unmappable field     -> retained in an `extra` bag, not discarded
          enum not in mapping  -> pass through with a flag; monitored
```

**Timestamps deserve real care.** Half of correlation is temporal, and a source that reports local time without an offset will silently misalign every sequence rule against it.

---

## 13. Stage 5 — Enrich

**Purpose:** attach context the record does not carry.

```
  IN   normalized record
  OUT  normalized record + context

  BUILD   asset lookup       IP/hostname -> known asset (time-bounded!)
          identity lookup    account string -> known identity
          geo/ASN lookup     external IPs only
          threat intel tags  external indicators
          business context   ServiceNow/Workday: owner, criticality, env
          tag/label lookup   cloud tags, K8s labels

  FAILS   lookup miss     -> proceed unenriched, flag it; DO NOT block
          stale mapping   -> an IP maps to the wrong asset after DHCP churn
                             mitigate with time-bounded IP->asset bindings
```

### 13.1 The circularity, and how to live with it

Enrichment reads the entity store, which is populated by resolution, which happens *after* enrichment. That is a genuine circular dependency.

The resolution: **enrichment is best-effort and eventually consistent.** The first record from a new asset arrives unenriched; by the time the tenth arrives, the asset exists and enrichment succeeds. Nothing blocks, nothing retries in-line. Facts built from unenriched records carry lower confidence and are naturally superseded as later observations merge in.

Do **not** attempt synchronous enrichment with blocking lookups. It converts a streaming pipeline into a request-response system and will not hold at volume.

---

## 14. Stage 6 — Resolve

**Purpose:** decide *which entity* this record is about. The stage that determines whether the graph is coherent or fragmented.

```
  IN   enriched record
  OUT  OBSERVATION — an assertion tied to resolved entity identities

  BUILD   canonical key derivation (priority lists per entity type)
          deterministic matcher   (stage 1, ~70% of volume)
          probabilistic matcher   (stage 2, weighted attributes)
          graph reinforcement     (stage 3, shared-neighbour evidence)
          negative-evidence rules (pins entities apart)
          Resolution Review queue (the 0.65–0.85 band, human-adjudicated)
          Resolution Directory client (alias -> canonical key, tenant-wide)
          entity store writer
          token map writer

  FAILS   under-merge  -> a missed finding. Silent. Acceptable-ish.
          over-merge   -> a FALSE ACCUSATION. Unacceptable.
                          Bias the thresholds accordingly.
          directory unreachable -> fall back to local resolution with a
                          cached alias set; mark affected entities
                          "resolution degraded"
```

The algorithm is specified in `01-system-design.md` §8.2. What matters here is that this stage's output — the **Observation** — is the first object in the pipeline that is about *entities* rather than about *records*.

### 14.1 The Observation

```jsonc
{
  "observed_at": "2026-08-12T04:00:11Z",
  "source": { "connector": "aws", "collector": "iam.roles", "run_id": "run_01JC8Q..." },
  "subject": { "canonical_key": "email:priya.s@corp.com", "type": "IDENTITY",
               "resolution_confidence": 1.0, "resolution_method": "deterministic" },
  "predicate": "CAN_ASSUME",
  "object":  { "canonical_key": "arn:aws:iam::123456789012:role/DevOpsAdmin",
               "type": "ROLE", "resolution_confidence": 1.0 },
  "attributes": { "mechanism": "sts_assume_role", "conditions": ["aws:MultiFactorAuthPresent=true"] },
  "evidence_ref": "sha256:8a1f...c4d2",
  "source_confidence": 0.99
}
```

Note it still holds **plaintext canonical keys**. Tokenization has not happened yet, and must not — resolution, derivation, and fact merging all need plaintext to work.

---

## 15. Stage 7 — Derive

**Purpose:** produce observations that no single source reported, by computing over accumulated state.

This is the "analytics engine", and it is not one engine. It is five subsystems with different shapes:

```
  7a  PERMISSION CLOSURE       (02-iam-deep-dive §3-5)
      IN   grants + constraints from cloud connectors
      OUT  effective capability observations
      SHAPE  graph computation, CPU-heavy, bursty, incremental
      NOTE  requires a LOCAL GRAPH of entities + relationships

  7b  ESCALATION MATCHING      (02-iam-deep-dive §7-12)
      IN   effective capability sets + collected preconditions
      OUT  SYNTHESIZED relationship observations
      SHAPE  pattern matching, cheap, runs after 7a
      NOTE  content-driven; the primitive catalog ships separately

  7c  POSTURE EVALUATION
      IN   current entity/property state
      OUT  FINDING observations
      SHAPE  stateless rules over current state, batch, cheap

  7d  CORRELATION
      IN   event stream over a time window
      OUT  FINDING / EVENT_SUMMARY observations
      SHAPE  stateful, windowed, memory-bound, continuous

  7e  CLASSIFICATION (DSPM)
      IN   sampled content from data sources
      OUT  PROPERTY observations (data classes present)
      SHAPE  IO-heavy, long-running, partitioned, rolling
```

### 15.1 The local graph — a decision that must be made here

7a cannot compute a permission closure without a store of entities and relationships. **The appliance therefore holds a graph**, distinct from the one in SaaS:

```
   APPLIANCE GRAPH                 SaaS GRAPH
   plaintext canonical keys        tokens only
   this site's scope               all sites, merged
   full policy/condition detail    resolved edges + condition classes
   short retention                 bitemporal, full history
   serves closure + de-tokenize    serves paths + investigation
```

This was not stated in the earlier documents and it forces a choice: **one graph implementation deployed in two configurations, or two implementations.** One implementation is clearly correct here — but it means the graph layer must be designed for both roles from the outset rather than built for SaaS and retrofitted.

### 15.2 Ordering within Derive

```
   7a closure  ──►  7b escalation      (hard dependency)
   7c posture      independent
   7d correlation  independent, continuous
   7e classification independent, rolling

   Only 7a -> 7b is ordered. Everything else is parallel.
```

---

## 16. Stage 8 — Fact Build

The subject of Part IV. Summarised here for pipeline completeness:

```
  IN   observations (collected + derived)
  OUT  facts (plaintext, merged, arbitrated, emission-decided)

  BUILD   merge key computation, merge semantics, confidence arbitration,
          emission policy, retraction logic, coverage-window integration,
          the local fact store
```

---

## 17. Stage 9 — Privacy Gate

**Purpose:** enforce the boundary. Everything that leaves passes here; nothing bypasses it.

```
  IN   fact (plaintext)
  OUT  fact (tokenized, bucketed, stripped, validated)

  BUILD   tokenizer          deterministic HMAC, tenant key from KMS/HSM
          bucketizer         counts, volumes, durations, financials
          stripper           field-level allow-list per fact type
          schema validator   fail CLOSED
          policy engine      customer-configurable, versioned, inspectable
          outbound inspector data source for the local UI's "what left" view

  FAILS   validation failure -> QUARANTINE locally, never transmit,
                                surface in the UI with the reason
          tokenization key unavailable -> HALT outbound entirely.
                                Facts accumulate locally. Alarm.
                                Never fall back to sending plaintext.
```

### 17.1 Allow-list, not deny-list

The gate must operate on an **explicit allow-list per fact type**: only these fields may leave. A deny-list fails open — a new field added by a connector author silently ships to SaaS. An allow-list fails closed: the new field is dropped until someone deliberately permits it.

This is the single most important implementation detail of the privacy claim.

### 17.2 The customer must be able to watch it

The gate feeds the local UI's outbound inspection view: every fact that left, in the last hour or day, in its final form. That converts the privacy claim from a promise into something the customer can verify themselves — and it is worth more in a procurement review than any whitepaper.

---

## 18. Stage 10 — Sign and Queue

```
  IN   tokenized fact
  OUT  signed fact in a durable, ordered outbound queue

  BUILD   content hasher      per-fact SHA-256, included in the fact
          batch signer        Ed25519 over the batch manifest
          durable queue       ordered, segmented, encrypted at rest
          batching policy     1000 facts / 5 MB / 60 s, whichever first
          shedding policy     by fact class when the queue nears capacity
          ack/prune handler   remove only after SaaS confirms
```

### 18.1 Sign the batch, hash each fact

Per-fact Ed25519 signatures are wasteful at volume. Instead: each fact carries its own content hash; the batch carries a manifest of those hashes; the **manifest** is signed. This gives per-fact integrity verification with one signature per batch.

### 18.2 Shedding order when the queue fills

```
   at 60%  warn in local UI
   at 80%  warn + notify the Overlook operator
   at 95%  shed EVENT_SUMMARY first
   at 98%  shed PROPERTY refreshes (keep first-observations)
   at 100% keep only FINDING and RELATIONSHIP facts

   RELATIONSHIP is never shed. It is the graph.
```

---

# PART IV — THE FACT BUILDER

The least specified and most consequential component. Everything before it is mechanical; everything after it is mechanical. This is where the judgement lives.

---

## 19. Observation versus Fact

```
   OBSERVATION   one sighting, from one source, at one moment.
                 Immutable. Cheap. Numerous. Never leaves the appliance.

   FACT          the accumulated, deduplicated, confidence-arbitrated
                 assertion built from many observations.
                 Mutable. Expensive. Few. Leaves the appliance.
```

```
   14,882 observations of "svc-deploy CAN_ASSUME DevOpsAdmin"
   collected over 61 days from 3 sources
        ↓
   ONE fact, with observation_count=14882, first_seen 61 days ago,
   last_seen 4 minutes ago, sources [aws.iam, cloudtrail, access_analyzer],
   confidence 0.99
```

That collapse is where the 200,000:1 reduction is finally realised.

---

## 20. The merge key

The identity of a fact is **not** its `fact_id` (a per-emission ULID used only for transport dedup). Semantic identity is:

```
   merge_key = hash(
        // no tenant_id — one appliance, one customer (09 §2)
        fact_type,
        subject.canonical_key,
        predicate,
        object.canonical_key,
        significant_attributes_signature
   )
```

### 20.1 Which attributes are "significant"

The hard part. An attribute is significant if changing it means **this is a different assertion**, not an updated one.

```
   SIGNIFICANT (part of the key)
     mechanism           sts_assume_role vs oidc_federation are different edges
     conditions          conditional vs unconditional are different edges
     privilege_level     if it changes, the assertion changed materially
     granted_via         the path by which it was granted

   NOT SIGNIFICANT (merge, don't split)
     last_seen, observation_count, confidence
     evidence_ref        (accumulates as a list)
     source              (accumulates as a list)
     collection metadata

   The failure modes:
     TOO MANY significant -> fact explosion. Every observation creates
                             a "new" fact. The reduction never happens.
     TOO FEW significant  -> distinct realities collapse into one fact.
                             A conditional edge and an unconditional edge
                             become indistinguishable. The graph lies.
```

**Recommendation:** significance is declared **per predicate**, in the schema, not inferred. It is a small, reviewable table, and getting it wrong is expensive in both directions.

---

## 21. Granularity: the fork

Two defensible models:

```
   MODEL A — one fact per (edge)
     smallest on the wire
     provenance collapsed into a sources[] array
     disagreement between sources must be resolved BEFORE emission

   MODEL B — one fact per (edge, source)
     preserves full provenance
     SaaS sees each source's independent claim and merges centrally
     N× the volume; disagreement is visible but must be handled downstream
```

**Recommendation: Model A, with structured provenance retained inside the fact.**

```jsonc
{
  "subject": "...", "predicate": "CAN_ASSUME", "object": "...",
  "confidence": 0.99,
  "sources": [
    { "id": "aws.iam",             "confidence": 0.99, "last_seen": "...", "agrees": true },
    { "id": "aws.access_analyzer", "confidence": 0.97, "last_seen": "...", "agrees": true },
    { "id": "aws.cloudtrail",      "confidence": 0.85, "last_seen": "...", "agrees": true }
  ],
  "disagreement": false
}
```

This keeps wire volume at Model A levels while preserving what Model B was protecting: the analyst can still see *who* claimed this and whether anyone dissented. Arbitration happens at the appliance, where full plaintext context exists — which is the only place it can be done well.

---

## 22. Confidence arbitration

When sources disagree, something must decide.

```
   INPUTS
     source authority        per (connector, collector, predicate)
                             aws.iam on CAN_ASSUME  = 0.99 (authoritative)
                             cloudtrail on CAN_ASSUME = 0.85 (observational)
     resolution confidence   how sure are we this is the right entity?
     corroboration           independent sources agreeing
     recency                 decays toward the staleness horizon
     verification            provider simulation confirmed? -> 0.99

   COMBINATION
     confidence = min(resolution_confidence,
                      authority_weighted_agreement)
                  × recency_factor
                  × corroboration_boost

   min() on resolution_confidence is deliberate: a fact can never be
   more trustworthy than our certainty about WHO it is about.
```

### 22.1 Handling actual disagreement

```
   aws.iam says:        NO CAN_ASSUME edge exists
   aws.cloudtrail says: this principal assumed that role yesterday

   DO NOT silently pick one.
     - emit the fact with disagreement=true
     - confidence drops to the lower value
     - record both claims in sources[]
     - raise an internal diagnostic

   This case is diagnostically valuable: it means either a collection
   gap (we didn't see the grant) or a closure bug (we evaluated wrong).
   Its RATE is the best available proxy for closure correctness.
```

---

## 23. Emission policy

Not every merge needs to be transmitted. Without an emission policy the appliance re-sends the entire graph continuously.

```
   NEW fact                          -> emit immediately.  High value.
   SIGNIFICANT attribute changed     -> emit immediately.  Privilege changed.
   Confidence changed by > 0.05      -> emit.
   disagreement flag flipped         -> emit.
   Fact RETRACTED                    -> emit immediately.  High value.
   ONLY last_seen / count changed    -> DO NOT emit every time.
                                        Emit on heartbeat (default 24h),
                                        or when approaching the staleness
                                        horizon, whichever is sooner.
```

The last rule is the one that makes the 12 MB/day figure achievable. A stable environment produces almost no emissions — which is correct: **a graph that hasn't changed shouldn't generate traffic.**

```
   Steady state, mid-size tenant:
     ~40,000 facts held locally
     ~1,200 emitted per day (changes)
     ~1,700 emitted per day (heartbeat refresh, amortised)
     ≈ 12 MB/day compressed
```

---

## 24. Retraction and coverage windows

The most dangerous operation in the pipeline: asserting that something **no longer exists**.

```
   A fact may be retracted ONLY when a source that WOULD have
   reported it ran to completion and did NOT report it.

   REQUIRES a coverage window:
     {
       "collector": "aws.iam.roles",
       "scope": "account:123456789012",
       "started": "2026-08-12T04:00:00Z",
       "completed": "2026-08-12T04:03:12Z",
       "enumeration_complete": true,
       "object_count": 412
     }

   Given that window:
     any CAN_ASSUME fact scoped to that account, sourced from that
     collector, not observed within the window -> retract.

   WITHOUT a complete window:
     retract NOTHING.
     mark the affected subgraph STALE with the reason.
```

### 24.1 Why this matters more than it sounds

```
   Connector breaks at 04:00. Nobody notices.
   Without coverage windows: 8,400 facts go unobserved,
     get retracted, and the customer's exposure score IMPROVES.
   Their actual exposure did not change at all.

   This single bug can end the product's credibility with a customer,
   and it is entirely preventable at the Fact Builder.
```

---

# PART V — WHAT WE BUILD

---

## 25. Component inventory

| # | Component | Stage | Build / Borrow | Difficulty |
|---|---|---|---|---|
| 1 | Syslog receiver | 1 | Build (thin) | Low |
| 2 | Flow receiver + decoders | 1 | Borrow decoder, build aggregator | Medium |
| 3 | Webhook receiver | 1 | Build | Low |
| 4 | Agent gateway | 1 | Build | Medium |
| 5 | Ingest journal | 1 | Build | Medium — correctness-critical |
| 6 | Connector scheduler | 1 | Build | Medium |
| 7 | Connector worker pool + sandbox | 1 | Build | Medium |
| 8 | Rate governor | 1 | Build | Medium |
| 9 | Credential broker | 1 | Build (separate process) | Medium — security-critical |
| 10 | Cursor + coverage store | 1 | Build | Low |
| 11 | Source fingerprint engine | 2 | Build + content | Medium |
| 12 | Parser runtime | 3 | Build (declarative) | High |
| 13 | Parser grammars | 3 | Content, ongoing | Ongoing forever |
| 14 | Quarantine + parse-rate monitor | 3 | Build | Low |
| 15 | Evidence store | 3 | Build | Medium |
| 16 | Normalizer runtime + mappings | 4 | Build + content | Medium |
| 17 | Enrichment lookups | 5 | Build | Medium |
| 18 | Local analytics dataset writer | 5 | Build | Medium |
| 19 | Canonical key derivation | 6 | Build | Medium |
| 20 | Entity resolver (3-stage) | 6 | Build | **High — hardest** |
| 21 | Resolution Review queue + UI | 6 | Build | Medium |
| 22 | Resolution Directory | 6 | Build | Medium |
| 23 | Entity store | 6 | Build on Postgres | Medium |
| 24 | Token map store | 6 | Build | Medium — security-critical |
| 25 | **Local graph store** | 7 | Build (shared with SaaS) | High |
| 26 | Permission closure engine | 7a | Build | **High — hardest** |
| 27 | Escalation matcher | 7b | Build + content | Medium |
| 28 | Posture rule engine | 7c | Build + content | Medium |
| 29 | Correlation engine | 7d | Build | High |
| 30 | Classification engine | 7e | Build + content | High |
| 31 | **Fact Builder** | 8 | Build | **High — most consequential** |
| 32 | Local fact store | 8 | Build on Postgres | Medium |
| 33 | Tokenizer | 9 | Build (stdlib crypto) | Low |
| 34 | Bucketizer + stripper | 9 | Build | Low |
| 35 | Schema validator | 9 | Build | Low |
| 36 | Privacy policy engine | 9 | Build | Medium |
| 37 | Outbound inspector (UI feed) | 9 | Build | Low |
| 38 | Signer | 10 | Build (stdlib crypto) | Low |
| 39 | Outbound queue | 10 | Build | Medium |
| 40 | Sync client | 10 | Build | Medium |

**Four components carry disproportionate risk:** the entity resolver (20), the permission closure engine (26), the Fact Builder (31), and the local graph store (25). Everything else is tractable engineering. Those four determine whether the graph is *true*.

---

## 26. The process boundary

Thirty-odd concerns, but only four processes — and only where isolation buys something real:

```
  P1  CREDENTIAL BROKER
      WHY SEPARATE: holds 40+ customer credential sets. Memory isolation
      from connector code is a security control, not a preference.
      Small, stable, rarely changes.

  P2  CONNECTOR WORKERS
      WHY SEPARATE: eventual 118 connectors = 118 chances for a leak or
      hot loop. Resource caps and crash isolation per worker.
      Restarted freely by a supervisor.

  P3  SCANNER (DSPM / classification)
      WHY SEPARATE: a classification crawl will otherwise starve
      everything else. Hard resource ceiling, lowest priority,
      cannot borrow capacity.

  P4  CORE
      Everything else: receivers, journal, pipeline stages 2-10,
      stores, local graph, local UI, sync.
      One process, many goroutine pools.
```

Four processes, not one and not thirty. This is a decision worth locking before anything is laid out.

---

## 27. State stores

```
  ingest journal        append-only segments on disk
  evidence store        content-addressed, encrypted, TTL'd
  entity store          Postgres
  local graph           Postgres (same engine as SaaS, different config)
  fact store            Postgres
  token map             Postgres, separately encrypted   <- crown jewel
  cursor/coverage store Postgres
  outbound queue        durable segments on disk
  local analytics set   compressed columnar files, encrypted, TTL'd
  credential vault      separate process, KMS/HSM-wrapped
```

Postgres for the structured state. The journal, queue, evidence, and analytics dataset are file-based — they are append-heavy and TTL-driven, which is not what a relational store is for.

---

## 28. What the appliance retains locally

Worth stating explicitly, because it is a product surface, not a byproduct:

```
   Ingested            2.4 TB/day
   Shipped upward      12 MB/day
   Retained locally    everything the customer chose to retain

   evidence store       default 90 days   ~3 TB at profile M
   local analytics set  default 30 days
   entity + graph       current + 30 days of history
   fact store           current + retracted tombstones
```

The customer paid for an appliance that saw everything. What they can *do* with the retained data — query it, investigate in it, or only resolve evidence hashes against it — is an open product decision, not settled by this document.

---

# PART VI — PREREQUISITES

---

## 29. Contracts that must exist first

Nothing in Part V can be built correctly until these are fixed. They are the genuine blockers.

```
  1. SECURITY FACT SCHEMA
     the five types, field-by-field, versioned, with a registry
     and a conformance suite.
     WHY FIRST: it is the output of this entire pipeline.

  2. ENTITY MODEL + PREDICATE VOCABULARY
     closed enum of node types, subtypes, and predicates.
     WHY FIRST: resolution, derivation and fact-building all key on it.

  3. CANONICAL KEY PRIORITY RULES
     per entity type, ordered, with normalization rules.
     WHY FIRST: get this wrong and every Edge Node produces different
     tokens for the same entity, fragmenting the graph invisibly.

  4. SIGNIFICANT-ATTRIBUTE TABLE (per predicate)
     which attributes are part of the merge key.
     WHY FIRST: it defines fact identity. Changing it later
     re-partitions every fact ever built.

  5. CAPABILITY / ACTION-GROUP MAPPING
     cloud action strings -> capabilities.
     WHY FIRST: permission closure has no output vocabulary without it.

  6. CONNECTOR MANIFEST SCHEMA
     WHY FIRST: the scheduler, rate governor, health model and
     coverage tracker are all driven by it.

  7. PRIVACY POLICY SCHEMA
     per-fact-type field allow-lists, bucketing rules, tokenization
     classes, customer overrides.
     WHY FIRST: the gate fails open without it.

  8. OBSERVATION SCHEMA
     the internal contract between stages 6, 7 and 8.
     WHY FIRST: it is the seam between collection and derivation.
```

## 30. Decisions that must be made first

```
  D1  Fact granularity                 -> recommended: Model A + provenance (§21)
  D2  Significant attributes per predicate  -> declared, not inferred (§20.1)
  D3  Process boundary                 -> recommended: four processes (§26)
  D4  One graph implementation or two  -> recommended: one, two configs (§15.1)
  D5  Emission heartbeat interval      -> recommended: 24h default (§23)
  D6  Evidence retention default       -> recommended: 90 days (§28)
  D7  Journal durability per class     -> fsync PUSH/AGENT, batch STREAM,
                                          cursor-only PULL (§8)
  D8  Appliance packaging              -> OVA / AMI / container / installer
  D9  Local retention query surface    -> searchable, or evidence-lookup only
  D10 Quarantine retention + sampling rate
```

D1, D2 and D4 are the ones that are expensive to reverse. The rest can be changed later without re-partitioning stored data.

## 31. What is NOT a prerequisite

Worth stating, to avoid inventing blockers:

```
  - the attack path engine        (consumes facts; independent)
  - the SaaS console              (independent)
  - the Overlook Agent            (an ingress class; pipeline works without it)
  - the AI Gateway                (same)
  - multi-tenancy                 (single tenant per appliance by definition)
  - HA / clustering               (single node is a valid deployment)
  - the escalation primitive catalog  (content; the ENGINE is the prerequisite,
                                       the catalog grows forever)
```

## 32. Definition of done for this pipeline

Testable, not aspirational:

```
  [ ] A connector run produces facts with correct canonical keys,
      verified against a second independent source.
  [ ] 10,000 identical observations collapse to ONE fact with
      observation_count=10000.
  [ ] Replaying the ingest journal produces byte-identical facts.
  [ ] A partial connector run tombstones NOTHING and marks the
      subgraph stale.
  [ ] A complete run correctly retracts a deleted object.
  [ ] The Privacy Gate rejects a fact containing a plaintext email,
      and quarantines rather than transmits.
  [ ] The same entity observed by two appliances with the same tenant
      key produces the SAME token.
  [ ] Parse failures are quarantined and counted; parse rate is visible.
  [ ] Killing the appliance mid-batch loses no PUSH or AGENT data.
  [ ] Steady-state emission for a stable environment is near zero.
  [ ] The outbound inspector shows a customer exactly what left,
      in final form, for the last 24 hours.
```

---

# PART VII — END-TO-END TRACES

---

## 33. Trace A — an IAM policy becomes a CAN_ASSUME fact

```
 T0  SCHEDULER   band 2 opens; aws connector, iam.roles collector dispatched
 T1  BROKER      worker requests credential -> scoped handle, TTL 5 min
 T2  GOVERNOR    acquires tokens: tenant -> aws -> account -> iam -> list
 T3  FETCH       ListRoles + GetRolePolicy + trust policies, 412 roles,
                 3 pages, 1,247 API calls, 41 s
 T4  RECEIVE     journaled with provenance, enumeration.complete=true
 T5  IDENTIFY    trivial — connector declares aws/iam/v2
 T6  PARSE       JSON -> typed role objects. Trust policy documents
                 hashed and written to the evidence store.
 T7  NORMALIZE   ARNs normalized, timestamps to UTC, policy documents
                 canonicalized (statement ordering, NotAction expansion)
 T8  ENRICH      account 123456789012 -> "Production" (from Organizations)
 T9  RESOLVE     role ARN -> canonical "arn:aws:iam::123456789012:role/DevOpsAdmin"
                 principal in trust policy -> canonical
                 "email:priya.s@corp.com" via the Resolution Directory
 T10 DERIVE 7a   permission closure: identity policy + boundary + SCP
                 -> effective capability set for the principal
 T11 DERIVE 7b   escalation matcher: principal ALSO holds
                 iam:PassRole(*) + lambda:CreateFunction + InvokeFunction;
                 target role's trust policy permits lambda.amazonaws.com
                 -> SYNTHESIZED observation, primitive aws.privesc.passrole_lambda
 T12 FACT BUILD  merge_key computed. Existing fact found (first seen 61 days
                 ago). last_seen updated, observation_count 14,881 -> 14,882,
                 sources[] unchanged, confidence recomputed = 0.99.
                 Attributes unchanged -> NOT emitted (heartbeat not due).

                 The SYNTHESIZED escalation edge is NEW ->
                 emitted immediately.
 T13 PRIVACY     canonical keys -> tokens via HMAC(deployment_key, key)
                 ARN, account name, role name stripped
                 conditions retained as satisfiability classes
                 allow-list validated -> pass
 T14 SIGN+QUEUE  content hash computed, appended to batch,
                 batch manifest signed Ed25519
 T15 COVERAGE    coverage window emitted for aws.iam.roles/account:123456789012
                 -> any role fact in scope not seen this run is retracted
```

## 34. Trace B — 4.1 billion flows become one CONNECTS_TO fact

```
 T0  RECEIVE     NetFlow v9 arrives on UDP 2055, ~47,000 records/sec
 T1  AGGREGATE   in-memory tumbling window, 15 min, keyed
                 (src_subnet, dst_subnet, dst_port, protocol)
                 4.1B records/day -> ~180,000 aggregates/day
 T2  JOURNAL     the AGGREGATE is journaled, not the records
 T3  IDENTIFY    source registry: 10.4.0.9:2055 -> "core-switch-01, NetFlow v9"
 T4  PARSE       template-driven field extraction
 T5  NORMALIZE   subnets to CIDR, ports to int, protocol to canonical enum
 T6  ENRICH      10.8.0.0/24 -> "VPN pool"; 10.4.2.0/24 -> "DB subnet"
                 both time-bounded bindings
 T7  RESOLVE     subnets -> NETWORK entities; 10.4.2.17 -> AST-oracle-01
 T8  DERIVE      none required — this is a direct observation
 T9  FACT BUILD  merge_key: (NETWORK vpn-pool, CONNECTS_TO, AST-oracle-01,
                 {port:1521, protocol:tcp})
                 Existing fact. last_seen bumped. observation_count += 96.
                 No significant change -> NOT emitted.
 T10 OUTCOME     One fact. Emitted once when first observed 41 days ago,
                 and once per 24h heartbeat since.
                 4.1 billion records/day -> ~1 emission/day.
```

## 35. Trace C — an agent finds an MCP config

```
 T0  AGENT       4-hourly AI config scan on LT-4471 reads
                 ~/.claude/claude_desktop_config.json
                 -> mcp-filesystem, args ["/Users/priya/work"],
                    env contains GITHUB_TOKEN (presence only, never the value)
 T1  AGENT       batches with other findings, sends over mTLS on next heartbeat
 T2  RECEIVE     agent gateway journals + fsync, acks; agent prunes
 T3  IDENTIFY    trivial — agent declares schema version
 T4  PARSE       structured already; validated against the agent schema
 T5  NORMALIZE   paths normalized, env var names retained, VALUES DISCARDED
                 at this stage — they never enter the pipeline
 T6  ENRICH      host LT-4471 -> AST-77c2; logged-on user -> corp\priyas
 T7  RESOLVE     corp\priyas -> "email:priya.s@corp.com" via Resolution Directory
                 mcp-filesystem -> MCP_SERVER canonical
                 "mcp:filesystem@AST-77c2"
 T8  DERIVE 7c   posture rule: MCP filesystem server rooted at a directory
                 containing files classified PII (known from a prior DSPM
                 observation) -> FINDING observation
 T9  FACT BUILD  three facts:
                 ENTITY        MCP_SERVER exists, with tool list
                 RELATIONSHIP  AI_AGENT INVOKES MCP_SERVER
                 RELATIONSHIP  MCP_SERVER CAN_READ DATASTORE(/Users/priya/work)
                 FINDING       over-scoped MCP filesystem access to PII
                 All NEW -> all emitted immediately
 T10 PRIVACY     hostname, username, file paths -> tokens
                 credential PRESENCE retained as a boolean + type
                 credential VALUE was never collected
 T11 OUTCOME     SaaS learns that an agent identity can read a datastore
                 containing PII, on 14 hosts, without ever learning a
                 username, a hostname, or a file path.
```

---

## Appendix — open questions from this document

```
  Q1  Fact granularity: Model A + provenance confirmed? (§21)
  Q2  Significant-attribute table: who owns it, where does it live? (§20.1)
  Q3  One graph implementation across appliance and SaaS? (§15.1)
  Q4  Four-process boundary confirmed? (§26)
  Q5  Is the local analytics dataset queryable by the customer,
      or evidence-lookup only? (§28)
  Q6  Emission heartbeat: 24h, or tied to the staleness horizon? (§23)
  Q7  Appliance packaging: OVA, AMI, container, installer — which first?
  Q8  Correlation engine (7d): does it exist at all before the
      attack path engine, or is early correlation purely posture rules?
```

---

*End of document.*

# 5 — The Enrichment Engine

**Series:** [The Edge Collector](00-index.md) · **LLD:** §22, §40, §72, §86

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. Divergences are
> recorded in [13-escalations.md](13-escalations.md).
> Budget: **1.0 vCPU · 14 GB RAM** — the largest memory allocation
> alongside the Fact Engine (`00 §4.2`).

---

## 1. Purpose

Attach the context that turns an event into something the Fact Engine can reason
about. An event says `10.4.22.81 authenticated as jsmith`. Enrichment makes it
*the finance laptop LT-4471, owned by John Smith, employee, MFA enrolled but
excluded from the Conditional Access policy, on the corporate VLAN.*

It is also **the last stage that requires plaintext**, which fixes its position
in the pipeline and gives it a second role: it is the boundary the Privacy
Engine sits behind.

---

## 2. Position

```
  INPUTS
    normalized events (04)
    enrichment caches (§4)

  OUTPUTS
    enriched events → Security Fact Engine (06)
    cache-miss keys → background warmers
    enrichment coverage telemetry → the UI

  CONSUMED BY
    06 security fact engine

  ⚠ HARD CONSTRAINT
    Privacy (07) CANNOT run before this stage. Geo needs the real IP.
    Identity enrichment needs the real username. Threat intel needs
    the real hash. Tokenizing earlier makes every lookup miss.
```

---

## 3. The six enrichment types

LLD §22 lists them.

```
  ASSET        ip / hostname / device id  →  asset identity, owner,
                                             criticality, tags, OS
               SOURCE  CMDB, cloud inventory, EDR, MDM, DHCP

  IDENTITY     username / UPN / SID / ARN →  canonical identity,
                                             type, department, MFA
                                             state, privilege, enabled
               SOURCE  AD, Entra, Okta, IAM, Scalefusion

  CLOUD        resource id / account      →  provider, account name,
                                             region, tags, environment
               SOURCE  cloud inventory connectors

  NETWORK      ip                         →  segment, zone, VLAN,
                                             internal/external, DMZ
               SOURCE  IPAM, firewall zones, VPC/subnet config

  APPLICATION  process / port / repo      →  application, owner,
                                             business service, tier
               SOURCE  CMDB, service catalog, repo metadata

  THREAT       ip / domain / hash / url   →  known-bad, category,
                                             confidence
               SOURCE  TI feeds, synced from SaaS
```

Geo is folded into NETWORK for public addresses — an offline MaxMind-style
database, memory-mapped, ~80 MB, no network call.

### 3.1 Enrichment is lookup, never computation

```
  IN SCOPE       a cache lookup. O(1). Sub-millisecond. No network
                 call on the hot path. No I/O that can block.

  OUT OF SCOPE   reverse DNS on the hot path
                 a live CMDB call per event
                 permission closure       ← SaaS (12 §6)
                 graph traversal          ← SaaS
                 anything whose latency another system controls

  ONE BLOCKING CALL PER EVENT AT 10,000 EPS IS 10,000 CONCURRENT
  OPERATIONS AGAINST SOMEONE ELSE'S SERVER. It is how a collector
  takes down a customer's domain controller, and it will be
  attributed to Overlook.
```

---

## 4. Cache design — in-process, not Redis

LLD §40 says *"Redis is not mandatory"* and §86 lists Redis as optional-later.
That is the right call for V1 and worth arguing explicitly, because Redis is the
reflex answer.

```
  IN-PROCESS CACHE WINS HERE BECAUSE

    LLD §5 IS A MONOLITH. There is one process. A shared cache
    between processes solves a problem that does not exist.

    a map lookup is ~20 ns. A Redis round trip over loopback is
    ~50–100 µs. At 10,000 EPS × 6 enrichments that is 60,000
    lookups/sec — 6 ms/sec in-process, 3–6 SECONDS/sec via Redis.
    Redis would need its own worker concurrency to keep up.

    one fewer process to install, monitor, secure, back up and
    explain to a customer's platform team.

  REVISIT ONLY IF enrichment is ever split into its own process, or
  a warm cache must survive a collector restart. Neither is true in
  V1.
```

### 4.1 What is held, and for how long

```
  asset:<ip>            → asset record        TTL 4 h
  asset:<hostname>      → asset record        TTL 4 h
  asset:<device_id>     → asset record        TTL 24 h
  identity:<upn|sid|arn>→ identity record     TTL 1 h
  cloud:<resource_id>   → cloud record        TTL 6 h
  net:<cidr>            → radix trie          rebuilt on config change
  app:<key>             → application record  TTL 6 h
  threat:<indicator>    → TI record           TTL 6 h

  TOTAL BOUND  14 GB, enforced by configuration with LRU eviction
               per key space. NOT by available RAM.
```

```
  WHY THE TTLs DIFFER BY AN ORDER OF MAGNITUDE

  identity 1 h   the most consequential and the most volatile. A
                 disabled account that still enriches as enabled
                 produces a false relationship into a crown jewel.
                 Cheap to refresh, expensive to get wrong.

  asset 4 h      assets move slowly. DHCP-derived ip→asset is the
                 exception and gets 30 m — see §7.

  network        invalidated on config change, not by expiry.
                 Topology changes on a change-control calendar.

  threat 6 h     below every common feed interval.
```

### 4.2 Warm, do not lazy-load

```
  LAZY     the first event for an asset misses, triggers a lookup,
           and either blocks or emits unenriched. After a restart —
           and in a monolith, every upgrade is a restart — the miss
           rate across a 2.9M-entity estate is catastrophic.

  WARMED   populated from SQLite on startup, refreshed in the
           background ahead of expiry, prioritised by access
           frequency.

           startup  top 200,000 keys by recent access from SQLite
                    ~40 s, not a cold hour
           steady   refresh at 80% of TTL, in the background,
                    never on the hot path
```

**The hot path never triggers a lookup.** A miss is a miss: the record is emitted
with that enrichment absent and a marker set, and the key is queued for the
warmer. This is what keeps enrichment latency bounded and independent of every
external system's availability — and it is what makes LLD §72's *"Enrichment
unavailable → Continue without enrichment"* actually implementable.

---

## 5. Miss handling — absent beats wrong

```
  ON A MISS
    1  emit WITHOUT that enrichment
    2  mark it   enrichment.missing = ["identity"]
    3  queue the key for background resolution
    4  count it  overlook_enrichment_miss_total{type,connector}

  NEVER
    ✕ block the pipeline waiting for a lookup
    ✕ drop the record
    ✕ guess — "unknown_user", "default_asset", 0.0.0.0
    ✕ silently reuse a value past its TTL
```

```
  THE FACT ENGINE TREATS A MISSING ENRICHMENT AS UNKNOWN,
  NOT AS A VALUE.

  An event with no resolved identity produces an OBSERVATION —
  "something authenticated to X" — but no RELATIONSHIP asserting
  who. A guessed identity would produce a relationship, it would be
  wrong, and in the graph it would be indistinguishable from a
  correct one.

  Under-enriching costs coverage. Mis-enriching costs credibility.
```

---

## 6. Deduplication lives here

LLD §40 places dedup adjacent to enrichment, with a fingerprint and a TTL cache.

```
  fingerprint = hash(connector_id + source_event_id + timestamp
                     + event_type)

  in-memory TTL cache, 15 minutes, bounded entry count

  WHAT THIS CATCHES
    · syslog retransmits
    · a PULL connector refetching after a checkpoint rollback
    · webhook redelivery
    · the same event arriving on two paths

  WHAT IT DOES NOT CATCH — AND IS OFTEN CONFUSED WITH
    ten thousand DISTINCT authentication events describing ONE
    relationship. Those are not duplicates; every one is a real,
    separate event. Collapsing them is AGGREGATION, it happens in
    the Fact Engine (06 §4), and it is where the 19,000:1 reduction
    comes from.

    Dedup removes retransmits — typically 0.1–2% of volume.
    Aggregation removes redundancy — typically 99%+.
```

---

## 7. Considerations

**DHCP makes `ip → asset` a time-bounded claim, not a fact.** With an 8-hour
lease, an IP maps to different machines on different days, and a cached mapping
outliving its lease attributes one employee's traffic to another. Every
`ip → asset` carries the lease window it was valid for, and events outside it do
not use it. Without DHCP data, dynamic ranges are marked low confidence rather
than treated as reliable.

**Identity enrichment begins entity resolution and must not finish it.** The
collector maps `jsmith` to a canonical *local* identity. It must not decide that
`jsmith` in AD, `john.smith@meridian.com` in Entra and `arn:aws:...:user/jsmith`
are one person — that is cross-source resolution, it needs the whole estate, and
it is SaaS's job (`12 §5`). The collector's contribution is high-quality,
well-attributed local identifiers.

**Threat intel is enrichment, not detection.** "This IP appears on a feed" is a
property. Deciding it means something is correlation, which is SaaS. This is the
boundary that keeps behavioural detection out of the collector.

**Enrichment coverage should be reported, not just counted.** **PROPOSED:**

```
  enrichment_coverage{type} = enriched / applicable

  COL-mer-02:  asset 96.2% · identity 89.4% · network 99.8% ·
               cloud 99.1% · app 62.0% · threat 100%

  A graph built on 89% identity resolution is a different object
  from one built on 99%, and the customer should not have to guess
  which they have. This belongs beside the exposure score for the
  same reason coverage does.
```

**Cardinality is the memory risk.** `asset:<ip>` across a cloud provider's
ephemeral range is unbounded. Every key space needs a bound and an eviction
policy, and 14 GB is a ceiling the code must enforce, not a hope.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Blocking lookup on the hot path | 10,000 concurrent calls at a customer's DC | Cache-only path; misses go to a warmer |
| Guessed default values | Wrong relationships, indistinguishable from right ones | Missing is a marker, never a value |
| Stale identity (disabled account) | False path into a crown jewel | 1 h TTL, refresh at 80% |
| Stale DHCP mapping | One user's traffic attributed to another | Lease-window-bounded mappings |
| Lazy loading after a restart | Hours of unenriched records after every upgrade | Warm 200,000 keys from SQLite |
| Unbounded cache cardinality | Exceeds 14 GB; in a monolith that is an OOM of everything | Per-key-space bound + LRU |
| Redis introduced for V1 | A process, a dependency, and 3–6 s/sec of round trips | In-process, §4 |
| Dedup confused with aggregation | Either 0.1% reduction claimed as 99%, or real events dropped | §6 |
| Cross-source resolution attempted here | Wrong merges with no estate-wide view to correct them | Local identifiers only |

---

## 9. Example: Meridian

### 9.1 One record, enriched

```
  IN
    event.action    authentication_success
    source.ip       10.4.22.81
    user.name       jsmith
    timestamp       2026-08-18T09:14:22Z

  OUT
    + asset       AST-lt-4471 · Dell Latitude · Windows 11 ·
                  finance · owner CN=John Smith · criticality 34
                  via asset:10.4.22.81 (DHCP lease 09:00–17:00,
                      event inside window ✓)

    + identity    IDN-jsmith-ad · human · Finance · enabled
                  MFA enrolled ✓ · privileged ✗
                  ⚠ EXCLUDED from CA policy "Require MFA — All"

    + network     NET-corp-vlan-22 · internal · corp · not DMZ
                  via the radix trie, 10.4.16.0/20

    + application —  (not applicable)
    + threat      no indicator match

    enrichment.missing  []
```

The CA-exclusion attribute is doing more work than everything else combined. It
is what makes this identity a phishable start condition with a `× 1.30` modifier
(`../analytics/02 §3`), and it came from an enrichment a lazy design would have
skipped because the authentication succeeded and nothing looked wrong.

### 9.2 A miss, handled correctly

```
  02:04  svc-etl-nightly authenticates to AST-srv-etl-03

  identity:svc-etl-nightly  →  MISS
  (created 40 minutes ago; the Entra connector polls hourly)

  OUT   asset ✓ · network ✓ · identity —
        enrichment.missing  ["identity"]

  THE FACT ENGINE THEN
    OBSERVATION   "an unresolved principal authenticated to
                   AST-srv-etl-03, 1 time, at 02:04:11"
    RELATIONSHIP  NOT emitted — no identity node exists, so no
                  AUTHENTICATES_TO edge is asserted

  02:47  the Entra poll discovers svc-etl-nightly
  02:47  the warmer populates the cache
  02:48  subsequent events enrich, and the relationship is asserted

  The 02:04 observation is never retro-attributed. It stays
  unresolved, which is honest: at 02:04 we genuinely did not know.
```

### 9.3 The 10.6% identity gap, explained

```
  identity coverage 89.4% on COL-mer-02 — investigated, because a
  tenth of all authentications resolving to nothing is not drift.

  6.1%  local Windows accounts on non-domain-joined machines
        → genuinely in no identity source. Structural. Will never
          resolve, and must be counted separately from "not yet".

  3.2%  service accounts in an OU the Entra connector's filter
        excludes
        → A CONNECTOR CONFIGURATION DEFECT. Filter widened.
          Recovered 3.2 points.

  1.1%  accounts created and deleted inside one poll cycle
        → irreducible at 60 m. A 15 m cycle costs 4× the API quota
          to recover ~0.8%. Accepted and documented.

  0.2%  malformed usernames from one legacy application
        → dead-lettered at the parser, tracked separately

  89.4% → 92.6% by fixing ONE connector filter. The remaining 7.4%
  is now EXPLAINED rather than unknown.

  That distinction is the whole value of measuring it. An
  unexplained 10% is a reason to distrust the graph. An explained
  7.4%, of which 6.1% is structural, is a known limitation.
```

---

*Next: [Security Fact Engine](06-security-fact-engine.md)*

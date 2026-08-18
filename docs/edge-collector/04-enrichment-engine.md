# 4 — The Enrichment Engine

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 4. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **1.0 vCPU · 4 GB RAM** (plus 12 GB Redis).

---

## 1. Purpose

Attach the context that turns an event into something the Fact Engine can reason
about. An event says `10.4.22.81 authenticated as jsmith`. Enrichment makes it
`the finance-dept laptop LT-4471, belonging to John Smith, employee, MFA
enrolled, on the corporate VLAN`.

It is also **the last stage in the collector that requires plaintext**, which
makes its position in the pipeline non-negotiable and gives it a second,
architectural role: it is the boundary that the Privacy Engine sits behind.

---

## 2. Position

```
  INPUTS
    normalized records, from the Parser Engine (03)
    enrichment caches, from Redis (08 §2)

  OUTPUTS
    enriched records → Security Fact Engine (05)
    cache-miss signals → the cache warmers
    enrichment coverage telemetry → the Controller

  CONSUMED BY
    05 security fact engine

  ⚠ CONSTRAINT
    Privacy (06) CANNOT run before this stage. Geo needs the real
    IP. Identity enrichment needs the real username. Threat intel
    needs the real hash and the real domain. Tokenizing earlier
    would make every lookup miss.
```

---

## 3. The six enrichment types

```
  ASSET       ip / hostname / device id  →  asset identity, owner,
                                            criticality, tags, OS,
                                            business unit, location
              SOURCE  CMDB, cloud inventory, EDR, MDM, DHCP leases

  IDENTITY    username / UPN / SID / ARN →  canonical identity,
                                            type, department,
                                            MFA state, privilege,
                                            enabled/disabled
              SOURCE  AD, Entra ID, Okta, IAM, Scalefusion

  GEO         public ip                  →  country, ASN, org
              SOURCE  a local MaxMind-style database, offline

  THREAT      ip / domain / hash / url   →  known-bad, category,
                                            first seen, confidence
              SOURCE  TI feeds, synced from SaaS

  NETWORK     ip                         →  segment, zone, VLAN,
                                            internal/external,
                                            DMZ/prod/corp/OT
              SOURCE  IPAM, firewall zone config, cloud VPC/subnet

  TAGS        any entity                 →  customer-defined labels
              SOURCE  the Controller, customer policy
```

### 3.1 Enrichment is lookup, never computation

```
  IN SCOPE      a cache lookup. O(1). Sub-millisecond. No network
                call on the hot path. No I/O that can block.

  OUT OF SCOPE  reverse DNS on the hot path
                a live API call to the CMDB per event
                permission closure          ← handoff §3.2, SaaS
                graph traversal             ← handoff §3.2, SaaS
                anything whose latency is another system's to control

  handoff §7 says "LIGHTWEIGHT enrichment". This is the line that
  word draws, and it is worth being strict about: one blocking call
  per event at 10,000 EPS is 10,000 concurrent operations against
  someone else's server, and it is how a collector takes down a
  customer's domain controller.
```

---

## 4. Cache design

### 4.1 What lives where

```
  REDIS — 12 GB, the working set

    asset:<ip>              → asset record        TTL 4 h
    asset:<hostname>        → asset record        TTL 4 h
    asset:<device_id>       → asset record        TTL 24 h
    identity:<upn>          → identity record     TTL 1 h
    identity:<sid>          → identity record     TTL 1 h
    identity:<arn>          → identity record     TTL 1 h
    net:<cidr>              → segment record      TTL 24 h
    threat:<indicator>      → TI record           TTL 6 h
    tag:<entity_id>         → tag set             TTL 1 h

  IN-PROCESS — inside the enrichment workers

    the geo database, memory-mapped, ~80 MB, no TTL
    the network segment tree, a radix trie, rebuilt on config change
    a small LRU in front of Redis for the hottest 10,000 keys

  ⚠ REDIS HOLDS ONLY DERIVED, REBUILDABLE STATE.
    Nothing in it is authoritative. If Redis is lost, enrichment
    degrades and the caches refill. Nothing is unrecoverable.
    Durable state belongs in the Local Store (08 §3).
```

### 4.2 Why the TTLs differ by an order of magnitude

```
  identity  1 h    identity state is the most consequential and the
                   most volatile. A disabled account that still
                   enriches as enabled produces a false relationship
                   into a crown jewel. Cheap to refresh, expensive
                   to get wrong.

  asset     4 h    assets move and change owner, but slowly.
                   DHCP-derived ip→asset mappings are the exception
                   and get 30 m, because a lease turnover attributes
                   one machine's traffic to another. See §7.

  network  24 h    segment topology changes on a change-control
                   calendar, not continuously. Invalidated on
                   config change rather than by expiry.

  threat    6 h    feeds update on their own cadence; 6 h is below
                   every common feed interval.
```

### 4.3 Warming, not lazy loading

```
  LAZY      the first event for an asset misses, triggers a lookup,
            and either blocks or is emitted unenriched.
            At 10,000 EPS across a 2.9M-entity estate, the miss rate
            after a restart is catastrophic and the recovery is slow.

  WARMED    the cache is populated from the Local Store on startup
            and refreshed continuously in the background, ahead of
            expiry, prioritised by observed access frequency.

            startup     load the top 200,000 keys by recent access
                        from the Local Store — ~40 s, not a cold hour
            steady      refresh at 80% of TTL, in the background,
                        never on the hot path
```

**The hot path never triggers a lookup.** A miss is a miss: the record is
emitted with that enrichment absent and a `enrichment.missing` marker, and the
key is queued for the background warmer. This is the rule that keeps enrichment
latency bounded and independent of every external system's availability.

---

## 5. Miss handling — and why absent beats wrong

```
  ON A CACHE MISS

    1  emit the record WITHOUT that enrichment
    2  mark it:  enrichment.missing = ["identity"]
    3  queue the key for background resolution
    4  count it: enrichment_miss_total{type,connector}

  WHAT MUST NOT HAPPEN

    ✕ block the pipeline waiting for a lookup
    ✕ drop the record
    ✕ guess — "unknown_user", "default_asset", 0.0.0.0
    ✕ fall back to a stale value past its TTL without marking it
```

```
  THE FACT ENGINE TREATS A MISSING ENRICHMENT AS UNKNOWN,
  NOT AS A VALUE.

  An event with no resolved identity produces an OBSERVATION —
  "something authenticated to X" — but does NOT produce a
  relationship asserting WHO. A guessed identity would produce a
  relationship, and it would be wrong, and it would be
  indistinguishable in the graph from a correct one.

  Under-enriching costs coverage. Mis-enriching costs credibility.
```

---

## 6. Enrichment coverage as a reported metric

**PROPOSED** — not in the handoff, and the same argument as coverage windows.

```
  enrichment_coverage{type} =
      records enriched with <type>  /  records where <type> applies

  Meridian, COL-mer-02:
    asset      96.2%
    identity   89.4%    ← 10.6% of authentications resolve to
                          nothing, and every one of those is a
                          relationship the graph does not have
    network    99.8%
    geo        99.9%
    threat    100%      (a miss here means "not known bad", which
                         is a legitimate answer, not a gap)
    tags        41.3%   (expected — tagging is opt-in)

  This number belongs beside the exposure score, for the same
  reason coverage does (../analytics/06 §5.1): a graph built on
  89% identity resolution is a different object from one built
  on 99%, and the customer should not have to guess which they have.
```

---

## 7. Considerations

**DHCP makes `ip → asset` a time-bounded claim, not a fact.** In a corporate
network with an 8-hour lease, an IP maps to different machines on different
days, and a cached mapping outliving its lease attributes one employee's
traffic to another. Every `ip → asset` enrichment carries the lease window it
was valid for, and events outside it do not use it. Where DHCP data is not
available, `ip → asset` on dynamic ranges should be marked low confidence
rather than treated as reliable.

**Identity enrichment is where entity resolution begins, and it must not finish
here.** The collector maps `jsmith` → a canonical local identity. It must not
decide that `jsmith` in AD and `john.smith@meridian.com` in Entra and
`arn:aws:iam::...:user/jsmith` are the same person — that is cross-source
resolution, it needs the whole estate, and it is SaaS's job (`09 §5`). The
collector's contribution is high-quality, well-attributed local identifiers.

**Threat intel is enrichment, not detection.** A record enriched with "this IP
is on a TI feed" is a property. Deciding that property means something is
correlation, which is SaaS. This is the boundary that keeps `Detection` out of
the collector (`00 §5`).

**A cold Redis must not stall the collector.** If Redis is unavailable, all
enrichment misses, everything is marked `enrichment.missing`, facts still flow
at reduced quality, and the Controller says so. Redis is not on the critical
path for liveness — only for quality.

**Cardinality is the memory risk.** `asset:<ip>` across a /8 of cloud ephemeral
addresses is unbounded. Every cache key space needs a bound and an eviction
policy, and the 12 GB is a ceiling that Redis must be configured to respect
(`maxmemory` with `allkeys-lru`), not a hope.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Blocking lookup on the hot path | 10,000 concurrent calls against a customer's DC | Cache-only on the path; misses go to a background warmer |
| Guessed default values | Wrong relationships, indistinguishable from right ones | Missing is a marker, never a value |
| Stale identity (disabled account) | False path into a crown jewel | 1 h TTL, refresh at 80% |
| Stale DHCP mapping | One user's traffic attributed to another | Lease-window-bounded mappings |
| Cold start, lazy loading | Hours of unenriched records after a restart | Warm 200,000 keys from Local Store on startup |
| Redis down | If treated as fatal, the collector stops | Degrade to unenriched, keep flowing, report it |
| Unbounded cache cardinality | Redis exceeds 12 GB, OOM, and the ceiling is breached | `maxmemory` + LRU per key space |
| Cross-source resolution attempted here | Wrong merges with no estate-wide view to correct them | Local identifiers only; resolution is SaaS |
| Enrichment coverage unreported | Graph completeness silently varies | `enrichment_coverage` beside exposure |

---

## 9. Example: Meridian

### 9.1 One record, enriched

```
  IN, from the parser

    event.action     authentication_success
    source.ip        10.4.22.81
    user.name        jsmith
    event_time       2026-08-18T09:14:22Z
    observer.name    FortiEDR-dc-02

  OUT, enriched

    + asset          AST-lt-4471
                     Dell Latitude · Windows 11 · finance
                     owner CN=John Smith · criticality 34
                     via  asset:10.4.22.81  (DHCP lease valid
                          09:00–17:00, event inside window ✓)

    + identity       IDN-jsmith-ad
                     human · Finance · enabled
                     MFA enrolled ✓ · privileged ✗
                     ⚠ excluded from CA policy "Require MFA — All"
                     via  identity:jsmith@meridian.local

    + network        NET-corp-vlan-22
                     internal · corp · not DMZ
                     via  the radix trie, 10.4.16.0/20

    + geo            —  (RFC1918, not applicable)

    + threat         no indicator match

    + tags           ["pci-scope:no", "vip:no"]

    enrichment.missing  []
```

The `excluded from CA policy` attribute is doing more work than everything else
combined. It is what turns this identity into an S2 start condition with a
`× 1.30` modifier (`../analytics/02 §3`), and it came from an identity
enrichment that a lazy design would have skipped because the authentication
succeeded and nothing looked wrong.

### 9.2 A miss, handled correctly

```
  IN
    event.action     authentication_success
    source.ip        10.4.31.19
    user.name        svc-etl-nightly
    event_time       2026-08-18T02:04:11Z

  IDENTITY LOOKUP
    identity:svc-etl-nightly  →  MISS
    (a service account created 40 minutes ago; the Entra connector
     runs on a 60-minute PULL cycle and has not seen it yet)

  OUT
    + asset          AST-srv-etl-03
    + network        NET-prod-vlan-31
    + identity       —
    enrichment.missing  ["identity"]

  WHAT THE FACT ENGINE DOES WITH IT (05 §4)

    OBSERVATION emitted
      "an unresolved principal authenticated to AST-srv-etl-03,
       1 time, at 02:04:11"

    RELATIONSHIP  NOT emitted
      no IDN- node exists, so no AUTHENTICATES_TO edge is asserted

  02:47  the Entra PULL cycle discovers svc-etl-nightly
  02:47  the background warmer populates identity:svc-etl-nightly
  02:48  subsequent events enrich correctly and the relationship
         is asserted from then on

  The 02:04 observation is never retro-attributed. It stays as an
  unresolved observation, which is honest: at 02:04 we genuinely
  did not know who that was.
```

### 9.3 The 10.6% identity gap on COL-mer-02

```
  identity coverage 89.4% — investigated, because a tenth of all
  authentications resolving to nothing is not acceptable drift.

  BREAKDOWN OF THE MISSES
    6.1%  local Windows accounts on non-domain-joined machines
          → genuinely not in any identity source. Not a bug.
            These will never resolve and should be counted
            separately from "not yet resolved".

    3.2%  service accounts in an OU the Entra connector's filter
          excludes
          → a CONNECTOR CONFIGURATION DEFECT. Fixed by widening
            the filter. Recovered 3.2 points.

    1.1%  accounts created and deleted inside one PULL cycle
          → irreducible with a 60 m cycle. Reducing the cycle to
            15 m would cost 4× the API quota to recover ~0.8%.
            Accepted, documented.

    0.2%  malformed usernames from one legacy application
          → quarantined at the parser, tracked separately

  → 89.4% became 92.6% by fixing one connector filter, and the
    remaining 7.4% is now EXPLAINED rather than unknown.

  That distinction is the entire value of measuring it. An
  unexplained 10% is a reason to distrust the graph. An explained
  7.4%, of which 6.1% is structural, is a known limitation.
```

---

*Next: [Security Fact Engine](05-security-fact-engine.md)*

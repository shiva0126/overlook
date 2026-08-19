# 13 — Escalations Against the LLD

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. This document
> records the four places where following it as written produces a result
> we believe is wrong, with the arithmetic. **None of these are resolved
> here.** Each needs a decision from the LLD's owner.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 0. Summary

| ID | Severity | Subject | LLD | Decision needed from |
|---|---|---|---|---|
| ESC-1 | **CRITICAL** | Six-stage persistence exceeds the disk | §15, §16, §70 | LLD owner — arithmetic |
| ESC-2 | **HIGH** | Retention config is internally inconsistent | §69, §70, §71 | LLD owner — arithmetic |
| ESC-3 | **HIGH** | Unkeyed hashing is not a privacy control | §26, §27 | Product / positioning |
| ESC-4 | **MEDIUM** | Per-event facts, and a risk score the collector cannot compute | §23 | LLD owner + SaaS owner |
| ESC-5 | **MEDIUM** | Coverage windows absent | §72 | LLD owner |

ESC-1 and ESC-2 are arithmetic and should be settleable in one meeting. ESC-3 is
a product decision above the collector. ESC-4 depends on a SaaS design that does
not yet exist.

---

## ESC-1 — Six-stage persistence exceeds the disk it is allocated

**Severity: CRITICAL. Blocks the Large sizing in LLD §71.**

### The specification

LLD §16 defines the consumer chain, and §15 backs three of those subjects with
**file** storage:

```
  RAW → parser-workers → PARSED → normalization-workers → NORMALIZED
      → enrichment-workers → ENRICHED → fact-workers → SECURITY FACT
      → privacy-workers → FORWARD

  OVERLOOK_RAW          overlook.raw.*                        Storage: File
  OVERLOOK_PROCESSING   overlook.parsed, .normalized, .enriched  Storage: File
  OVERLOOK_FORWARD      overlook.fact, .forward, .retry        Storage: File
```

LLD §70 retains `Raw Queue 6–24 hours` and `Parsed Queue 6–24 hours`.

### The arithmetic

At LLD §71 "Large" — 10,000 EPS, 1 TB SSD — with the **minimum** retention §70
permits:

```
  STAGE         SIZE/EVENT   RATE       6 HOURS
  ────────────────────────────────────────────────
  RAW              1.0 KB    10 MB/s     216 GB
  PARSED           1.2 KB    12 MB/s     259 GB
  NORMALIZED       1.2 KB    12 MB/s     259 GB
  ENRICHED         1.5 KB    15 MB/s     324 GB
                                        ────────
                                        1,058 GB

  THE DISK IS 1,024 GB.

  This is before OS, binaries, SQLite state, dead letter (7 days),
  operational logs (30 days), the encrypted spool, or the FORWARD
  stream.

  At §69's stated raw_hours: 24 the write volume is ~4.2 TB/day
  against a 1 TB disk.
```

Sustained write bandwidth is the second problem: 10 + 12 + 12 + 15 = **49 MB/s of
continuous fsync-backed writes** for a workload that ingests 10 MB/s.

### Why this happens

The chain was designed as though the stages were separate services needing a
broker between them. **LLD §5 says they are not** — it is a modular monolith,
one process, all modules in one address space.

### Proposed resolution

```
  PERSIST TWICE, NOT SIX TIMES

    OVERLOOK_RAW        file-backed, fsync before ack
                        → this IS the durability contract for PUSH
                          ingress. Nothing may weaken it.

    in-process channels  parsed → normalized → enriched → fact
                        → Go channels between worker pools.
                          No serialization, no disk, no broker hop.

    OVERLOOK_FORWARD    file-backed
                        → facts awaiting SaaS acknowledgement,
                          plus the encrypted spool (§29, §35)
```

### What this costs and what it buys

```
  COSTS
    a crash loses in-flight events between RAW and FORWARD.
    → they replay from RAW, which is exactly what RAW retention
      is for. Net data loss: zero.
    → LLD §16's "each worker ACKs only after successful processing"
      is preserved: the RAW ack moves to after FORWARD publish.

  BUYS
    disk        1,058 GB → 216 GB at 6 h  (or 420 GB at 11.6 h)
    write I/O   49 MB/s → 10 MB/s
    CPU         removes 4 serialize/deserialize round trips per
                event. At 10,000 EPS that is 40,000 marshal
                operations per second recovered — plausibly the
                difference between reaching 10,000 EPS and not.
    latency     removes 4 broker round trips from the hot path
```

### If the chain must stay

If per-stage durability is required for reasons not stated in the LLD, then
either:

```
  · OVERLOOK_PROCESSING becomes memory-backed (fast, bounded,
    lost on restart — acceptable, since RAW can replay), or
  · retention on PARSED/NORMALIZED/ENRICHED drops to MINUTES,
    sized as a crash-recovery window rather than a replay window, or
  · LLD §71 "Large" is re-rated from 10,000 EPS to ~2,500 EPS,
    which is the rate at which the six-stage model fits 1 TB.
```

The third option is the one that happens by default if nothing is decided.

---

## ESC-2 — The retention configuration is internally inconsistent

**Severity: HIGH. Three stated numbers cannot all be true.**

```
  LLD §69    queue.max_disk_gb: 300
  LLD §69    retention.raw_hours: 24
  LLD §71    Large = 5,000–10,000+ EPS

  AT 10,000 EPS × 1 KB
    24 hours of RAW  =  864 GB     ✗ exceeds max_disk_gb: 300
    300 GB of RAW    =  8.3 hours  ✗ not 24

  AT 5,000 EPS
    24 hours         =  432 GB     ✗ still exceeds 300
    300 GB           =  16.6 hours

  raw_hours: 24 AND max_disk_gb: 300 ARE ONLY BOTH TRUE
  BELOW ~3,500 EPS — the bottom of the Medium tier.
```

### Proposed resolution

Retention should be **derived and reported, not configured as a constant**:

```
  · max_disk_gb is the hard limit and the only configured value
  · the collector computes and DISPLAYS the resulting window:
      "RAW retention: 11.6 hours at current 10,200 EPS"
  · the UI (LLD §35) already shows "Estimated Buffer Remaining:
    31 hours" — this is the same computation, and it should drive
    the config rather than merely report it
  · alarm when the computed window falls below a floor (default 4 h)
```

This also makes LLD §35's UI figure honest. A fixed `raw_hours: 24` that silently
becomes 8 hours under load is worse than no number, because it will be quoted in
an incident.

---

## ESC-3 — Unkeyed hashing is not a privacy control

**Severity: HIGH. Product decision, not an engineering one.**

### The specification

```
  LLD §26   "Hash identifiers where configured"
  LLD §27   sensitive fields (password, token, api_key …) removed
```

Secret removal in §27 is correct and complete. The problem is §26's hashing.

### Why it does not hold

```
  Identifiers are LOW ENTROPY. An attacker who obtains the fact
  stream can enumerate candidates and compare hashes offline.

    hostnames      DESKTOP-A4F91K, PROD-WEB-01, LT-4471
    usernames      jsmith, john.smith, svc-etl-nightly
    emails         first.last@meridian.com — the domain is known,
                   and an org's name list is often public
    private IPs    10.0.0.0/8 is 16.7M values. Exhaustible in
                   seconds.

  SHA-256 of 16.7 million IPs is a sub-second computation.
  Hashing an RFC1918 address provides NO confidentiality at all.
```

Second problem: **"where configured" means off by default**, so the shipped
behaviour is plaintext identifiers leaving the customer environment — which
contradicts LLD §1 and §88.

Third problem: **there is no de-tokenization path anywhere in the LLD.** With
hashing on, the SaaS UI displays hashes and there is no specified mechanism to
resolve them back. The feature as written makes the product unusable rather than
private.

### Proposed resolution

```
  HMAC-SHA256 WITH A TENANT KEY, NOT A BARE HASH

    token = base32(HMAC-SHA256(tenant_key, type || ":" || value))[:128 bits]
    prefixed by type:  t_id_ · t_host_ · t_ip_ · t_arn_

  PROPERTIES
    deterministic   same value → same token on every collector,
                    so SaaS joins by byte equality
    irreversible    without the key, enumeration is useless
    keyed           the key never leaves the customer environment
                    and is never sent to SaaS

  REQUIRES, and none of these are in the LLD today
    · a tenant key, generated at onboarding, sealed locally,
      distributed collector-to-collector, never via SaaS
    · a token↔value map in SQLite (see 09 §3)
    · a de-tokenization endpoint on the collector's local API,
      called by the analyst's BROWSER, not by SaaS (see 07 §6)
    · a key rotation procedure with remap facts, or rotation
      permanently splits the graph
```

### Cost

```
  CPU    ~8 tokenizable fields × 10,000 EPS = 80,000 HMAC/sec
         ≈ 1.5% of one core. Identical cost to the SHA-256 §26
         already specifies.

  DISK   token map at Meridian scale: 2.9M entries ≈ 20 GB in SQLite

  WORK   a de-tokenization endpoint, a key ceremony, a rotation
         runbook. Days, not weeks.
```

### Why it matters beyond privacy

This is the part most easily missed. Tokenization is usually argued as a privacy
control; **its larger practical value is entity resolution.** See `12 §5`:

```
  Meridian: 412,006 entities appear on more than one collector.

  WITH DETERMINISTIC TOKENS   99.8% resolve by byte equality.
                              An `==` comparison.

  WITH MASKING OR BARE HASH   all 412,006 need probabilistic
                              merging, with confidence penalties,
                              biased toward under-merge — meaning
                              most would stay fragmented, and
                              cross-collector attack paths would
                              not exist.
```

Meridian's flagship path spans three collectors. Under LLD §26 as written, it is
not discoverable.

---

## ESC-4 — Per-event facts, and a risk score the collector cannot compute

**Severity: MEDIUM. Partly self-resolving.**

### The specification

```json
  LLD §23
  {
    "fact_id": "fact-892188",
    "fact_type": "identity_access",
    "subject": { "type": "identity", "id": "john@acme.com" },
    "action": "assume_role",
    "object": { "type": "cloud_role", "id": "ProductionAdmin" },
    "severity": "high",
    "risk_score": 78,
    "observed_at": "2026-08-18T10:31:00Z"
  }
```

### Two separate problems

**(a) `observed_at` is singular — this is one fact per event.**

```
  10,000 authentication events between the same identity and the
  same role produce 10,000 facts.

  AT 10,000 EPS  →  864,000,000 facts/day, per collector.

  That is the data lake this architecture exists to avoid (LLD §1,
  §88), moved one stage later and given a different name.
```

**This is already half-solved in the LLD.** §25's Relationship Object carries
`first_seen`, `last_seen` and `confidence` — relationships are correctly
aggregated. The gap is only in §23's Security Fact, and the fix is small:

```json
  {
    "fact_type": "identity_access",
    "subject": …, "action": …, "object": …,
    "count": 10000,
    "first_seen": "2026-08-18T10:31:00Z",
    "last_seen":  "2026-08-18T10:36:00Z",
    "confidence": 0.94
  }
```

`10,000 → 1`. LLD §40 already specifies a dedup fingerprint and a TTL cache; this
is the same mechanism applied over a window rather than to exact retransmits.

**(b) `risk_score: 78` cannot be computed here.**

```
  A risk score needs
    crown jewel designation      SaaS — customer-declared
    start conditions             SaaS — whole-estate
    the full attack path         SaaS — spans collectors
    blast radius                 SaaS — whole-graph

  The collector has none of these. Whatever number it emits will
  either be a per-event severity mislabelled as risk, or it will
  disagree with the SaaS score for the same entity — and the
  customer will see both.

  → the collector should emit SEVERITY (which it can determine
    from the source and the rule) and CONFIDENCE (which only it
    can determine). SaaS computes risk. One number, one owner.
```

---

## ESC-5 — Coverage windows are absent

**Severity: MEDIUM. Cheapest to fix, most visible when missing.**

### The problem

LLD §72 marks a failed connector "degraded" locally. Nothing tells SaaS what
period was and was not observed.

```
  ABSENCE OF OBSERVATION IS NOT OBSERVATION OF ABSENCE.

  SaaS holds   IDN-svc-batch ─CAN_ASSUME→ ROL-ghadeploy
               last_seen 04:00

  It is 09:00 and the fact has not been re-observed. Two worlds:

    A  the permission was revoked at 04:00
       → remove the edge, a path closes, the score improves

    B  the connector's credentials expired at 04:00
       → nothing changed, and removing the edge INVENTS a
         security improvement

  The fact stream as specified cannot distinguish them.
```

At Meridian this is 1,847 relationships, 6 attack paths and a 19-point exposure
score movement caused by an expired credential — see `06 §10.3`.

### Proposed resolution

Every batch carries the window its source was known to be collecting for:

```json
  "coverage": {
    "connector_id": "con-aws-prod",
    "window_start": "2026-08-18T03:00:00Z",
    "window_end":   "2026-08-18T04:00:00Z",
    "completeness": "FULL",
    "gaps": []
  }
```

```
  SAAS RETRACTION RULE
    remove a relationship ONLY IF a fact arrives whose coverage
    window CONTAINS the period in question, is marked FULL, and
    does not re-assert it.
    Otherwise hold as STALE with the last coverage timestamp.

  COST
    ~120 bytes per BATCH, not per fact.
    The inputs already exist: LLD §50's overlook_connector_status,
    parse failure counters, and §52's connector health block.
```

---

## 1. Recommended order of decision

```
  ESC-1   arithmetic. Should be decidable in one meeting, and it
          blocks the Large sizing being real.

  ESC-2   arithmetic. Same meeting.

  ESC-5   cheap, self-contained, and the failure it prevents is
          the kind a customer notices in week one.

  ESC-4a  small schema change; §25 already does it correctly for
          relationships.

  ESC-3   product decision. Needs whoever owns positioning, not a
          collector design review — it determines whether Overlook
          is differentiated or is a well-built version of what
          Wiz, XM Cyber, Microsoft and Google already ship.

  ESC-4b  needs a SaaS design owner, which does not exist yet.
          Until it does, ESC-3 and ESC-4 get decided by default,
          by whoever writes collector code first.
```

---

*Back to the [index](00-index.md).*

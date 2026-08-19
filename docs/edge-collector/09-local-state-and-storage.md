# 9 — Local State and Storage

**Series:** [The Edge Collector](00-index.md) · **LLD:** §41–45, §66, §67, §70

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. Divergences are
> recorded in [13-escalations.md](13-escalations.md).
> Budget: **0.25 vCPU · 40 GB SQLite · 200 GB spool · 30 GB dead letter.**

---

## 1. Purpose

Hold what cannot be rebuilt, and be explicit about what can.

LLD §41 selects SQLite and lists what it stores. This document adds the boundary
that decides *what belongs there at all* — because without one, state lands
wherever it is convenient and the collector acquires the failure mode where
losing a cache loses something irreplaceable.

```
  IRREPLACEABLE     exists nowhere else. Loss is permanent.
                    → SQLite, backed up

  REBUILDABLE       derivable from a source or from SQLite.
                    → in-memory. Never persisted.

  DISPOSABLE        useful for a bounded time, then worthless.
                    → its own store, its own retention, its own cap
```

---

## 2. SQLite is the right choice

LLD §41 says SQLite for the first implementation. The obvious objection is
single-writer contention, and it does not apply here.

```
  LLD §5 IS A MODULAR MONOLITH. ONE PROCESS. ONE WRITER.

  SQLite's single-writer model is a constraint only when multiple
  processes contend. Inside one Go binary with a serialized write
  path it is simply how the database works.

  AND IT REMOVES
    a database process to install, monitor, patch, back up and
    explain to a customer's platform team
    a second set of credentials
    a version-compatibility matrix across the fleet
    a whole class of "the collector is up but the database is not"

  CONFIGURATION THAT IS NOT OPTIONAL
    journal_mode  = WAL        concurrent readers during writes
    synchronous   = FULL       on the tables in §3.1. Anything less
                               and a power cut loses checkpoints.
    busy_timeout  = 5000
    foreign_keys  = ON

  REVISIT ONLY IF the collector is ever split into multiple
  processes — which LLD §5 says it will not be for V1.
```

**LLD §41's warning is the important one:** *"Do not use SQLite for high-volume
raw event storage."* Raw lives in JetStream (`02`), facts in flight live in
memory, and the spool is files. SQLite holds state, and state only.

---

## 3. What SQLite holds

LLD §42 lists the tables. Grouping them by what losing each costs:

### 3.1 Irreplaceable

```
  ┌───────────────────────────────────────────────────────────────┐
  │  connector_checkpoints                          (LLD §44)     │
  │    LOSING IT: refetch from the beginning (duplicates and API  │
  │    quota) or skip forward (a permanent gap). Neither is        │
  │    acceptable, and both are avoidable.                        │
  │    → synchronous = FULL. Written before the batch is acked.   │
  │                                                               │
  │  forwarding_checkpoints                         (LLD §42)     │
  │    LOSING IT: re-send facts SaaS already has (deduplicated by  │
  │    batch_id, so survivable) or skip (permanent loss).         │
  │                                                               │
  │  certificates + keystore refs                   (LLD §42, §45)│
  │    LOSING IT: the collector cannot authenticate. Re-enrolment. │
  │                                                               │
  │  agents                                         (LLD §42, §55)│
  │    LOSING IT: every endpoint agent must re-enrol.             │
  │                                                               │
  │  audit_events                                   (LLD §54)     │
  │    LOSING IT: the record of who changed what is gone. This is  │
  │    a compliance artifact, not an operational one.             │
  │                                                               │
  │  ⚠ TOKEN MAP  — PROPOSED, does not exist in the LLD (07 §6)   │
  │    token ↔ plaintext, encrypted at rest                       │
  │    LOSING IT: EVERY FACT EVER SHIPPED BECOMES PERMANENTLY     │
  │    UNREADABLE. The SaaS graph survives and is anonymous       │
  │    forever. The most important 20 GB in the product.          │
  └───────────────────────────────────────────────────────────────┘
```

### 3.2 Expensive to rebuild

```
  entity + relationship state   what has been emitted, used to decide
                                whether something CHANGED (06 §3)
                                → losing it re-emits everything once.
                                  A volume event, not a loss event.
                                  SaaS deduplicates.

  fact window checkpoints       every 30 s (06 §8)
                                → losing them emits one window
                                  unmerged. Bounded.
                                  ⚠ MUST BE ENCRYPTED — the windows
                                    hold plaintext (07 §7)

  cache warm set                top keys by access, for 05 §4.2
                                → losing it costs a slow start
```

### 3.3 Operational

```
  collectors · connectors · parsers · settings   (LLD §42, §43)
  response_commands                              (LLD §42, §57)
  health state                                   (LLD §41)
```

### 3.4 Sizing

```
  token map          2.9M × ~180 B, indexed              ~20 GB
  entity state       2.9M × ~150 B                       ~ 8 GB
  relationship state 2.9M × ~120 B                       ~ 5 GB
  audit + response   90 days                             ~ 2 GB
  config + parsers + agents + certs                      ~ 1 GB
  indexes, WAL, free pages                               ~ 4 GB
                                                       ────────
                                                         ~40 GB
```

**The token map is half the database and 100% of the product's readability.**

---

## 4. Backups — and the one that must never leave

```
  BACK UP        token map · keystore reference · connector
                 checkpoints · certificates · agents · audit

  FREQUENCY      HOURLY, not daily. The map is append-mostly, so an
                 incremental costs almost nothing, and the backup
                 window is the only window in which loss is possible
                 at all. Worked example in §9.3.

  DESTINATION    CUSTOMER-CONTROLLED STORAGE, INSIDE THEIR
                 ENVIRONMENT

  ENCRYPTION     at rest, with a key the customer holds

  ⚠ NEVER to Overlook SaaS, never to an Overlook-controlled bucket,
    never through an Overlook-operated backup service.

    A backup of the token map in Overlook's possession is the entire
    privacy architecture undone by an operational convenience, and
    it is the first thing a serious security review will look for.
```

---

## 5. The credential vault

LLD §45 specifies reference-not-value, and LLD §11 shows `vault_ref` in the
connector config. Both are right.

```
  CONFIG HOLDS    "auth": { "type": "assume_role",
                            "vault_ref": "vault://aws-prod" }
  NEVER           "password": "MySecretPassword"

  LLD §11: "Credentials must never be returned through normal
            management APIs."

  WHICH MEANS, CONCRETELY
    · GET /api/v1/connectors/{id} returns the vault_ref, never the
      secret — not even redacted, because a redacted field tells an
      attacker the shape
    · POST accepts a secret and returns a ref. One direction only.
    · LLD §48's connector test uses the credential without exposing
      it — the test result says "authentication: success", never
      what was used
    · LLD §53's automatic log redaction covers the vault, and the
      test for it belongs in CI

  AT REST        LLD §67 — /etc/overlook/secrets owned by the
                 overlook user, directory 700, individual files 600,
                 encrypted with a key from the keystore opened at
                 startup (LLD §8)
```

---

## 6. Retention

LLD §70 sets the policy. Restated against the disk budget in `00 §4.3`:

```
  STORE            LLD §70           ALLOCATION   NOTE
  ───────────────────────────────────────────────────────────────
  Raw queue        6–24 hours        420 GB       ⚠ ESC-2 —
                                                   derive from EPS
  Parsed queue     6–24 hours        —            ⚠ ESC-1 — should
                                                   not persist
  Security facts   72 h or until     60 GB        OVERLOOK_FORWARD
                   acknowledged                    + spool
  Dead letter      7 days             30 GB       ⚠ needs sampling
                                                   (03 §7)
  Operational logs 30 days            20 GB       LLD §53
  SQLite state     —                  40 GB
  Spool            until delivered   200 GB       LLD §35
```

```
  "ALL DISK-BACKED SENSITIVE DATA SHOULD BE ENCRYPTED"  — LLD §70

  THIS APPLIES TO MORE THAN THE SPOOL:

    OVERLOOK_RAW      holds raw customer telemetry. Highest
                      sensitivity on the box.
    dead letter       holds raw excerpts by definition
    fact checkpoints  hold plaintext window state (07 §7)
    SQLite            holds the token map

  In practice this argues for full-volume encryption on
  /var/lib/overlook rather than per-store encryption — simpler,
  harder to get wrong, and it survives someone adding a new store
  and forgetting.
```

---

## 7. Filesystem layout

LLD §66 and §67:

```
  /opt/overlook/     bin/ · web/ · parsers/ · scripts/     read-only
  /etc/overlook/     collector.yaml · certificates/ · secrets/
                     secrets/ → 700, files 600, owner overlook
  /var/lib/overlook/ state/ · nats/ · spool/ · updates/
                     ← the encrypted volume
  /var/log/overlook/ 30 days (LLD §53)
```

```
  PROPOSED — SEPARATE VOLUMES FOR nats/ AND spool/.

  The spool writes in bursts during an outage; NATS needs
  consistent low-latency fsync (02 §8). On one volume, a spool
  drain after a four-day outage competes with live ingestion at
  exactly the moment the collector is catching up on everything.

  Where separate volumes are not possible, I/O priority: NATS above
  spool, always.
```

---

## 8. Considerations

**Migrations must be forward-only and online.** LLD §7 has `migrations/`. In a
monolith, a schema migration that blocks stops *collection*, and per `01 §4.3`
that means STREAM data is lost for its duration. Additive columns, background
backfills, no blocking rewrite of the token map.

**SQLite is an appliance component, not a database deployment.** No external
connections, no ad-hoc access, no extensions beyond what ships. It should be as
invisible as the filesystem, and its configuration belongs in the image rather
than in customer-tunable settings.

**Every store needs a bound in configuration, not in hope.** JetStream has
`max_bytes`. SQLite has retention per table. Dead letter has sampling and a cap.
The spool has a percentage ladder. A hard ceiling that depends on nothing growing
unexpectedly is not a ceiling.

**Local analytics is Phase 3 and should stay there.** LLD §84 places it
correctly. When it arrives it must be a *reduced projection* — tokenized
identifiers, ~14 bytes/event columnar, 7 days — not parsed events, and it must be
starvable under load. The pre-handoff design's 30-day raw dataset does not fit
1 TB and never did.

**Do not let dead letter become a shadow copy of raw.** 7 days at 3 EPS of
unparseable data with full payloads is 30 GB of one repeated shape. Sample to
1,000 distinct shapes per connector per day and store 200-byte excerpts
(`03 §7`).

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Token map lost, no backup | Every fact ever shipped becomes permanently unreadable | Hourly encrypted customer-side backup |
| Token map backup sent to SaaS | The entire privacy architecture voided | Never leaves the environment, §4 |
| `synchronous` below FULL on checkpoints | A power cut loses the fetch position | FULL on §3.1 tables |
| Checkpoint lost | Duplicate refetch or a permanent gap | Written before the batch is acked |
| Fact window checkpoint unencrypted | Plaintext at rest, bypassing the Privacy Engine | Encrypt; or full-volume encryption |
| Credentials returned by the API | Secret exposure through a management call | Reference only, one direction, §5 |
| Blocking migration | STREAM data lost for its duration | Forward-only, additive, online |
| Dead letter unsampled | 30 GB consumed in hours by one shape | Sampling + excerpts |
| Spool and NATS on one volume | fsync latency spikes during a drain | Separate volumes or I/O priority |
| SQLite exposed for ad-hoc access | An unversioned schema change nobody can trace | Appliance component, no external access |

---

## 10. Example: Meridian

### 10.1 Footprint on COL-mer-01, day 40

```
  SQLITE                                    38.4 GB / 40 GB
    token map                     19.1 GB   2.84 M entries
    entity state                   7.9 GB
    relationship state             4.8 GB
    audit + response               1.8 GB   90 days
    config, parsers, agents        0.9 GB
    indexes + WAL                  3.9 GB

  JETSTREAM OVERLOOK_RAW                   401 GB / 420 GB
                                            → ~10.6 h at 10,900 EPS
  JETSTREAM OVERLOOK_FORWARD                 2 GB / 60 GB
  DEAD LETTER                              1.4 GB / 30 GB   (sampled)
  SPOOL                                        0 / 200 GB
  LOGS                                     6.2 GB / 20 GB

  TOTAL                                    449 GB of 790 GB allocated
```

### 10.2 The 200 GB question

```
  The spool is empty and allocated 200 GB. §8.1 of doc 08 shows it
  would take twelve years to fill. The obvious move is to take it
  back.

  GIVE IT TO OVERLOOK_RAW.

  RAW at 420 GB is 10.6 hours on COL-mer-01 and only 4.4 hours on
  COL-mer-03 (02 §10.1). The parser incident in 03 §10.2 took
  7 h 20 m to detect, diagnose and fix.

  REALLOCATED
    spool          200 GB → 60 GB     still 3+ years of outage
    OVERLOOK_RAW   420 GB → 560 GB    COL-mer-01 → 14 h
                                      COL-mer-03 →  5.9 h

  WHAT IT BUYS   a full working day to detect, diagnose, fix and
                 replay before data is unrecoverable
  WHAT IT COSTS  nothing that has ever been used

  Retention on a buffer that PREVENTS PERMANENT LOSS is worth more
  than retention on a store that has never exceeded 0.14%. This is
  the sort of reallocation the fleet should be reviewed for after
  sixty days in production — and it is only visible because the
  budget was written down.
```

### 10.3 A token map restore

```
  DRILL, WEEK 6. COL-mer-02's disk is deliberately destroyed.

  BEFORE   2.84 M tokens issued, all facts shipped and in the graph

  09:00  new collector provisioned from the image (LLD §66)
  09:04  token map restored from Meridian's encrypted backup —
         19.1 GB, taken 03:00, SIX HOURS OLD
  09:11  keystore unsealed from Meridian's HSM (LLD §8)
  09:12  connector checkpoints restored, 6 h stale → 6 h of PULL
         data refetched. SaaS deduplicates by batch_id.

  09:14  ⚠ 6 HOURS OF TOKENS ISSUED SINCE THE BACKUP ARE MISSING.
         ~2,100 entities discovered between 03:00 and 09:00 are in
         the SaaS graph, correctly joined, and CANNOT BE
         DE-TOKENIZED.

  09:15  they are re-discovered on the next full enumeration and
         re-tokenized — TO THE SAME TOKENS, because the HMAC is
         deterministic and the key is unchanged (07 §4.5).

  10:40  enumeration completes across all connectors. The map is
         whole. Nothing is lost.

  THE PROPERTY THAT SAVED IT: DETERMINISM.

  A random token map — which is what you get if nobody writes this
  down — would have made those 2,100 entities permanently
  unreadable.

  THE RUNBOOK CHANGE
    back up hourly, not daily. It is 19 GB and append-mostly; the
    incremental is minutes, and it closes the only window in which
    loss is possible at all.
```

---

*Next: [Control Plane](10-control-plane.md)*

# Collectors — Anatomy

**Version:** 0.1
**Date:** 2026-08-14
**Parent:** `../03-connectors.md`, `../13-contracts.md`
**Status:** Architecture. No implementation.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
>
> **RECONCILED WITH THE LLD.**
> Collectors are **modules inside one Go binary** (LLD §5, §6), reached
> through the `Connector` interface in LLD §10 and configured per LLD §11.
> Checkpoints live in SQLite (LLD §44).

---

## What this series is, and why it exists

The `../connectors/` series is a **catalog** — 121 connectors and 826 collectors enumerated, with what each pulls and produces. That answers *what exists*.

It never answers *how a collector works*. There is no collector equivalent of the engine series. This series is that.

```
  ../connectors/    WHAT — the catalog. One table row per collector.
  ./collectors/     HOW  — the mechanics, the spec format, and full
                           specifications for the ones being built.
```

| # | Doc | Covers |
|---|---|---|
| 00 | Anatomy (this) | Lifecycle, state, interface, failure taxonomy |
| 01 | [Specification format](01-specification-format.md) | What a complete collector spec contains |
| 02 | [Identity and IAM specs](02-spec-identity-and-iam.md) | The collectors that feed E6, E7 and E8 |
| 03 | [Deployment set specs](03-spec-deployment-set.md) | FortiGate, CrowdStrike, FortiEDR, Scalefusion, Agent |
| 04 | [Multi-source vendor playbooks](04-multi-source-vendor-playbooks.md) | Mixed-source intake, Fortinet, LDAP, cloud IAM, firewall streams |

---

## 1. What a collector is

```
  CONNECTOR   one authenticated integration with one source system
              one credential set · one rate-limit domain · one manifest

  COLLECTOR   one data-gathering routine inside a connector
              one API family or object type
              one cadence
              one coverage window
              one health state
              one class of entities or relationships
```

A collector is the **unit of scheduling, the unit of health, and — most importantly — the unit of retraction**. Coverage windows are emitted per collector per scope, so collector granularity *is* retraction granularity. Merge two collectors and a failure in one prevents safe tombstoning in the other.

---

## 2. Lifecycle of one run

```
  1  DISPATCH
     E15 assigns the collector for a scope. A worker claims a
     lease in Postgres — time-bounded, renewed while running.
     A dead worker's lease expires and the job is reclaimed.
     No orphans, no double execution.

  2  CREDENTIAL ACQUISITION
     The worker requests a scoped handle from the credential broker
     (separate process). It never receives the raw secret at rest;
     the value is injected into the HTTP client at call time and
     zeroed after. TTL 5 minutes, renewable.

  3  PREFLIGHT
     Non-mutating reachability and permission check.
     A collector that cannot see its scope is SKIPPED, not failed —
     and never retried into an account lockout.

  4  FETCH LOOP
     paginate → rate-govern → decode → emit → repeat
     Each page acquires tokens at every rate-limit level before
     issuing. A 429 backs off and halves the ceiling.

  5  EMIT
     Raw objects with a provenance envelope go to RECEIVE, which
     journals them. The collector does not parse, normalize or
     resolve — those are engines.

  6  CURSOR ADVANCE
     Only after the batch is durably downstream. Advancing early
     and crashing means silently skipped data.

  7  COVERAGE WINDOW
     Emitted ONLY if the enumeration completed. This is the single
     most consequential decision a collector makes.

  8  HEALTH REPORT
     Success criteria evaluated, metrics recorded, baselines
     compared. Silence is reported as loudly as failure.

  9  RELEASE
     Credential handle released, lease released, worker returned
     to the pool.
```

---

## 3. State a collector owns

Five pieces, all durable in Postgres, all per `(collector, scope)`:

```
  CURSOR           watermark for delta collection
                   advanced only after durable downstream write

  COVERAGE STATE   in-flight enumeration progress
                   objects seen, pages completed, started_at
                   → discarded on failure, promoted to a coverage
                     window on success

  RATE BUDGET      tokens consumed this period, per rate-limit level
                   → shared across collectors in the same domain

  CIRCUIT STATE    closed | half_open | open, with a reason
                   → auth failures open it and it does NOT self-close

  HEALTH BASELINE  expected object count, parse rate, field presence
                   → the reference for detecting silence and drift
```

The baseline is what makes the `SILENT` state possible (`../05-controller-ui.md §6.1`). A collector that normally returns 412 roles and today returned 0, with no error, is broken — and only a baseline can tell.

---

## 4. Pagination is the collector's real work

Most collector bugs live here.

```
  SCHEMES
    cursor / token      opaque continuation token       most common
    offset / limit      numeric page offset             drift risk
    link header         RFC 5988 next links             GitHub-style
    time window         start/end, chunked              logs, events
    marker              provider-specific key           AWS-style

  RULES
    1  A page failure mid-enumeration means NO coverage window.
       Emit what was collected, mark the run incomplete.
    2  Offset pagination over a mutating collection can skip or
       duplicate objects. Prefer cursor where offered; where not,
       accept duplicates (facts are idempotent) and never accept
       silent skips.
    3  Page size is a rate-limit trade, not a performance one.
       Larger pages mean fewer calls against a per-call quota.
    4  Never hold an entire enumeration in memory. Stream pages
       downstream; a 400-account IAM enumeration is not a buffer.
    5  A pagination loop must have a hard iteration ceiling.
       A provider returning the same cursor forever is a real
       failure mode.
```

---

## 5. The coverage window decision

The single most consequential thing a collector does, restated here because it is stated in E12, E13, the contracts and now here — and violating it in any one place produces the same outcome.

```
  EMIT a coverage window when, and only when:
    the enumeration was COMPLETE for a BOUNDED SCOPE
    every page succeeded
    no rate-limit truncation occurred
    the source returned a definitive end-of-collection

  {
    "collector": "aws.iam.roles",
    "scope": "account:123456789012",
    "started":   "2026-08-14T04:00:00Z",
    "completed": "2026-08-14T04:03:12Z",
    "enumeration_complete": true,
    "object_count": 412
  }

  DO NOT EMIT when:
    any page failed
    the run was throttled into truncation
    a filter or scope was narrower than the declared scope
    the source is a STREAM — a stream can never prove completeness
    the collector samples rather than enumerates
```

**Why it matters, concretely.** A connector breaks at 04:00 and nobody notices. Without the window, 8,400 edges go unobserved, get retracted, and the customer's exposure score *improves* while their exposure is unchanged. One bug, and the product's credibility with that customer is gone.

---

## 6. Failure taxonomy

Classification is the collector's job. Getting it wrong causes either account lockouts or silent data loss.

| Class | Trigger | Behaviour |
|---|---|---|
| **Transient** | network error, 5xx, timeout | Retry with exponential backoff and full jitter. Up to N attempts. Coverage window still possible if it ultimately completes. |
| **Rate limited** | 429, provider throttle signal | Back off, halve the bucket ceiling, recover 10%/min. **Not** a failure — the run finishes late. |
| **Authentication** | 401 | **Circuit break after 2 attempts.** Never retry-loop: that locks the customer's service account and turns a monitoring tool into an outage. |
| **Authorisation** | 403 | Circuit break. Surface the *exact* missing permission and the policy to fix it. |
| **Scope gone** | 404 on the scope itself | Mark the scope as removed, continue with others. This is legitimate — accounts get closed. |
| **Partial** | page failure mid-enumeration | Emit what was collected. **No coverage window.** Mark the subgraph stale. |
| **Poison object** | one object fails to decode | Quarantine that object with a sample, continue the enumeration. Never abort a run for one bad record. |
| **Schema drift** | fields missing or renamed | Emit with a flag. Field-presence monitoring catches it; the run is not a failure. |
| **Silent** | success, zero objects, no error | Compare against baseline. Below threshold → `SILENT` state, surfaced. |

### 6.1 The two that are most often got wrong

```
  AUTH FAILURE → RETRY LOOP
    The instinct is to retry a failed credential. Three retries
    across 42 account instances is 126 failed authentications
    against one directory, which locks the account. Circuit break
    on the SECOND failure, and do not self-close.

  PARTIAL → COVERAGE WINDOW ANYWAY
    The instinct is "we got 90% of it, that's basically complete."
    It is not. The missing 10% will be tombstoned. Completeness is
    binary.
```

---

## 7. Graceful degradation

A collector must handle missing optional permissions without failing the connector.

```
  requires_scope: ["iam:ListRoles", "iam:GetRole", "iam:GetRolePolicy"]
  optional_scope: ["iam:GetServiceLastAccessedDetails"]

  Missing a REQUIRED scope
    → collector does not run
    → surfaced with the exact permission and remediation
    → other collectors in the connector continue

  Missing an OPTIONAL scope
    → collector runs, that enrichment is absent
    → declared in health output so coverage is honest
    → degrades_gracefully: true in the manifest
```

The Controller must show this as a **capability gap**, not an error: *"iam.last_accessed unavailable — CIEM rightsizing disabled for this account."*

---

## 8. What a collector must NOT do

```
  ✕ parse, normalize or resolve
      those are E3, E4, E6. A collector emits raw objects with
      provenance and stops.

  ✕ read secret VALUES
      metadata, names, types and locations only. Reading a
      credential to check it makes us the risk we are reporting.

  ✕ mutate the source
      collect manifests are read-only. Response is a separate
      manifest with separate credentials, always.

  ✕ hold an entire enumeration in memory

  ✕ retry authentication failures

  ✕ emit a coverage window it did not earn

  ✕ decide severity or criticality
      that is E9 and the risk model
```

---

## 9. The interface, conceptually

Four functions. The framework supplies everything else — retry, backoff, pagination helpers, rate governance, cursor persistence, coverage emission, metrics, sandboxing.

```
  preflight(scope, credential) → ok | skip(reason)
      non-mutating reachability and permission check

  fetch(scope, cursor, credential) → pages of raw objects
      the only place provider SDK code lives

  key(rawObject) → canonical key candidates, ordered
      per the priority list in ../13-contracts.md Part IV

  map(rawObject) → observations
      declarative where possible; the bulk of a collector's
      definition
```

Everything a connector author writes lives in those four. If they are writing retry logic or token buckets, the framework has failed.

---

## 10. Cost, and why it must be measured

Manifests declare cost (`"~3 calls per role"`). Reality varies by an order of magnitude across accounts.

```
  MEASURE per run:
    api_calls_issued · bytes_received · objects_emitted
    wall_clock · rate_limit_waits · retries

  FEED BACK into:
    scheduling      an expensive collector runs less often under
                    budget pressure rather than failing
    the Controller  the cost sparkline that ranks collectors
                    against each other (05 §10.1)
    the customer    "22% of your AWS IAM quota, 41% of it from
                    cloudtrail"
```

A collector whose measured cost diverges sharply from its declared cost is a manifest bug, and it should be surfaced as one.

---

## 11. Idempotency

```
  Running a collector twice must be HARMLESS.

  Leases make double-execution rare, not impossible — a network
  partition can produce two live workers believing they hold the
  lease.

  Safety comes from downstream: observations are immutable, facts
  merge on semantic identity, and the graph upserts. So a duplicate
  run costs API calls and nothing else.

  A collector must NEVER rely on being run exactly once.
```

---

## 12. Testing a collector

The conformance suite every collector passes before it ships (`../03-connectors.md §4.4`), stated from the collector's side:

```
  [ ] SCHEMA        every emitted observation validates against
                    overlook.observation.v1
  [ ] CANONICAL KEY at least one key from the priority list, correctly
                    normalised, for every entity emitted
  [ ] IDEMPOTENCY   two runs against one fixture produce identical
                    observations
  [ ] PAGINATION    a 3-page fixture yields all records, no
                    duplicates, no skips
  [ ] RATE LIMIT    a 429 fixture triggers backoff, not failure
  [ ] AUTH FAILURE  a 401 fixture opens the circuit and does NOT
                    retry into a lockout
  [ ] PARTIAL       a mid-enumeration failure emits NO coverage window
  [ ] DEGRADATION   a missing optional scope skips cleanly and
                    declares the gap
  [ ] PRIVACY       no raw secret values in any emitted observation
  [ ] RESOURCE      a 10k-object fixture completes within the declared
                    budget, without buffering the whole set
  [ ] POISON        one malformed object is quarantined; the run
                    continues
  [ ] SILENCE       a zero-object fixture against a non-zero baseline
                    produces the SILENT state
```

### 12.1 Fixtures are recorded, not written

```
  Every collector records real API responses on first run against a
  real source, redacted, and commits them.

  WHY THIS MATTERS MORE THAN USUAL
    We cannot ask a customer to send us their data (../03 §11.5).
    Recorded fixtures are the ONLY way a support engineer reproduces
    a mapping bug, and the only way a collector is regression-tested
    after a provider changes its API.
```

---

## 13. The anatomy in one frame

```
       E15 ORCHESTRATION
              │ dispatch + lease
              ▼
       ┌─────────────────┐      ┌──────────────────┐
       │  COLLECTOR RUN  │◄────►│ CREDENTIAL BROKER│
       │                 │      │ scoped, TTL 5m   │
       │  preflight      │      └──────────────────┘
       │      ↓          │
       │  fetch loop ────┼────►  RATE GOVERNOR
       │      ↓          │       token buckets, 5 levels
       │  emit ──────────┼────►  RECEIVE → JOURNAL
       │      ↓          │
       │  cursor advance │       only after durable
       │      ↓          │
       │  coverage ──────┼────►  E12 GRAPH
       │      ↓          │       governs retraction
       │  health ────────┼────►  CONTROLLER
       └─────────────────┘       state, baselines, cost

       STATE, per (collector, scope), in Postgres:
         cursor · coverage progress · rate budget ·
         circuit state · health baseline
```

---

*Next: [Specification format](01-specification-format.md)*

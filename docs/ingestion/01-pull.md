# Pull Ingestion

**Series:** [Ingestion](00-index.md) · **Sources:** cloud APIs · IdP · repos · LDAP · databases · EDR · scanners

---

## 1. What it is

The appliance calls out and fetches. Complete, self-describing objects from an authenticated API, returned in full rather than as a stream of fragments.

This is the highest-value ingress class in the system — essentially all of IAM, cloud inventory, identity and posture arrives this way — and the one with the **lightest durability requirement**, because it is re-fetchable.

---

## 2. How a run works, step by step

```
  1  DISPATCH
     E15 assigns (collector, scope) to a worker.
     The worker claims a LEASE in Postgres — time-bounded, renewed
     while running. A dead worker's lease expires and the job is
     reclaimed. No orphans, no double execution.

  2  CREDENTIAL ACQUISITION
     The worker asks the credential broker (separate process) for
     a scoped handle. TTL 5 minutes, renewable.
     The raw secret is never held at rest by the worker — it is
     injected into the HTTP client at call time and zeroed after.

  3  PREFLIGHT
     A non-mutating reachability and permission check.
     Cannot see the scope → SKIP, not fail. Never retried into an
     account lockout.

  4  FETCH LOOP
     acquire rate tokens → issue call → decode → emit → repeat
     Each page acquires tokens at EVERY rate-limit level before the
     call is made (tenant → provider → account → service → op class).

  5  EMIT
     Raw objects plus a provenance envelope go to RECEIVE, which
     journals them. The collector does not parse, normalize or
     resolve.

  6  CURSOR ADVANCE
     Only after the batch is durably downstream.
     Advancing early and then crashing means silently skipped data —
     the worst outcome available to this class.

  7  COVERAGE WINDOW
     Emitted only if the enumeration completed. See §5.

  8  HEALTH
     Success criteria evaluated, metrics recorded, baseline compared.
     Silence reported as loudly as failure.

  9  RELEASE
     Credential handle released, lease released, worker returned.
```

---

## 3. Pagination — where most of the work is

Five schemes, each with a different failure mode.

```
  CURSOR / TOKEN          opaque continuation token
    most common, and safest. The provider guarantees consistency
    across pages.
    FAILURE: a provider that returns the same cursor forever.
    → MANDATORY hard iteration ceiling (max_pages).

  MARKER                  AWS-style, IsTruncated + NextMarker
    same shape as cursor. Completion signal is explicit, which
    makes it the easiest to emit a coverage window from.

  LINK HEADER             RFC 5988 next links
    GitHub, Microsoft Graph. The absence of a next link IS the
    completion signal.
    FAILURE: a proxy that strips headers. Detect and fail loudly
    rather than treating a stripped header as completion.

  OFFSET / LIMIT          numeric page offset
    DRIFT RISK. Over a mutating collection, objects inserted or
    deleted mid-enumeration cause skips and duplicates.
    → duplicates are harmless (facts are idempotent).
    → SKIPS ARE NOT. Where the provider offers nothing better,
      accept the risk, sort by a stable key, and never emit a
      coverage window from an offset enumeration over a
      high-churn collection.

  TIME WINDOW             start/end, chunked
    logs and events. Chunk small enough that one failed chunk is
    cheap to retry. Overlap chunk boundaries slightly and rely on
    idempotency to absorb the duplicates.
```

### 3.1 Rules that apply to all five

```
  1  NEVER buffer an entire enumeration in memory.
     Stream pages downstream. A 400-account IAM enumeration is not
     a buffer.

  2  A page failure mid-enumeration means NO coverage window.
     Emit what was collected, mark the run incomplete.

  3  Page size is a RATE-LIMIT trade, not a performance one.
     Larger pages mean fewer calls against a per-call quota.

  4  Hard iteration ceiling on every loop. Always.
```

---

## 4. Cursors

```
  WHAT A CURSOR IS
    a watermark per (collector, scope), durable in Postgres,
    describing where the last successful run reached

  WHEN IT ADVANCES
    after the batch is durably downstream — not after the fetch,
    not after the emit

  DELTA VS FULL
    delta run   uses the cursor · cheap · does NOT emit a coverage
                window, because it did not enumerate everything
    full run    ignores the cursor · expensive · DOES emit a
                coverage window

    Both are scheduled. Delta hourly or 4-hourly; full daily.
    Only the full run can drive retraction.

  WHEN A CURSOR IS INVALID
    provider says the token expired (Graph deltaLink, for example)
    → fall back to a full enumeration
    → do NOT attempt to resume from a stale token
```

---

## 5. Coverage windows

The one output of this class that carries real consequence.

```
  EMIT when ALL of:
    the enumeration was complete for a BOUNDED scope
    every page succeeded
    no rate-limit truncation occurred
    the provider returned a definitive completion signal

  {
    "collector": "aws.iam.roles",
    "scope": "account:123456789012",
    "started":   "2026-08-17T04:00:00Z",
    "completed": "2026-08-17T04:03:12Z",
    "enumeration_complete": true,
    "object_count": 412
  }

  DO NOT EMIT when any page failed, the run was throttled into
  truncation, the scope collected was narrower than declared, or
  the run was a delta.
```

---

## 6. Why no fsync

```
  PULL is the only class that does not require a durable write
  before proceeding, because it is RE-FETCHABLE.

  Crash mid-fetch → the cursor was never advanced → the next run
  refetches from the last good point. Cost: API calls. Loss: none.

  BUT IT IS STILL JOURNALED — without fsync — for one reason:
  REPLAY (07). A parser or mapping fix must be testable against
  what the provider actually returned, and we cannot ask the
  customer for it again in a form we can compare.
```

---

## 7. Failure handling

| Class | Trigger | Behaviour |
|---|---|---|
| Transient | 5xx, network error, timeout | Exponential backoff with full jitter. Coverage window still possible if it ultimately completes |
| Rate limited | 429 | Back off, halve the bucket ceiling, recover 10%/min. **Not a failure** — the run finishes late |
| Authentication | 401 | **Circuit break after 2 attempts.** A retry loop across 42 account instances is 126 failed auths and a locked service account |
| Authorisation | 403 | Circuit break. Surface the exact missing permission and the policy to fix it |
| Scope gone | 404 on the scope | Mark the scope removed, continue with others. Accounts do get closed |
| Partial | page failure mid-run | Emit what was collected. **No coverage window.** Mark stale |
| Poison object | one object fails to decode | Quarantine that object with a sample, continue. Never abort a run for one record |
| Silent | success, zero objects, no error | Compare against baseline → `SILENT` state |

---

## 8. Considerations

**The auth-failure retry loop is the single most dangerous behaviour in this class.** It is also the instinctive one. Two attempts, then circuit break, and the circuit does not self-close.

**Cursor advance ordering is not negotiable.** Every "we lost a day of data" incident in this class traces to a cursor advanced before the data was safe.

**Cost diverges from the manifest.** A manifest declares `~3 calls per role`; reality varies by an order of magnitude across accounts. Measure actual cost per run and feed it back into scheduling, or one large account repeatedly blows the budget.

**Idempotency is assumed, not enforced.** Leases make double execution rare, not impossible — a network partition can produce two workers believing they hold the lease. Safety comes from downstream: observations are immutable, facts merge on semantic identity. A duplicate run costs API calls and nothing else.

---

## 9. Example: Meridian, the AWS IAM delta

```
  09:00  E15 dispatches aws.iam.roles for 41 accounts.
         41 leases claimed across the worker pool.

  09:00  Each worker requests a credential handle. EDGE-CLD uses
         IRSA, so there is no stored secret — the handle is an
         STS session, TTL 5 minutes.

  09:00  Preflight: 40 accounts respond. Account 445566778899
         returns 403 on a non-mutating check.
         → SKIPPED. Circuit opened. NOT retried.
         → attention item with the graph consequence stated.

  09:01  Fetch loops begin. Marker pagination, page size 100.
         Governor: aws→account→iam→list buckets acquired per call.

  09:07  Account 12 receives a 429.
         → backoff with jitter, ceiling halved 30% → 15%
         → recovers 10% per successful minute
         → that account finishes 4 minutes late. Nothing fails.

  09:11  40 accounts complete. IsTruncated false on the final page
         of each.
         → 40 coverage windows emitted, scope account+region

  09:11  Account 445566778899: no window. Its 1,204 entities are
         marked STALE with the reason. NOTHING tombstoned.

  09:47  A worker crashes mid-account on a later collector.
         Its lease expires at 09:48; the job is reclaimed and
         refetched from the last good cursor position.
         Cost: 40 seconds of API calls.

  TOTALS
    ~180 MB fetched · journaled without fsync
    cursors advanced only after each batch reached E4
```

---

*Next: [Push ingestion](02-push.md)*

# E15 — Orchestration Engine

**Series:** [Engine documentation](00-index.md) · **v1:** required

---

## 1. Purpose

The Orchestration Engine decides **what collects, when, in what order, at what rate, and with which credential.** Nothing else in the appliance starts work on its own. Every connector run, every scheduled sweep and every credential request passes through it.

It is the only engine whose failure is silent by default: if orchestration stops scheduling, no error appears anywhere — collection simply stops, freshness decays, and the graph quietly goes stale. That property shapes most of its design.

---

## 2. Position

```
  INPUTS
    connector manifests            what exists, what it costs, its cadence
    connector configuration        which instances are enabled, at what scope
    cursors and coverage state     where the last run got to
    rate-limit state               how much quota is left
    health state                   what is broken and should be skipped
    the clock                      schedules, blackout windows, jitter

  OUTPUTS
    dispatched collection jobs     to the worker pool
    credential handles             requested from the vault on behalf of a job
    coverage windows               emitted when an enumeration completes
    health events                  to the Controller's attention inbox

  DEPENDS ON
    the credential broker (separate process)
    the worker pool
    Postgres for durable job and cursor state
```

---

## 3. Mechanics

### 3.1 The banded cycle

Ordering exists for one reason: **entity resolution needs identity authorities to land before anything that references them.** Everything else is parallel.

```
  BAND 0  PREFLIGHT
          non-mutating credential and reachability checks
          a failing credential is SKIPPED, never retried into a lockout

  BAND 1  IDENTITY AUTHORITIES
          AD, Entra, Okta, cloud org structure
          → canonical keys and the Resolution Directory update first

  BAND 2  PLATFORM INVENTORY AND GRANTS
          cloud IAM, resources, virtualisation
          → feeds permission closure

  BAND 3  WORKLOAD AND SUPPLY CHAIN
          Kubernetes (needs cloud IDs for the IRSA bridge)
          repos and CI (OIDC trusts need both sides present)

  BAND 4  DATA, AI, NETWORK
          isolated resource pool — must never starve bands 1-3

  BAND 5  OVERLAYS
          scanners, EDR, DLP, CMDB — attach properties to nodes that
          already exist, so they run last and always find their target
```

### 3.2 Quorum gating, not completion gating

A band opens when the previous band reaches **quorum**, not completion:

```
  open_next_band when:
      healthy_fraction(previous_band) >= 0.80
      OR elapsed(previous_band) > band_timeout

  Late arrivals still emit. Their facts merge normally on the next
  cycle. Nothing is lost — only delayed.
```

Completion gating would let one slow or broken source stall the entire cycle, which in a 42-account AWS estate is a near-certainty rather than an edge case.

### 3.3 Cadence

Each collector declares its own interval; orchestration jitters within it to avoid thundering herds against a shared API.

```
  CONTINUOUS   syslog, flow, webhooks, agent traffic — never banded
  15 min       cloud audit trails
  1 hour       identity deltas (Entra delta query, AD uSNChanged)
  4 hours      resource and IAM deltas, repo/CI state, agent AI scans
  12 hours     full IAM enumeration, network device configs
  24 hours     full resource enumeration WITH coverage windows
  7 days       deep classification, rolling and partitioned
```

### 3.4 Rate governance

Hierarchical token buckets, one per rate-limit domain:

```
  tenant → provider → account → service → operation class

  A job acquires tokens at EVERY level before issuing a call.
  Default ceiling: 30% of the source's published quota.
  The other 70% belongs to the customer, always.

  On throttle (429):
    exponential backoff with full jitter
    halve that bucket's ceiling
    recover 10% per successful minute
    5 throttles in 10 minutes → open circuit, mark degraded
```

### 3.5 Fair scheduling

Without fairness, one 400-account AWS org starves everything else.

```
  weighted fair queueing across connectors
  every connector holds a guaranteed minimum share of worker slots
  no connector may hold more than 40% of the pool
  leftover capacity distributed by priority class P0-P4
```

### 3.6 Job durability

Jobs live in Postgres, not memory. A crash mid-cycle must not lose the knowledge that account 17 of 42 was in progress.

```
  job record: connector, collector, scope, cursor, attempt,
              lease holder, lease expiry, state

  lease-based: a worker claims a job with a time-bounded lease
  and renews it. A dead worker's lease expires and the job is
  reclaimed. No orphaned jobs, no double execution.
```

---

## 4. Considerations

**Skew between declared and actual cost.** A manifest declares `~3 API calls per role`. Reality varies by an order of magnitude across accounts. Orchestration must measure actual cost per run and feed it back into scheduling, or a single large account will repeatedly blow the budget.

**Blackout windows are customer-specified and non-negotiable.** "Do not touch the domain controllers between 02:00 and 04:00" is a backup window. Violating it once destroys trust permanently.

**Initial load versus steady state.** The first sweep is a batch job that may run 14 hours. The steady state is an 8-minute loop. Orchestration must handle both without separate code paths — the difference is cursor state, not logic.

**Do not conflate scheduling with retrying.** A collector that fails because of a bad credential must not be rescheduled on its normal cadence; it must be circuit-broken. A collector that fails because of a transient network error should be retried with backoff. Classifying the error correctly is orchestration's job.

**Placement.** With two Edge Nodes, a job must run on the node that can reach the source. Placement comes from the connector instance configuration (`../08 §6.2`), and orchestration must refuse to dispatch a job to a node that cannot reach its target rather than letting it fail.

---

## 5. Failure modes

| Failure | Behaviour |
|---|---|
| Orchestration stops scheduling | **Silent.** Requires an external liveness check: the Controller alerts if no job has been dispatched in N minutes |
| A band never reaches quorum | Timeout opens the next band anyway; degraded state surfaced |
| Credential broker unreachable | All dispatch halts. Loud alarm. No fallback to cached secrets |
| Worker pool saturated | Jobs queue; the queue depth is a first-class metric |
| Clock skew / NTP failure | Schedules drift, coverage windows get wrong timestamps. Monitor clock drift explicitly |
| A job runs twice | Must be harmless — collection is idempotent by design, and leases make it rare rather than impossible |

---

## 6. Contracts

```
  IT MUST HONOUR
    manifest-declared cadence, cost and dependencies
    customer-configured blackout windows and quota ceilings
    local policy (a disabled connector is never dispatched)
    the 30%-of-quota default

  IT MUST GUARANTEE
    a coverage window is emitted ONLY on a complete enumeration
    no credential is ever held longer than its lease
    no source is called during its blackout window
    every dispatch is attributable in the audit log
```

---

## 7. Scope

```
  BUILD IN V1
    banded cycle with quorum gating
    per-collector cadence with jitter
    hierarchical rate governor
    lease-based durable jobs
    error classification (transient vs credential vs permanent)
    coverage window emission

  DEFER
    weighted fair queueing (matters at 30+ connectors, v1 has 6)
    cost feedback loop (measure first, adapt later)
    multi-node placement (v1 may be a single Edge Node)
```

---

## 8. Example: Meridian, 00:00 to 02:07

```
  00:00  BAND 0 — PREFLIGHT
         71 instances probed. 70 respond.
         aws/445566778899 returns 403 on a non-mutating check.
         → circuit opened. NOT dispatched. NOT retried.
         → attention item raised with the graph consequence.

  00:02  BAND 1 — IDENTITY AUTHORITIES
         EDGE-DC1: AD full sweep across 2 forests, 3 domains
                   24,000 objects, 720,000 ACEs, 2.8 GB
                   paged and rate-limited — Meridian's SOC has this
                   collection profile allowlisted
         EDGE-CLD: Entra delta query, AWS Organizations (42 accounts),
                   Azure management groups, GCP hierarchy

         AD takes 31 minutes. Entra finishes in 90 seconds and waits.
         At 00:33, 5 of 6 band-1 collectors are healthy = 83% quorum.
         → BAND 2 OPENS. AD is still finishing; that is fine.

  00:34  BAND 2 — PLATFORM
         41 AWS accounts dispatched (one is circuit-broken).
         Rate governor: aws→iam bucket ceiling is 30% of Meridian's
         quota. At 00:41 account 12 hits a 429.
         → backoff, ceiling halved to 15%, recovering 10%/min.
         → that account finishes 4 minutes late. Nothing fails.

         Azure RBAC across 18 subs, GCP across 6 projects, VMware.
         E7 closure begins consuming policy documents as they land.

  01:00  BAND 3 — WORKLOAD
         Kubernetes across 3 clusters — dispatched only now because
         the IRSA annotation bridge needs the AWS role IDs from
         band 2 to resolve.
         GitHub: 3 orgs. The oidc_trusts collector declares
         depends_on: [aws.iam], so it was deferred until now.

  01:09  BAND 4 — DATA / AI / NETWORK  (isolated pool)
         Oracle and MSSQL grants, file share ACLs,
         4 firewall rulebases (~16,000 rules),
         AI platform org APIs.
         The Oracle grant enumeration is slow and IO-heavy —
         it cannot borrow capacity from bands 1-3.

  01:31  BAND 5 — OVERLAYS
         CrowdStrike (8,500 hosts), Forcepoint DLP, vuln scanner.
         These attach properties to assets that bands 2-4 created.

  01:45  Cycle collection complete. Orchestration emits:
           38 coverage windows (complete enumerations)
            3 partial markers (the 403 account, plus 2 collectors
              that returned incomplete pages)
         → E12 may retract only within those 38 windows.
         → the 403 account's 1,204 entities are marked STALE.
           NOTHING is tombstoned.

  BLACKOUT RESPECTED
    Meridian declared 02:00-04:00 as the AD backup window.
    The nightly AD sweep is scheduled for 00:02 precisely so it
    completes before it. The hourly uSNChanged delta scheduled for
    02:00 and 03:00 is suppressed and resumes at 04:00.
```

**What orchestration prevented that night:** an account lockout (circuit break instead of retry loop), a quota exhaustion affecting Meridian's own automation (governor backoff), a stalled cycle (quorum gating past the slow AD sweep), a violated maintenance window, and 1,204 entities being wrongly deleted from the graph (no coverage window, no retraction).

---

*Next: [Receive, Journal, Aggregator](02-receive-journal-aggregator.md)*

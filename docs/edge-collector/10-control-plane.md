# 10 — The Control Plane

**Series:** [The Edge Collector](00-index.md) · **LLD:** §4.2, §8–11, §46–54, §63–69, §73–75

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. Divergences are
> recorded in [13-escalations.md](13-escalations.md).
> Budget: **0.25 vCPU · 1 GB RAM** (shared with the Agent Gateway).

---

## 1. Purpose

Everything a human does to the collector: configure it, watch it, diagnose it,
upgrade it.

Low volume, high consequence. The control plane's defining requirement is
**availability when the data plane is not** — an operator whose collector is
saturated needs the UI most at exactly the moment it is under most pressure.

---

## 2. Position

```
  INPUTS
    the operator, via the React UI on 8443 (LLD §61, §63)
    SaaS, for version notifications (LLD §74)

  OUTPUTS
    configuration → every data-plane module
    health and metrics → the operator and SaaS (LLD §49, §50)
    audit events → SQLite (LLD §54)

  COMPONENTS (LLD §4.2)
    Connector Manager · Configuration Manager · Credential Vault ·
    Certificate Manager · Health Manager · Parser Manager ·
    Update Manager
```

---

## 3. It must not share fate with the data plane

LLD §5's monolith means the API server and the parser workers are one process.
That is acceptable for V1 and it creates one obligation.

```
  THE CONTROL PLANE MUST BE RESOURCE-ISOLATED FROM THE DATA PLANE,
  EVEN INSIDE ONE PROCESS.

  · a dedicated goroutine pool for the API server, never drawn from
    the worker pools in LLD §17
  · its own bounded request queue — an API call must not wait
    behind 10,000 events
  · read paths that never take a lock the data plane holds. Health
    reads a snapshot, not live state.
  · a hard 0.25 vCPU floor reserved before worker scaling consumes
    the machine

  WITHOUT THIS: at LLD §37's RED, the UI becomes unresponsive at
  exactly the moment the operator needs it to explain why. That
  failure is self-reinforcing — the operator cannot see the cause,
  so they restart the collector, so STREAM data is lost (01 §4.3).
```

---

## 4. The local API

LLD §46 defines the surface. Three additions are proposed.

```
  /api/v1/health                    LLD §49
  /api/v1/status
  /api/v1/connectors                LLD §47
  /api/v1/connectors/{id}
  /api/v1/connectors/{id}/test      LLD §48
  /api/v1/connectors/{id}/health    LLD §52
  /api/v1/parsers
  /api/v1/agents                    LLD §55
  /api/v1/system
  /api/v1/metrics                   LLD §50

  PROPOSED
  /api/v1/detokenize                07 §6.2 — called by the
                                    analyst's BROWSER, not by SaaS
  /api/v1/coverage                  06 §6 — per-connector windows,
                                    so an operator can answer
                                    "what did we miss, and when?"
  /api/v1/deadletter                03 §7 — inspect and replay
                                    LLD §83 puts the UI for this in
                                    Phase 2; the API should exist
                                    in V1 so support can use it
```

### 4.1 Authentication and RBAC

LLD §75 requires RBAC and lists it among the minimum controls. Concretely:

```
  ROLE        CAN
  ─────────────────────────────────────────────────────────────
  viewer      read health, status, connector list, metrics
  operator    viewer + test connectors, restart connectors,
              enable/disable, inspect dead letter
  admin       operator + create/modify connectors, change
              credentials, rotate certificates, upgrade
  detokenize  a SEPARATE grant, not implied by admin — see below

  ⚠ DE-TOKENIZATION IS ITS OWN PERMISSION.
    Being able to administer the collector is not the same as being
    entitled to read the customer's identities and hostnames. In an
    MSSP, the person who configures a connector and the person
    authorized to see who a token represents are frequently
    different people, and the customer's auditor will ask.
    Every use is logged per operator per token (LLD §54).
```

---

## 5. Connector lifecycle

LLD §47 creates, §48 tests, §43 stores. The states between them matter and are
not enumerated in the LLD.

```
  DRAFT        created, not validated, not collecting
    │  test
    ▼
  VALIDATED    credentials work, permissions confirmed (LLD §48)
    │  enable
    ▼
  ACTIVE       collecting. LLD §51 HEALTHY.
    │
    ├─ transient failure ──► DEGRADED   still collecting, errors
    │                          │         rising. LLD §72.
    │                          ▼
    ├─ auth failure ────────► FAILED     not collecting.
    │                                    ⚠ COVERAGE WINDOW CLOSES
    │                                      (06 §6)
    ├─ disable ─────────────► DISABLED   deliberate. LLD §51.
    │                                    ⚠ coverage window closes,
    │                                      and this is EXPECTED —
    │                                      SaaS must distinguish
    │                                      deliberate from broken
    ▼
  DELETED      config removed, checkpoint retained 30 days
```

```
  THE DISTINCTION THAT MATTERS MOST

  FAILED    and DISABLED both stop collection. To SaaS they look
            identical unless we say otherwise.

  A DISABLED connector's relationships should be held STALE and
  clearly labelled "collection stopped deliberately".
  A FAILED connector's should be held STALE and ALARMED.

  Without the distinction, an operator who deliberately disables a
  noisy connector gets paged, and an operator whose connector broke
  does not.
```

### 5.1 The test endpoint is the most valuable thing here

```
  LLD §48 returns
    { status, authentication, latency_ms, permissions[] }

  THE permissions[] ARRAY IS WHY THIS MATTERS.

  Most connector failures are not "wrong password". They are
  "credentials work, but the role lacks iam:ListRoles", and that
  failure appears three hours later as a silently incomplete
  enumeration — which, per 06 §8, is the input that licenses
  RETRACTIONS.

  A read-only credential that can enumerate EC2 but not IAM will
  cheerfully report healthy and produce a graph missing every
  permission edge.

  → the test must enumerate what the credential CAN do and diff it
    against what the connector NEEDS, and report the gap explicitly:

      "authentication: success
       missing: iam:ListRoles, iam:GetRolePolicy
       impact: permission relationships will not be collected"
```

---

## 6. Health, and the two audiences it serves

```
  LLD §49  GET /api/v1/health
           { status, nats, state_db, saas, connectors{h/w/f} }
  LLD §51  HEALTHY · DEGRADED · FAILED · UNKNOWN · DISABLED
  LLD §52  per-connector: last_success, last_event, events_5m,
           errors_5m, latency_ms
  LLD §50  the metric names
```

```
  THIS IS PULL-BASED, FROM INSIDE. It cannot detect the failure in
  08 §6 — a forwarder that has silently stopped shipping while
  everything internal reports HEALTHY.

  → SaaS must alarm independently on "no batch from col-sg-01 in
    10 minutes". The two health signals answer different questions:

      the local API   "is this collector working?"
      the SaaS alarm  "is this collector still there?"

    Only the second survives the collector being wrong about itself.
```

### 6.1 What the dashboard should lead with

LLD §64 shows EPS, Queue GB, CPU, RAM, Disk. Two changes:

```
  QUEUE AS A PERCENTAGE, NOT GB.  "4.2 GB" means nothing without
  the limit. LLD §37's pressure levels are defined against
  percentages, so the operator cannot map the displayed number onto
  the documented behaviour.

  ADD THE DERIVED RETENTION WINDOW (ESC-2).
    "RAW retention: 11.6 hours at current 10,200 EPS"
  This is the number that decides whether an incident is
  recoverable, and LLD §35 already computes something like it for
  the outage view.
```

---

## 7. Configuration

LLD §69 gives the YAML shape. Three properties it needs:

```
  VALIDATED BEFORE APPLIED    a config that fails validation is
                              rejected with the reason, and the
                              running config is untouched. Never
                              partially applied.

  VERSIONED AND AUDITED       every change is an audit event
                              (LLD §54) with a before/after diff and
                              an operator. "Who changed the noise
                              policy?" must be answerable.

  HOT WHERE POSSIBLE          connector add/remove, parser bundles
                              (03 §5.1), noise rules, rate limits
                              and worker bounds should apply without
                              restart.
                              ⚠ IN A MONOLITH EVERY RESTART LOSES
                                STREAM DATA (01 §4.3). Restart-only
                                settings should be listed explicitly
                                in the UI as such, so an operator
                                knows the cost before clicking save.
```

---

## 8. Upgrades

LLD §74 is well specified — notification, download, verify signature, backup
config, install, restart, health check, rollback on failure. Three additions:

```
  DRAIN BEFORE RESTART        LLD §73's graceful shutdown already
                              does this. The upgrade path must use
                              it rather than SIGKILL.

  STAGGER ACROSS THE FLEET    Meridian has four collectors. Upgrading
                              them simultaneously means four
                              simultaneous STREAM gaps and four
                              coverage windows closing at once.
                              → one at a time, health-verified
                                between, and never during a
                                customer's change freeze.

  THE HEALTH CHECK MUST BE FUNCTIONAL, NOT LIVENESS.
                              "the process started" is not
                              "collection resumed". The check should
                              assert: connectors ACTIVE, parse rate
                              within 5% of pre-upgrade, a fact
                              emitted, a batch acknowledged by SaaS.
                              A process that starts and parses
                              nothing passes a liveness check and
                              fails the customer.
```

---

## 9. Considerations

**The UI is for the person installing and troubleshooting, not for the analyst.**
LLD §63's sections — Dashboard, Connectors, Agents, Parsing, Queue, System,
Security, Diagnostics — are exactly right and notably do *not* include findings,
paths or risk. Those live in SaaS. Keeping the collector UI operational is what
keeps it lightweight, and every request to add "just one security view" should be
declined.

**Secrets must be unreadable through every path, including logs.** LLD §53
requires automatic redaction. The test for it belongs in CI, running against real
log output, because the failure is a new field name nobody added to the redaction
list.

**Audit must be append-only and tamper-evident.** LLD §75 asks for
"tamper-resistant logs". On an appliance inside a customer's environment, where
an insider may be the subject of the investigation, this means hash-chaining
audit records and periodically anchoring the chain head to SaaS — a few bytes,
and it turns "the audit log says nothing" into a detectable state.

**Ports are a deployment negotiation, not a default.** LLD §61 lists 514, 6514,
8443, 9443. Each is a firewall change request in a regulated environment. The
list belongs in the pre-deployment questionnaire.

**Certificate rotation must be tested before the first customer.** LLD §74 and
§75 both mention it; §83 puts auto-rotation in Phase 2. Until then it is a manual
runbook, and a manual runbook that has never been executed is not a runbook.

---

## 10. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Control plane starved at RED | The UI is unresponsive exactly when it is needed | Reserved goroutine pool + 0.25 vCPU floor |
| Health read takes a data-plane lock | The status endpoint hangs under load | Read a snapshot |
| FAILED and DISABLED indistinguishable | Deliberate disables page; real failures do not | Distinct states carried to SaaS, §5 |
| Connector test checks auth only | A permission gap surfaces as a silent incomplete enumeration | Enumerate and diff permissions, §5.1 |
| Queue shown in GB | The operator cannot map it to LLD §37's levels | Percentage of limit |
| De-tokenize implied by admin | Anyone who can configure can read the estate | A separate grant, logged |
| Config partially applied | An inconsistent running state nobody can reproduce | Validate, then apply atomically |
| Restart-only settings not marked | An operator loses STREAM data by saving a setting | Label them in the UI |
| Fleet-wide simultaneous upgrade | Four coverage windows close at once | Stagger, health-verify between |
| Liveness-only upgrade health check | A collector that starts and collects nothing passes | Functional check, §8 |
| Audit not tamper-evident | An insider's changes are unprovable | Hash chain anchored to SaaS |
| Secrets in logs | Credential exposure through diagnostics | LLD §53 redaction, tested in CI |

---

## 11. Example: Meridian

### 11.1 The connector test that saved the graph

```
  Onboarding COL-mer-03. The AWS connector is created with a
  read-only role Meridian's cloud team provided.

  POST /api/v1/connectors/con-aws-prod/test

  {
    "status": "warning",
    "authentication": "success",
    "latency_ms": 88,
    "permissions": ["ec2:Describe*", "s3:List*", "rds:Describe*",
                    "cloudtrail:LookupEvents"],
    "missing":     ["iam:ListRoles", "iam:GetRolePolicy",
                    "iam:ListAttachedRolePolicies",
                    "iam:GetPolicyVersion"],
    "impact":      "IAM relationships will not be collected.
                    CAN_ASSUME edges and permission findings will
                    be absent."
  }

  WITHOUT THE missing/impact FIELDS
    status: healthy. Events flow. EC2 inventory populates. The
    dashboard is green.

    And every IAM relationship is silently absent — which means
    the flagship attack path (07 §9.2) does not exist, and nobody
    knows it does not exist. A missing path looks exactly like a
    secure environment.

  WITH THEM
    the gap is on screen at onboarding, before anyone trusts the
    graph. Meridian's cloud team added four permissions in an hour.
```

### 11.2 A staggered upgrade

```
  Collector 1.0.0 → 1.1.0 across Meridian's four collectors.

  02:00  COL-mer-04 first — the branch/OT collector, 2,000 EPS,
         lowest consequence if it goes wrong
         drain (LLD §73) → install → restart → functional check
         02:04  connectors ACTIVE · parse rate 99.94% (pre: 99.95%)
                · fact emitted · batch acknowledged  ✓
         STREAM gap: 47 seconds, counted, coverage shortened

  02:30  COL-mer-02
  03:00  COL-mer-01
  03:30  COL-mer-03 last — highest EPS, most connectors

  TOTAL STREAM LOSS   ~3 minutes across the estate, attributed,
                      each inside a declared coverage gap

  HAD ALL FOUR GONE AT 02:00
    four simultaneous coverage gaps, every connector at Meridian
    dark for ~50 seconds, and — without coverage windows (ESC-5) —
    a staleness sweep with nothing anywhere to contradict it.
```

---

*Next: [Response Plane](11-response-plane.md)*

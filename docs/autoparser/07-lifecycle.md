# Parser Lifecycle

**Series:** [Auto-parser](00-index.md)

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

A frozen parser is correct on the day it is frozen and decays from then on. Vendors change log formats in patch releases without notice, devices are upgraded, and applications are rewritten.

This document covers what happens after activation: **detecting drift, re-proposing, versioning, regression-testing, and sharing.**

The sharing part is where the economics change. A parser derived once is portable to every deployment with the same device — which turns per-customer effort into industry-wide effort paid once.

---

## 2. The states

```
  PROPOSED       derived, awaiting review
  ACTIVE         frozen, confirmed, executing in the ingest path
  DRIFTING       fallthrough or field-presence anomaly detected
  SUPERSEDED     a newer version is active; retained for replay
  RETIRED        source removed; retained for replay within journal
                 retention, then archived
```

---

## 3. Drift detection — three signals

A parser does not announce that it has stopped working. Three independent signals catch three different failures, and all three are needed.

```
  SIGNAL 1 — FALLTHROUGH RATE
    records matching no template
    baseline 2.8% → today 41%
    CATCHES: a format change, a new device version, a new event
             class
    → the loudest and most obvious signal

  SIGNAL 2 — TEMPLATE MATCH DISTRIBUTION
    template t_04 normally carries 41% of volume, today 3%
    while t_88 (previously rare) carries 38%
    CATCHES: a subtle format change where records still match
             SOMETHING, just the wrong thing
    → fallthrough stays flat. Only distribution shift reveals it.

  SIGNAL 3 — FIELD PRESENCE
    position 6 of t_04 populated in 99.8% of matches, today 2%
    CATCHES: the field was renamed, reordered or removed while the
             surrounding template still matches
    → BOTH other signals stay flat. This is the quietest failure
      and the one that silently stops producing edges.
```

**Signal 3 is the one that matters most and is checked least.** Parse rate at 100%, fallthrough at baseline, and the graph quietly stops receiving identity attribution from that source.

---

## 4. Re-proposal

```
  DRIFT DETECTED
    → parser marked DRIFTING, but LEFT ACTIVE
      (a partially-working parser beats none, and its output is
       flagged reduced-confidence rather than discarded)
    → attention item raised with the signal, the magnitude and
      the graph consequence
    → a re-proposal run is offered, scoped to the drift window

  RE-PROPOSAL
    samples the journal from the drift onset
    runs L0-L4 as before
    DIFFS AGAINST THE ACTIVE PARSER rather than proposing from
    scratch:

      unchanged templates      118   ← carried forward untouched
      changed templates          3   ← field positions moved
      new templates             14   ← the new event class
      obsolete templates         2   ← no longer observed

    → the operator reviews 19 items, not 141.
```

**Diffing against the active parser is what makes re-proposal tractable.** A full re-derivation would present 141 templates again and the operator would approve them blind.

---

## 5. Versioning and replay

```
  SEMANTIC VERSIONING
    PATCH   a template's field position corrected. Same semantics.
    MINOR   templates added. Existing behaviour unchanged.
    MAJOR   a mapping's PREDICATE or SUBJECT/OBJECT changed —
            the meaning of edges this parser produces has changed.

  A MAJOR BUMP IS A GRAPH EVENT
    edges produced by the previous version carry the old semantics.
    They are not automatically re-derived; they age out or are
    replaced as new observations arrive.
    → the version is recorded on every observation, so a wrong
      mapping can be traced to exactly the edges it produced and
      retracted in bulk.

  REPLAY COMPATIBILITY
    every parser version is retained for as long as the journal it
    can replay. Replaying a window from before an upgrade uses the
    parser version that was ACTIVE THEN, unless the operator
    explicitly pins a newer one to test a fix (../ingestion/07).
```

---

## 6. Regression testing

```
  EVERY PARSER CARRIES ITS FIXTURES

    fixtures/
      linux.rsyslog.meridian/
        t_04_publickey.sample
        t_11_failed_password.sample
        t_29_sudo.sample
        fallthrough_unmatched.sample
        expected_observations.json

  Captured at freeze time from the derivation sample, REDACTED —
  values replaced with structurally equivalent synthetic ones.

  ON EVERY VERSION CHANGE
    · all fixtures must still produce their expected observations
    · a changed expectation must be explicit in the diff
    · a fixture that no longer matches is a regression, not an
      update

  This is what makes a shared parser (§7) trustworthy: it arrives
  with proof of what it does.
```

**Redaction at fixture capture is mandatory**, because fixtures travel. A fixture containing a real username is customer data leaving the environment through the back door.

---

## 7. Sharing — where the economics change

```
  A PARSER ARTIFACT CONTAINS
    ✓ template patterns
    ✓ field positions and types
    ✓ predicate mappings and mechanisms
    ✓ confidence values and rationale
    ✓ redacted fixtures

    ✕ NO customer data
    ✕ NO sample values
    ✕ NO hostnames, usernames, addresses

  → IT IS PORTABLE.

  THE ECONOMICS
    one operator, at one customer, spends six minutes confirming
    a parser for a legacy Juniper SRX.
    Every other deployment with a Juniper SRX gets it for free.

    Per-customer effort → industry-wide effort, paid once.
```

### 7.1 The sharing model

```
  PRIVATE       stays on the collector. Default for anything
                derived from a customer-built application, because
                the TEMPLATE PATTERNS THEMSELVES may reveal
                internal system design.

  DEPLOYMENT    shared across one customer's Edge Collectors.
                Automatic — same customer, same trust boundary.

  CONTRIBUTED   offered back to Overlook, reviewed, and shipped as
                content to all customers.
                → OPT-IN, per parser, with a preview of exactly
                  what would be sent
                → reviewed by us before publication, not
                  auto-merged
                → attributed to the contributing customer if they
                  want it, anonymous if they do not
```

**Chronicle does this with community parsers in GitHub, and it works.** The difference in our case is that the artifact is derived rather than hand-written, so the contribution costs an operator six minutes rather than a day.

### 7.2 What must not be shared by default

```
  ⚠ TEMPLATE PATTERNS FROM CUSTOMER-BUILT APPLICATIONS

  "Processing settlement batch <*> for counterparty <*> in
   region <*>"

  contains no values, and still tells you that this organisation
  processes settlement batches by counterparty and region. For a
  bank, that is architecture disclosure.

  → parsers for sources classified as internal/custom default to
    PRIVATE and require explicit per-parser opt-in to leave, with
    the patterns shown before consent.
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Drift undetected | Source silently stops producing edges | Three independent signals, especially field presence |
| Drifting parser deactivated | Total loss instead of partial | Leave active, flag reduced confidence |
| Full re-derivation on drift | 141 templates re-approved blind | Diff against the active parser |
| Major mapping change unversioned | Edges with two meanings, indistinguishable | Version recorded per observation; bulk retraction possible |
| Fixtures contain real values | Customer data leaves via a shared parser | Redaction mandatory at capture |
| Custom-app patterns shared | Architecture disclosure | Private by default, explicit opt-in with preview |
| Old parser versions discarded | Replay of historical windows impossible | Retain for journal-retention duration |
| Contributed parser auto-merged | An untested or malicious mapping ships to everyone | Review before publication |

---

## 9. Considerations

**Drift is normal, not exceptional.** A parser is a hypothesis about a format that someone else controls and will change without telling you. Design for continuous re-proposal, not for a one-time onboarding event.

**A drifting parser should degrade, not stop.** Its output continues at reduced confidence, its state is visible, and its records are quarantined only where they no longer match. Deactivating on drift converts a partial gap into a total one.

**Sharing is the whole argument for auto-parsing at MSSP scale.** Deriving a parser per customer per device is linear cost. Deriving once and sharing is amortised across the customer base, and it is the mechanism that makes a 121-connector catalog reachable without a large engineering team.

**The contribution review is not optional.** A parser is executable configuration that produces graph edges. An auto-merged community contribution could map a benign log line to an ADMIN capability edge at every customer simultaneously. Review before publication, always.

---

## 10. Example: Meridian, the FortiOS upgrade, end to end

```
  DAY 0, 03:40   fw-branch-02 upgraded FortiOS 7.4 → 7.6

  DRIFT DETECTION
    03:41  fallthrough rate       2.1% → 61%     SIGNAL 1
    03:41  field presence: service field 99.1% → 0.2%   SIGNAL 3
    03:44  parser fortigate.fortios-7.4 v1.2.0 → DRIFTING
           LEFT ACTIVE. Output flagged reduced-confidence.

  06:12  Controller attention:
           ● PARSER DRIFT · fortigate.fortios-7.4
             fallthrough 61% · service field lost
             likely a format change after the overnight upgrade
             graph consequence: reachability edges from this
             device are STALE. Nothing tombstoned.
             [ Re-propose ]  [ View samples ]

  09:14  operator clicks RE-PROPOSE
           sample: drift window, 6 hours, 240,000 records
           L0-L4 run, 90 seconds

  09:16  DIFF AGAINST ACTIVE PARSER
           unchanged templates      34
           changed templates         3   ← the service field moved
                                           two positions and was
                                           renamed
           new templates             2   ← a new event class in 7.6
           obsolete templates        1

           → 6 items to review, not 40

  09:22  operator confirms all 6, adjusting one mapping where the
         new field name changed the type verdict

  09:23  SIMULATE
           matched 99.2% on the drift window
           held-out (last 2h) 99.0%
           fixtures: 11 of 12 pass
             ✕ t_18 expected observation changed — the operator
               reviews and accepts, because the underlying field
               genuinely moved
           → expectation updated explicitly, recorded in the diff

  09:25  FREEZE  fortigate.fortios-7.6 v2.0.0
                 MAJOR — one mapping's object position changed
         ACTIVATE
         previous version SUPERSEDED, retained for replay

  09:26  REPLAY the drift window with v2.0.0 (../ingestion/07)
           41,000 quarantined records reprocessed
           +340 aggregates · +12 reachability edges
           the 6-hour gap closes

  DAY 1   CONTRIBUTION
    the operator is offered: "this parser covers FortiOS 7.6.
    Contribute it?" with the full artifact shown — patterns,
    mappings, redacted fixtures, and confirmation that no
    customer data is included.

    Meridian opts in, anonymously.

    → reviewed by Overlook, published as content v2026.08.19
    → every other customer running FortiOS 7.6 receives a working
      parser BEFORE their own upgrade breaks anything

  TOTAL OVERLOOK ENGINEERING EFFORT
    a content review.
```

**That last line is the point of the entire series.** In the model we rejected — connectors built per request on a release train — the same event costs an engineer a day, a release cycle, and a customer waiting three weeks with a broken source.

---

*End of the auto-parser series. Back to the [index](00-index.md).*

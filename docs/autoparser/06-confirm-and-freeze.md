# Confirm and Freeze

**Series:** [Auto-parser](00-index.md)

---

## 1. Purpose

Turn a set of proposals into a **deterministic parser artifact**, with a human in the loop, and take every inference component out of the runtime path.

This is the step most auto-parser designs skip, and skipping it is why they cannot be trusted with a security graph. An auto-parser that maps at ingest time is a system whose behaviour changes without anyone approving the change.

---

## 2. The commitment

```
  THE AUTO-PARSER PROPOSES.
  A HUMAN CONFIRMS.
  THE CONFIRMED RESULT IS FROZEN INTO A DETERMINISTIC PARSER.

  NO INFERENCE RUNS AT INGEST TIME. EVER.
```

Everything in `01` through `05` is a **learning phase**, run offline in the scanner pool against a journal sample. Its output is an artifact. After that, ingestion executes the artifact and nothing else.

---

## 3. The workflow

```
  1  SAMPLE
     24-72 hours from the journal for the target source.
     Bounded, scoped, local.

  2  ANALYSE
     L0 detect → L1 extract → L2 mine → L3 type → L4 map
     scanner pool. Minutes, not hours.

  3  PROPOSE
     a reviewable artifact per template:
       pattern · typed positions · proposed mapping · evidence ·
       corroboration · confidence · sample matches

  4  REVIEW
     the operator confirms, adjusts or rejects each proposal
     above the display threshold, and can inspect the ones below

  5  SIMULATE
     run the candidate parser against the SAMPLE and show what
     it would produce: observations, entities, edges, and — just
     as importantly — what it would discard

  6  FREEZE
     emit a versioned parser artifact. Deterministic. Readable.

  7  ACTIVATE
     the parser enters the ingest path. From here it is executed,
     never inferred.
```

---

## 4. What "frozen" means

```yaml
parser:
  id: linux.rsyslog.meridian
  version: 1.0.0
  source_binding: [linux/rsyslog]
  created: 2026-08-17T09:41:00Z
  confirmed_by: shiva@meridian.com
  derived_from:
    sample_window: 48h
    sample_records: 4_100_000
    method: [detect_v2, drain_v1, type_v3, map_llm_local_v1]
    model: null            # or the model id, if one was used

  envelope:
    format: rfc3164
    fields: {timestamp: 1, host: 4, tag: 5, message: rest}

  templates:
    - id: t_04
      pattern: "Accepted publickey for <*> from <*> port <*> ssh2"
      match: exact_after_masking
      emits:
        observation_type: RELATIONSHIP
        subject:   {from: position_6, key_scheme: "sam:{source.domain}\\{value}"}
        predicate: AUTHENTICATES_TO
        object:    {from: source_context.host, key_scheme: "fqdn:{value}"}
        attributes: {mechanism: ssh_publickey}
        confidence: 0.64
      confirmed_by: shiva@meridian.com
      confirmed_at: 2026-08-17T09:41:00Z

    - id: t_11
      pattern: "Failed password for invalid user <*> from <*> port <*>"
      match: exact_after_masking
      emits:
        observation_type: EVENT_SUMMARY
        event_type: authentication_failure
      no_edge_reason: "outcome is failure — an attempt is not a capability"

    - id: t_88
      pattern: "Connection closed by authenticating user <*> <*> port <*>"
      emits: null
      no_edge_reason: "outcome indeterminate; model proposal vetoed
                       by the outcome rule and confirmed by operator"

    # ... 138 more, most with emits: null

  fallthrough:
    action: quarantine
    reason: "a record matching no template is a coverage gap, not
             a discard"
```

### 4.1 Properties this gives us

```
  DETERMINISTIC   the same input produces the same output, always.
                  Journal replay (../ingestion/07) works.

  READABLE        an operator, an auditor or a support engineer can
                  read exactly what this parser does without running
                  it.

  DIFFABLE        version 1.0.0 against 1.1.0 is a text diff.
                  Review is possible.

  PORTABLE        the artifact can be exported, shared, imported
                  elsewhere (07). It carries no customer data —
                  patterns and mappings only.

  ATTRIBUTED      every template records who confirmed it and when.

  AUDITABLE       "why did this log line become an edge?" has an
                  answer that is a file, not a model.
```

---

## 5. The review surface

```
  ┌──────────────────────────────────────────────────────────────┐
  │ PARSER PROPOSAL · linux.rsyslog @ 10.4.2.40                  │
  │ sample 48h · 4.1M records · 141 templates promoted            │
  │                                                              │
  │ ASSERTS RELATIONSHIPS                                    3   │
  │ EXPLICIT NO-EDGE (outcome rules)                        12   │
  │ ASSERTS NOTHING                                        126   │
  │ BELOW CONFIDENCE THRESHOLD                               2   │
  │                                                              │
  │ ─── needs your decision ────────────────────────────────────  │
  │                                                              │
  │ ✓ t_04  Accepted publickey for <*> from <*> port <*> ssh2    │
  │     → IDENTITY AUTHENTICATES_TO ASSET · ssh_publickey        │
  │     confidence 0.64 · 12,408 obs · corroborated by AD        │
  │     [ show 5 sample matches ]                                │
  │     [ Confirm ]  [ Adjust ]  [ Reject ]                      │
  │                                                              │
  │ ? t_29  sudo: <*> : ... USER=<*> ; COMMAND=<*>               │
  │     → IDENTITY CAN_ASSUME IDENTITY · sudo                     │
  │     confidence 0.58 BELOW THRESHOLD                          │
  │     ⚠ only source of sudo-derived edges at this deployment   │
  │     model rationale: [ show ]                                │
  │     [ Confirm ]  [ Adjust ]  [ Reject ]                      │
  │                                                              │
  │ ─── review if you want ─────────────────────────────────────  │
  │ 126 templates assert nothing         [ Review ]              │
  │  12 vetoed by outcome rules          [ Review ]              │
  │                                                              │
  │              [ Simulate ]        [ Freeze and activate ]     │
  └──────────────────────────────────────────────────────────────┘
```

**"126 templates assert nothing" is a button, not a footnote.** The most likely error in this whole layer is a missed mapping, and the only way it gets found is if the operator can look at what was skipped.

---

## 6. Simulation before activation

```
  Runs the candidate parser against the SAME sample it was derived
  from, plus a held-out window, and reports what it would produce.

  SIMULATION · linux.rsyslog.meridian v1.0.0
    sample 48h · 4.1M records

    matched a template            97.2%   (3,985,200)
    fell through to quarantine     2.8%     (114,800)

    OBSERVATIONS PRODUCED
      RELATIONSHIP     8,441   (after dedup within the window)
      EVENT_SUMMARY      612
      PROPERTY             0

    ENTITIES REFERENCED
      IDENTITY   341   of which 284 resolve against the entity
                       store (83%)
                       ⚠ 57 do not resolve — mostly service and
                         local accounts. Expected.
      ASSET      112   of which 112 resolve (100%)

    EDGES THAT WOULD BE CREATED
      AUTHENTICATES_TO   8,398 → collapses to 1,204 distinct
      CAN_ASSUME             43 → collapses to 11 distinct

    ⚠ HELD-OUT WINDOW (24h not used for derivation)
      matched 96.9% — consistent. No overfitting.

    [ Freeze and activate ]   [ Back to review ]
```

The held-out window matters. A parser derived from and validated against the same sample will always look perfect.

---

## 7. Fallthrough is quarantine, never discard

```
  A record matching NO template is not garbage. It is either:
    · a rare event type the sample did not contain
    · a format variant from a device version we did not see
    · genuine drift since the parser was frozen

  ALL THREE ARE COVERAGE GAPS AND MUST BE VISIBLE.

  → quarantine with a retained sample, grouped by shape
  → fallthrough rate is a monitored metric alongside parse rate
  → a rising fallthrough rate triggers re-proposal (07)
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Inference at ingest time | Non-deterministic, unauditable, unaffordable | Freeze to an artifact; no model in the path |
| Auto-activation without review | Parser behaviour nobody approved | Human confirmation required before freeze |
| Validated on the derivation sample only | Overfitted parser that fails on new data | Held-out window in simulation |
| No-edge templates hidden | Missed mappings never found | "Asserts nothing" is reviewable |
| Fallthrough discarded | Silent coverage gap | Quarantine, monitored rate |
| Artifact not readable | "Why did this map?" unanswerable | YAML, diffable, with rationale per template |
| Confirmation not attributed | No accountability for a wrong mapping | Who and when, per template |

---

## 9. Considerations

**Review effort must be proportionate to what matters.** 141 templates is not 141 decisions. Three assert relationships, twelve were vetoed by rules, 126 assert nothing. The operator decides on three, and can inspect the rest. Presenting 141 equal-weight items would guarantee they are all approved without reading.

**Freezing is what makes the whole approach defensible.** Every argument against auto-parsing in a security product — non-determinism, unauditability, silent drift, cost — is an argument against inference at ingest. None of them applies to a reviewed, versioned, deterministic artifact that happens to have been *derived* automatically.

**The artifact is more valuable than the process that produced it.** A parser for a Juniper SRX, confirmed by one operator, is portable to every other deployment with a Juniper SRX (`07`). The derivation cost is paid once, industry-wide, if we let it be shared.

**Adjust must be as easy as confirm.** The common case is a proposal that is 90% right — correct predicate, wrong position, or a missing mechanism. If adjusting means writing a grammar by hand, operators will confirm bad proposals instead.

---

## 10. Example: Meridian, freezing the Juniper

```
  THE SOURCE  10.4.9.22:514 — the unidentified device from 01 §10

  DAY 0
    detected as RFC3164 envelope, body unclassifiable
    4,200 records/hour quarantined
    Controller: "unidentified source"

  DAY 0, 14:20
    operator recognises the format as a legacy Juniper SRX and
    binds it manually.
    → 14 hours of quarantined records replayed from the journal

  DAY 1, 09:00 — PROPOSAL RUN
    sample: 48h, 201,600 records
    L2 mining: 74 templates, 61 promoted
    L3 typing: cross-source overlap against AD and the asset store
    L4 mapping: local model, 61 inferences, 40 seconds

  PROPOSAL
    asserts relationships          2
      "user <*> logged in from <*>"       → AUTHENTICATES_TO  0.61
      "<*> changed configuration <*>"     → CAN_MODIFY        0.66
    explicit no-edge (outcome rules)  9
    asserts nothing                  50

  REVIEW — 6 minutes
    operator confirms both
    adjusts the second: the model proposed the object as the
    device itself; the operator scopes it to the device's
    configuration, which is the correct target
    → adjustment recorded, confidence raised to 0.78 by
      operator authority

  SIMULATE
    matched 94.1% · held-out 93.8% · consistent
    would create 41 AUTHENTICATES_TO and 12 CAN_MODIFY edges
    ⚠ 5.9% fallthrough — a rare event class the sample missed
      → operator accepts; the fallthrough is quarantined and
        monitored

  FREEZE
    juniper.srx.legacy v1.0.0
    confirmed_by shiva@meridian.com
    61 templates · 2 emitting · 59 no-op
    ACTIVATED

  RESULT
    a device nobody had written a connector for is now producing
    graph edges, and the device's admin accounts — which the
    CAN_MODIFY mapping surfaces — turn out to be four local
    accounts not present in any directory.

    That is exactly the finding class from connectors/05: network
    device admin accounts are among the highest-privilege,
    least-governed identities in an enterprise, and this one had
    been invisible.

  TOTAL ENGINEERING EFFORT AT OVERLOOK: ZERO.
```

---

*Next: [Parser lifecycle](07-lifecycle.md)*

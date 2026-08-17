# Journal Replay

**Series:** [Ingestion](00-index.md)

---

## 1. Why this exists

Every other product debugs a parsing bug by asking the customer for a sample.

**We cannot.** The privacy architecture forbids it — not as policy, but structurally: the whole premise is that customer data does not leave their environment. A support engineer who receives a log file has broken the thing the customer bought.

So the debugging loop has to run entirely on the customer's premises, operated by the customer, with an artifact they can act on. Journal replay is that loop.

```
  content update ships
    → operator replays the journal locally, scoped
    → diffs the new output against what was originally produced
    → applies if the diff is what was expected

  No data leaves. No support ticket. No screenshot of a log line
  pasted into a chat window.
```

This is also why PULL sources are journaled at all despite being re-fetchable — re-fetching gives *today's* data, not the data that produced yesterday's wrong answer.

---

## 2. How it works, step by step

```
  1  SCOPE
     source class · time window · optionally a specific instance
       source:  fortigate.syslog
       window:  2026-08-16T03:00Z → 2026-08-17T03:00Z
       instance: fw-branch-02

  2  SELECT SEGMENTS
     the sparse offset index locates the byte range. Only the
     segments overlapping the window are read.

  3  PIN CONTENT VERSION
     which parser grammars, normalization mappings, escalation
     primitives and posture rules to run with.
       original: v2026.08.09
       proposed: v2026.08.15

  4  RUN IN AN ISOLATED CONTEXT
     a shadow pipeline, in the SCANNER pool, writing to a
     temporary output store. It does NOT touch the entity store,
     the graph, the fact store or the outbound queue.

  5  DIFF
     new output against the ORIGINAL output for those same records,
     retained alongside the journal.

  6  PRESENT
     what parses now that did not · what changed · what broke ·
     what entities and edges would result

  7  APPLY, or DISCARD
     apply merges the reprocessed output into the real pipeline
     from the resolve stage onward.
```

---

## 3. Dry run versus apply

```
  DRY RUN     default. Always. Writes nothing outside the temporary
              store. Costs CPU and time, nothing else.

  APPLY       merges reprocessed observations into the live
              pipeline. Requires:
                · a completed dry run
                · explicit operator confirmation
                · a reason, recorded in the audit log

  ⚠ APPLY IS NOT IDEMPOTENT WITH RESPECT TO COVERAGE.
    Reprocessing a window does NOT re-emit coverage windows.
    Replay can add and correct observations; it can never
    authorise a retraction, because the original enumeration
    completeness is a property of the original run.
```

---

## 4. The diff

The output an operator actually reads.

```
  REPLAY DIFF · fortigate.syslog · fw-branch-02
  window 2026-08-16 03:00 → 2026-08-17 03:00
  content v2026.08.09 → v2026.08.15

  RECORDS
    total in window            41,204
    parsed before                1,648   (4.0%)
    parsed after                40,981   (99.5%)
    newly parsing               39,333
    still failing                  223   → 1 remaining signature

  FIELD PRESENCE
    service          0.2% → 99.1%    ← the renamed field, resolved
    app             98.9% → 98.9%
    srcintf         99.7% → 99.7%

  DOWNSTREAM EFFECT (simulated)
    aggregate records         +340
    CONNECTS_TO edges          +12 new · 3 changed
    entities                    +0
    findings                    +0

  REGRESSIONS
    none

  [ Apply ]   [ Discard ]   [ Export diff ]
```

**"Regressions: none" is the line that matters.** A content update that fixes one grammar and quietly breaks another is the reason automatic rollback exists (`01 §17`), and the reason a dry run is mandatory before apply.

---

## 5. What replay cannot do

```
  ✕ CANNOT re-emit coverage windows
      completeness is a property of the original enumeration.
      Replay corrects what was observed; it cannot assert what
      was not.

  ✕ CANNOT recover data that was never journaled
      shed records, UDP loss, and anything dropped before receive
      are gone. Replay is not a time machine.

  ✕ CANNOT reprocess beyond journal retention
      processed + 24h grace, or the size cap. A bug found three
      weeks later has no journal to replay.

  ✕ CANNOT run against a different source
      scoped by source class. Cross-source correlation bugs need
      the graph, not the journal.
```

---

## 6. What it is used for, in practice

```
  1  PARSER FIXES                         the primary case
     a vendor changed a log format; a grammar update ships;
     verify locally before applying

  2  MAPPING CORRECTIONS
     a field was mapped to the wrong Overlook schema field. Replay
     shows exactly which entities change.

  3  CONTENT VALIDATION BEFORE ROLLOUT
     a new escalation primitive or posture rule ships. Replay
     against a week of real data answers "how many times would
     this have fired?" before it fires for real.
     → this is the cardinality check that stops a miscalibrated
       rule emitting 4,100 findings (../analytics/08 §3.1)

  4  INCIDENT RECONSTRUCTION
     "what did this source actually report between 03:00 and
     04:00?" — raw, local, with the original bytes.

  5  SUPPORT WITHOUT DATA TRANSFER
     the operator runs the replay, exports the DIFF — counts,
     field-presence deltas, failure signatures, structurally
     redacted samples — and sends that. No customer data moves.
```

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Replay touches live stores | Corruption from a diagnostic | Isolated context, temporary store, dry run by default |
| Replay in the realtime pool | Resolve API latency blown during a diagnostic | Scanner pool, hard-capped |
| Coverage windows re-emitted | Wrongful retraction from a replay | Structurally prohibited |
| Window outside retention | Nothing to replay | Retention surfaced with the request; 24h grace minimum |
| Apply without dry run | An untested content change merged | Dry run mandatory, enforced |
| Diff omits regressions | A fix that breaks something else looks clean | Diff reports both directions, always |
| Original output not retained | Nothing to diff against | Original observations retained alongside the journal for the same period |

---

## 8. Considerations

**Replay must be operable by the customer, not by us.** That is the entire point. It belongs in the Controller with named scopes and a readable diff, not behind a CLI flag that a support engineer talks someone through over a call.

**Retaining the original output doubles the cost of the journal.** Diffing requires knowing what was produced the first time. That is a real storage cost and it is worth paying — a diff against nothing is just "here is some new output," which an operator cannot evaluate.

**Content validation before rollout is the underrated use.** Case 3 in §6 is not debugging at all; it is a safety property. Answering *"how many times would this rule have fired last week?"* against real data, before shipping it, is how a rule set stays calibrated.

**Replay is bounded by journal retention, and retention is a slider.** A customer who sets journal retention low to save disk loses the ability to debug their own parsing failures. The Controller should say so at the point of the decision.

---

## 9. Example: Meridian, the FortiOS upgrade

```
  DAY 0

  03:40  Meridian's network team upgrades fw-branch-02 from
         FortiOS 7.4 to 7.6 during a change window.
  03:41  parse rate for that source drops 98.9% → 4%.
         Records QUARANTINED, not dropped. Samples retained.
  03:44  sustained drop crosses threshold. Source marked DEGRADED,
         re-fingerprinting attempted, confidence below threshold.

  06:12  Controller attention inbox:
           ● PARSE RATE COLLAPSE · fw-branch-02
             41,000 records quarantined · 3 failure signatures
             Graph consequence: reachability edges from this device
             are STALE. Nothing tombstoned — no coverage window.

  06:20  operator clicks "Report to Overlook."
         → a REDACTED bundle is generated: failure signatures,
           field-shape summaries, structurally anonymised samples.
         → no customer data leaves.

  DAY 1

  Overlook ships content v2026.08.15 with a FortiOS 7.6 grammar.
  Staged: canary → 10% → 100%.

  09:14  the Meridian operator opens Controller → Diagnostics
         → REPLAY
             source     fortigate.syslog
             instance   fw-branch-02
             window     last 48 hours
             content    v2026.08.15
             mode       DRY RUN

  09:16  the replay runs in the scanner pool. 41,204 records.
         Two minutes.

  09:18  DIFF
           newly parsing        39,333
           still failing           223  (signature C — a rare
                                        malformed record type,
                                        pre-existing)
           service field       0.2% → 99.1%
           downstream          +340 aggregates · +12 edges
           REGRESSIONS         none

  09:19  operator clicks APPLY, with the reason "FortiOS 7.6
         grammar fix, verified against 48h."
         → reprocessed observations merge from the resolve stage
         → the 48-hour reachability gap closes
         → NO coverage window re-emitted; nothing is retracted

  09:20  the source returns to HEALTHY. Parse rate 99.5%.

  TOTAL DATA THAT LEFT MERIDIAN'S NETWORK
    one redacted diagnostic bundle: counts, field-shape summaries,
    and structurally anonymised samples.

  TOTAL RAW LOG LINES THAT LEFT
    zero.
```

Without the journal, those 48 hours are simply gone, the graph has a hole nobody can fill, and the only way to debug the grammar would have been to ask Meridian to send us their firewall logs — which is the one thing the architecture exists to make unnecessary.

---

*End of the ingestion series. Back to the [index](00-index.md).*

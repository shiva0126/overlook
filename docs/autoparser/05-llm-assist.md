# LLM Assistance

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

Where a language model genuinely helps the auto-parser, where it must not be used, and the constraint that makes our situation different from everyone else's.

The 2024–2026 research is unambiguous that LLMs beat classical parsers on template accuracy. It is equally unambiguous, once you read the deployment assumptions, that **none of those papers is deployable in our architecture without modification.**

---

## 2. The constraint nobody else has

```
  WE CANNOT SEND CUSTOMER LOG LINES TO A HOSTED MODEL.

  Not as policy. Structurally. The entire premise is that customer
  data does not leave their environment. A support engineer who
  receives a log file has broken it; an API call that ships one to
  a third party is worse, because it is automated and continuous.

  THIS RULES OUT
    ✕ any hosted LLM API in the parsing path
    ✕ sending raw log lines anywhere for inference
    ✕ "we'll just redact PII first" — a log line's structure and
      vocabulary are themselves customer information, and
      redaction that is good enough to be safe is good enough to
      destroy the signal

  IT LEAVES TWO OPTIONS
    1  a SMALL LOCAL MODEL on the collector
    2  inference over STRUCTURAL REPRESENTATIONS only — templates
       with values stripped, field shapes, type distributions —
       which carry no customer data and CAN leave
```

This is why the research trajectory matters. **LibreLog and OpenLogParser are unsupervised and run on open-source models**, and OpenLogParser reports better group and parsing accuracy than the semi-supervised LILAC. The approaches we can actually deploy are also, currently, the better-performing ones.

---

## 3. Where a model helps, and where it does not

```
  L0  FORMAT DETECTION      ✕ NO. Deterministic parsers and
                              signature matching are faster, exact,
                              and explainable. A model here is
                              strictly worse.

  L1  STRUCTURE EXTRACTION  ✕ NO. It is parsing, not inference.

  L2  TEMPLATE MINING       ~ MARGINAL. Drain is accurate, streams,
                              and costs almost nothing. An LLM
                              improves template accuracy on hard
                              corpora but at a cost that only makes
                              sense for templates Drain fails on —
                              which is the LILAC pattern: cache
                              first, model only on a miss.

  L3  FIELD TYPING          ✓ USEFUL. Particularly for the bare-
                              username and free-text-versus-
                              hostname cases where shape gives
                              nothing and context is linguistic.

  L4  SEMANTIC MAPPING      ✓✓ THIS IS THE ONE.
                              The verb is in the constant tokens,
                              in plain English. "Accepted publickey
                              for X from Y" carries its own
                              semantics, and reading that is
                              exactly what a language model is for.
                              Every vendor still uses a human here.
```

**The value is concentrated at L4, and L4 is the cheapest place to use it** — because L4 runs once per *template*, not once per record. A source with 141 templates needs 141 inferences, not 4.1 million.

---

## 4. The architecture — LILAC's insight, taken further

LILAC's contribution is an **adaptive parsing cache**: the LLM generates templates, the cache stores and refines them, and the model is not called per line. It reports +69.5% average F1 on template accuracy over prior state of the art.

We take that one step further:

```
  LILAC          model generates templates → cache serves the rest
                 the model is still in the system at runtime

  OVERLOOK       model generates a PROPOSAL → a human confirms →
                 the result is FROZEN into a deterministic parser
                 THE MODEL IS NOT IN THE SYSTEM AT RUNTIME AT ALL

  WHY THE EXTRA STEP
    DETERMINISM   journal replay must reproduce byte-identical
                  output (../ingestion/07). A model in the path
                  makes that impossible, and replay is our only
                  debugging mechanism.
    AUDITABILITY  "why did this field map that way" must have an
                  answer a human can read. A frozen grammar can be
                  read; a model's activation cannot.
    COST          4.1 billion records/day. There is no inference
                  budget that survives contact with that number.
    STABILITY     a model that answers slightly differently after
                  an update silently changes the graph.
```

---

## 5. What the model actually sees

```
  IT SEES — a structural summary, assembled offline

    template pattern      "Accepted publickey for <*> from <*> port <*> ssh2"
    variable positions    [6, 8, 10]
    typed verdicts        [username_bare 0.94, ipv4 0.99, port 0.97]
    value SHAPES          ["lowercase alnum 3-12", "IPv4", "int 1-65535"]
    cardinality           [341, 1204, 8891]
    source context        linux/rsyslog, host is domain-joined
    our predicate vocabulary and their definitions
    a handful of confirmed mappings from other sources, as
      in-context examples

  IT DOES NOT SEE
    ✕ raw log lines
    ✕ actual values — no usernames, no IPs, no hostnames
    ✕ anything that identifies the customer

  THE PROMPT IS A SCHEMA QUESTION, NOT A DATA QUESTION:
    "given this template pattern and these typed positions, does
     this record assert a relationship from our closed predicate
     vocabulary? If so, which, between which positions, with what
     mechanism? If not, say so."
```

**Because the model sees only structure, the inference could in principle run anywhere.** We still run it locally by default — but the design does not *depend* on locality for its privacy properties, which is a much stronger position than "we promise to redact."

---

## 6. Model choice

```
  DEFAULT — a small local model on the collector
    3-8B parameter class, quantised, ONNX or GGUF, CPU inference
    runs in the SCANNER pool, never the realtime pool
    a few hundred inferences per source onboarding, offline
    → adds ~2-4 GB to the collector image and no runtime cost

  OPTION — customer-provided model endpoint
    a customer with their own internal LLM deployment may point us
    at it. Their infrastructure, their data boundary, their choice.

  OPTION — Overlook-hosted, structural payloads only
    permitted because the payload carries no customer data (§5),
    but OFF BY DEFAULT and shown explicitly in the privacy policy
    as a toggle with what would be sent.

  NEVER — a hosted model receiving raw log content.
```

---

## 7. Handling what models get wrong

```
  HALLUCINATED PREDICATES
    the model proposes CAN_ESCALATE_TO, which is not in our
    vocabulary
    → VALIDATED against the closed enum (../13-contracts.md §7).
      Rejected before a human ever sees it. Not a suggestion,
      an error.

  OVER-CONFIDENT MAPPING
    "Failed password for X" mapped to AUTHENTICATES_TO because the
    structure resembles a successful login
    → the OUTCOME RULE (04 §4) is deterministic and runs AFTER the
      model. A denied or failed outcome vetoes any capability edge
      regardless of what the model said.

  INCONSISTENCY ACROSS RUNS
    the same template proposed differently on two invocations
    → run N times, require agreement. Disagreement lowers
      confidence and pushes it below the human-confirmation
      threshold, which is the correct outcome.

  PLAUSIBLE NONSENSE
    a fluent, wrong explanation
    → the model NEVER decides. It proposes. A human confirms, and
      the proposal is shown with its evidence: the template, the
      typed positions, the corroboration, the confidence.
```

**Every one of these is caught by a deterministic check, not by trusting the model less.** The model's output passes through vocabulary validation, the outcome rule, consistency checking, and human confirmation before it becomes a parser.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Raw logs sent to a hosted model | The privacy premise is broken | Structural payloads only; local by default |
| Model in the ingest path | Replay non-deterministic, cost unbounded | Model produces artifacts, never processes traffic |
| Hallucinated predicate accepted | Vocabulary drift, untraversable edges | Closed-enum validation before human review |
| Model overrides the outcome rule | False edges from failed logins | Deterministic rules run after, and veto |
| Inconsistent proposals | Silent graph changes between runs | N-run agreement; disagreement lowers confidence |
| Model becomes load-bearing | Cannot ship without it, cannot audit it | Everything the model does is optional; the auto-parser must work without it, at lower coverage |
| Local model bloats the image | Collector too large to deploy | Quantised small model, and it is an optional package |

---

## 9. Considerations

**The auto-parser must work without a model at all.** L0–L3 are deterministic. L4 without a model falls back to shipped mappings, template-constant heuristics, and human declaration — which is exactly what every vendor ships today. The model raises coverage of the long tail; it is not a dependency.

**The proposal is the product, not the inference.** What the operator receives is a reviewable artifact — template, types, proposed mapping, evidence, confidence. Whether a model or a heuristic produced it should be visible but should not change how it is reviewed.

**Run inference on template diversity, not on volume.** 4.1 million records with 141 templates is 141 inferences. A source with 4,000 templates should trigger a warning about parsing suitability (`02 §6`) rather than 4,000 inferences.

**Do not use a model for anything deterministic.** Format detection, structure extraction, outcome rules, vocabulary validation, confidence arithmetic. Every one of those is exact, fast and explainable, and a model makes each strictly worse.

---

## 10. Example: Meridian, the sudo template

```
  TEMPLATE  t_29
    "sudo: <*> : TTY=<*> ; PWD=<*> ; USER=<*> ; COMMAND=<*>"

  WHAT WENT TO THE MODEL  (structural, no values)

    pattern:  "sudo: <*> : TTY=<*> ; PWD=<*> ; USER=<*> ; COMMAND=<*>"
    positions:
      2:  username_bare  conf 0.94  shape "lowercase alnum 3-12"  card 41
      6:  string         conf 0.60  shape "device path"           card 12
      9:  posix_path     conf 0.91                                card 340
      13: username_bare  conf 0.91  shape "lowercase alnum 3-12"  card 4
      17: string         conf 0.50  shape "free text"             card 2841
    source: linux/rsyslog, domain-joined host
    vocabulary: [MEMBER_OF, CAN_ASSUME, AUTHENTICATES_TO, CAN_READ,
                 CAN_WRITE, CAN_EXECUTE, ...] with definitions
    examples: 3 confirmed mappings from other sources

  WHAT DID NOT GO
    ✕ "sudo: priyas : TTY=pts/0 ; PWD=/home/priyas ; USER=root ;
        COMMAND=/usr/bin/systemctl restart nginx"
    ✕ any username, any path, any command

  MODEL PROPOSAL, 3 runs, unanimous

    predicate:  CAN_ASSUME
    subject:    position 2
    object:     position 13
    mechanism:  sudo
    rationale:  "the sudo command records a principal at position 2
                 executing as the principal at position 13. Position
                 13's cardinality of 4 against position 2's 41 is
                 consistent with a small set of privileged target
                 accounts."
    confidence: 0.80

  DETERMINISTIC CHECKS APPLIED AFTER

    vocabulary validation   CAN_ASSUME is in the enum          ✓
    outcome rule            no failure/denial token present     ✓
    log-derived cap         0.80 applied                        ✓
    combined confidence     0.91 × 0.80 × 0.80        = 0.58
    threshold 0.60          BELOW → proposed, NOT activated

  → surfaced to the operator with the model's rationale shown
  → operator confirms → authority 1.00 → 0.75 → activated
  → FROZEN into the parser. The model is never consulted again
    for this template.
```

### 10.1 And one the model got wrong

```
  TEMPLATE  t_88
    "Connection closed by authenticating user <*> <*> port <*>"

  MODEL PROPOSAL
    predicate:  AUTHENTICATES_TO
    rationale:  "the phrase 'authenticating user' indicates an
                 authentication relationship between the named
                 principal and the host."
    confidence: 0.75

  Fluent, plausible, and WRONG. This message is emitted when a
  connection is closed DURING authentication — it is a failure,
  not a success.

  CAUGHT BY
    the outcome rule saw "Connection closed by" in the constants
    and classified the outcome as INDETERMINATE, not success.
    → capability edge vetoed
    → downgraded to EVENT_SUMMARY
    → surfaced to the operator as "model proposed an edge; the
      outcome rule vetoed it. Review?"

  The operator agreed with the rule. The template now carries an
  explicit NO_EDGE verdict, and that verdict is shareable to
  other deployments (07).
```

**The model was overruled by a deterministic rule, and the disagreement was shown rather than hidden.** That is the pattern the whole layer depends on.

---

*Next: [Confirm and freeze](06-confirm-and-freeze.md)*

**Sources:** [LILAC](https://arxiv.org/pdf/2310.01796) · [LibreLog / OpenLogParser](https://arxiv.org/html/2408.01585v1) · [LogPPT](https://github.com/LogIntelligence/LogPPT)

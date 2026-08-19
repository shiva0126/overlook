# L4 — Semantic Mapping

**Series:** [Auto-parser](00-index.md) · **The actual problem**

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

Decide what a parsed record **means** in the graph.

L0–L3 are mechanical: detect the format, mine the template, type the tokens. They can be evaluated against a benchmark and scored. L4 cannot, because it asks a question no benchmark covers:

```
  DOES THIS RECORD ASSERT A RELATIONSHIP?
  IF SO — which predicate, between which entities, with what
  confidence, and under what mechanism?
  IF NOT — say so, and produce nothing.
```

**No vendor automates this.** Chronicle, Cribl, Splunk, Elastic and Datadog all stop at "here are the extracted fields" and ask a human which schema field each belongs in. That gap is where the auto-parser earns its name.

---

## 2. Why ours is harder than a SIEM's

```
  A SIEM MAPS FIELD → FIELD
    "srcip" → UDM principal.ip
    a bounded, flat problem. Every field has an answer, and a
    wrong answer is a cosmetic defect.

  WE MAP RECORD → OBSERVATION
    subject canonical key · predicate · object canonical key ·
    mechanism · conditions · confidence
    → a structural assertion, and a wrong answer is a FALSE EDGE
      that propagates into paths, scores and findings

  AND MOST RECORDS MAP TO NOTHING.
    A SIEM parser that extracts nothing has failed.
    Ours that extracts nothing has usually succeeded.
```

---

## 3. The mapping decision, in order

```
  STEP 1  IS THERE AN ACTOR?
          a field typed as an identity canonical key candidate,
          in a position the template's constants suggest is a
          subject
          → no actor, no relationship. Stop. Emit nothing, or
            emit a PROPERTY/EVENT_SUMMARY if the record describes
            state rather than action.

  STEP 2  IS THERE A TARGET?
          a second entity — asset, datastore, identity, application
          → no target → the record may still be an EVENT_SUMMARY
            about the actor, but it asserts no edge.

  STEP 3  WHAT PREDICATE?
          from the template's CONSTANT TOKENS, which carry the verb
            "Accepted publickey for"    → AUTHENTICATES_TO
            "session opened for user"   → AUTHENTICATES_TO
            "sudo: ... USER=root"       → CAN_ASSUME
            "Failed password for"       → NOTHING (an attempt is
                                          not a capability)
          → the predicate must come from the closed vocabulary
            (../13-contracts.md §7). No invention.

  STEP 4  WHAT MECHANISM?
          significant per predicate (13 §9). "publickey" and
          "password" are different mechanisms on the same predicate
          and must not merge.

  STEP 5  WHAT CONFIDENCE?
          the product of typing confidence, mapping confidence, and
          the intrinsic weakness of log-derived assertions (§6).

  STEP 6  DOES IT SURVIVE THE THRESHOLD?
          below it → the mapping is proposed to a human and not
          activated. See 06.
```

---

## 4. The distinction that governs everything

```
  OBSERVED ACTION          "priya authenticated to web-04 at 09:14"
                           → a fact about the PAST
                           → AUTHENTICATES_TO, weight 0.70
                           → tells you a capability EXISTS, weakly,
                             because it was exercised

  CONFIGURED CAPABILITY    "priya is in group Server-Admins"
                           → a fact about what IS POSSIBLE
                           → MEMBER_OF, weight 0.99
                           → the authoritative form

  ATTEMPTED ACTION         "failed password for priya"
                           → asserts NOTHING about capability
                           → an attempt is evidence of intent, not
                             of access
                           → EVENT_SUMMARY at most. NEVER an edge.
```

**The third case is where log-derived mapping most often goes wrong.** A naive mapper sees an identity and an asset in the same record and emits an edge. Failed authentications, denied firewall rules and blocked actions all contain both, and all assert the opposite of a capability.

```
  RULE: a record describing a DENIED, FAILED or BLOCKED outcome
  NEVER produces a capability edge. It may produce a FINDING.
```

---

## 5. Where the mapping knowledge comes from

Four sources, in descending authority.

```
  1  SHIPPED MAPPINGS                              authority 1.00
     for known vendors and known formats, we ship the mapping as
     content. FortiOS traffic logs, PAN-OS, Windows Security event
     IDs, sshd. These are hand-written, reviewed, and versioned.
     → this is what Chronicle does with CBN, and it is correct.
       Mining is for the long tail, not the known.

  2  TEMPLATE-CONSTANT INFERENCE                   authority 0.75
     the verb is in the constant tokens. "Accepted publickey for
     <*> from <*>" carries its own semantics in plain English, and
     a small local model reads it well (05).

  3  CROSS-SOURCE CORROBORATION                    authority 0.70
     if this template's proposed edge already exists in the graph
     from an authoritative source, the mapping is probably right.
     If AD says priya is in Server-Admins and this template
     proposes priya AUTHENTICATES_TO a server in that group's
     scope, the two agree.

  4  OPERATOR DECLARATION                          authority 1.00
     a human maps it in the Controller. Always wins, always
     recorded, always reusable.
```

---

## 6. Log-derived edges are intrinsically weaker

```
  A capability asserted from a log line is ALWAYS weaker than the
  same capability read from configuration.

    AD group membership       MEMBER_OF   confidence 0.99
    a login event implying
    the same membership       weight 0.70, confidence ≤0.75

  WHY
    · the log records ONE occurrence, not a standing grant
    · it may reflect a permission since revoked
    · attribution may be wrong (shared accounts, service contexts)
    · the record may be about a different entity than it appears

  CONSEQUENCE
    log-derived edges are capped. No mapping from L4 may produce
    confidence above 0.80, regardless of how certain the typing was.
    Configuration sources are the only path to 0.9+.
```

This cap is what stops a well-parsed log source from outranking the authoritative configuration it describes.

---

## 7. Most records map to nothing, and that must be cheap

```
  MERIDIAN'S LINUX SOURCE
    141 promoted templates
      3 assert relationships
    138 assert nothing

  MERIDIAN'S FIREWALL TRAFFIC
    ~40 templates after sub-classification
      1 asserts CONNECTS_TO, and only after aggregation
     39 assert nothing

  A mapper that has to reason about every record to conclude
  "nothing" is too expensive. So:

    · the NOTHING verdict is baked into the frozen parser (06)
    · at ingest, a record matching a no-op template is counted
      and discarded, with no per-record reasoning at all
    · the reasoning happened once, offline, during proposal
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Failed/denied mapped as a capability | **False edges from every failed login and blocked packet** | Outcome check before edge emission; denied never produces capability |
| Predicate invented | Vocabulary drift, path engine cannot traverse | Closed enum, enforced at validation |
| Log edge outranks config | A stale log assertion beats the authoritative source | Hard confidence cap at 0.80 for log-derived |
| Subject and object confused | Edge direction reversed — a path that runs backwards | Template constants determine roles; low confidence when ambiguous |
| Mechanism omitted | Distinct realities merge into one edge (13 §9) | Mechanism required for every predicate that declares it significant |
| Eager mapping | Graph fills with weak assertions, diluting every score | Threshold, and human confirmation below it |
| Shared-account attribution | One service account's actions attributed as a person's | Detect low-cardinality high-volume accounts; flag rather than assert |

---

## 9. Considerations

**The default answer is "nothing."** A mapper biased toward producing edges will produce false ones, and a false edge is far more damaging than a missing one — it propagates into paths, inflates scores, and is discovered by a customer rather than by us. Bias toward silence, and surface what was skipped so a human can review it.

**Ship mappings for known vendors; mine only the long tail.** Chronicle maintains hundreds of hand-written parsers because a reviewed grammar beats an inferred one every time. The auto-parser's job is the device nobody wrote one for, not to replace the ones we have.

**Cross-source corroboration is our unfair advantage and it compounds.** We hold an entity store built from authoritative sources, so we can check a proposed mapping against what we already know. That signal is unavailable to a standalone parser and it gets stronger as more connectors are added.

**Every mapping decision must be inspectable.** An operator asking *"why did this log line become an edge?"* must get the template, the typed fields, the mapping rule, the authority, and the confidence — not a model's opinion.

---

## 10. Example: Meridian, three mappings and two refusals

### 10.1 Accepted — an authentication

```
  TEMPLATE  t_04  "Accepted publickey for <*> from <*> port <*> ssh2"
  TYPED     pos_6 username_bare  0.94 → sam:<domain>\<value>
            pos_8 ipv4           0.99 → resolves via DHCP lease
            pos_10 port          0.97

  STEP 1  ACTOR?    pos_6, identity candidate                    ✓
  STEP 2  TARGET?   the HOST EMITTING THE LOG — from source
                    context, not from the line                   ✓
  STEP 3  PREDICATE constants "Accepted ... for ... from"
                    → AUTHENTICATES_TO                           0.85
  STEP 4  MECHANISM "publickey" → mechanism: ssh_publickey
  STEP 5  CONFIDENCE 0.94 × 0.85 × log_cap(0.80)          = 0.64
  STEP 6  THRESHOLD 0.60 → ACCEPTED, proposed for confirmation

  OBSERVATION
    subject   sam:corp\priyas          (domain from host context)
    predicate AUTHENTICATES_TO
    object    fqdn:web-prod-04.corp.meridian.com
    mechanism ssh_publickey
    confidence 0.64
```

Note that **the target came from source context, not from the record**. The log line does not name the host it happened on. Getting that from the source binding rather than the payload is routine and easy to get wrong.

### 10.2 Accepted — a genuine privilege assertion

```
  TEMPLATE  t_29  "sudo: <*> : TTY=<*> ; PWD=<*> ; USER=<*> ; COMMAND=<*>"
  TYPED     pos_2 username_bare 0.94 · pos_10 username_bare 0.91

  STEP 1  ACTOR?     pos_2                                       ✓
  STEP 2  TARGET?    pos_10 — ANOTHER IDENTITY                   ✓
  STEP 3  PREDICATE  "sudo ... USER=" → CAN_ASSUME               0.80
  STEP 4  MECHANISM  sudo
  STEP 5  CONFIDENCE 0.91 × 0.80 × 0.80                   = 0.58
  STEP 6  THRESHOLD  below 0.60 → PROPOSED, NOT ACTIVATED

  → surfaced to the operator: "this template appears to assert
    privilege escalation via sudo. Confirm?"
  → operator confirms → authority 1.00 → confidence rises to 0.75
  → and CROSS-SOURCE CORROBORATION: /etc/sudoers is not collected
    at Meridian, so this is the ONLY source of sudo-derived
    CAN_ASSUME edges. The operator's confirmation matters.
```

### 10.3 Refused — a failed authentication

```
  TEMPLATE  t_11  "Failed password for invalid user <*> from <*> port <*>"
  TYPED     pos_6 username_bare 0.94 · pos_8 ipv4 0.99

  STEP 1  ACTOR?     pos_6                                       ✓
  STEP 2  TARGET?    the emitting host                           ✓
  STEP 3  PREDICATE  constants contain "Failed" and "invalid user"
                     → OUTCOME IS FAILURE
                     → NO CAPABILITY EDGE. Rule, not a score.

  RESULT
    ✕ no edge
    ✓ EVENT_SUMMARY: failed authentication attempts, bucketed,
      by source and target
    ✓ feeds a posture rule (brute-force pattern), not the graph

  A naive mapper emits IDENTITY AUTHENTICATES_TO ASSET here,
  because both entities are present and the template looks
  structurally identical to t_04. Twelve thousand failed logins
  would become twelve thousand assertions that an attacker's
  guessed usernames can access production hosts.
```

### 10.4 Refused — a firewall traffic line

```
  TEMPLATE  FortiOS type=traffic, key-value
  TYPED     srcip ipv4 · dstip ipv4 · dstport port · action enum

  STEP 1  ACTOR?    srcip is an IP, not an identity.
                    An IP is a WEAK identity candidate at best,
                    and this record names no principal.
                    → NO ACTOR
  RESULT
    ✕ no relationship observation from the individual record
    ✓ aggregated at receive (../ingestion/03) into a
      FLOW_AGGREGATE, from which ONE CONNECTS_TO edge per
      (subnet, subnet, port, protocol) is derived per window

  3.1 billion records → ~40 edges/day, and the mapping decision
  for the individual record is "nothing," decided once and frozen.
```

### 10.5 What the operator saw

```
  ┌──────────────────────────────────────────────────────────────┐
  │ PROPOSED MAPPINGS · linux.rsyslog @ 10.4.2.40                │
  │ 141 templates analysed · 3 assert relationships               │
  │                                                              │
  │ ✓ t_04  Accepted publickey for <*> from <*> port <*> ssh2    │
  │         → IDENTITY AUTHENTICATES_TO ASSET                    │
  │         mechanism ssh_publickey · confidence 0.64            │
  │         12,408 observations · corroborated by AD             │
  │         [ Confirm ]  [ Adjust ]  [ Reject ]                  │
  │                                                              │
  │ ? t_29  sudo: <*> : ... USER=<*> ; COMMAND=<*>               │
  │         → IDENTITY CAN_ASSUME IDENTITY                        │
  │         mechanism sudo · confidence 0.58  BELOW THRESHOLD    │
  │         ⚠ only source of sudo-derived edges — /etc/sudoers   │
  │           is not collected                                   │
  │         [ Confirm ]  [ Adjust ]  [ Reject ]                  │
  │                                                              │
  │ ✕ t_11  Failed password for invalid user <*> ...             │
  │         → NO EDGE. Outcome is failure.                       │
  │         EVENT_SUMMARY only.                                  │
  │                                                              │
  │ 138 templates assert nothing. [ Review ]                     │
  └──────────────────────────────────────────────────────────────┘
```

**"138 templates assert nothing" is shown, not hidden.** An operator who wants to check that we did not miss something can, and the review is the mechanism by which a missed mapping gets found.

---

*Next: [LLM assistance](05-llm-assist.md)*

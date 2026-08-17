# L2 — Template Mining

**Series:** [Auto-parser](00-index.md)

---

## 1. Purpose

For records that survive L0/L1 as **unstructured free text**, separate the constant parts from the variable parts.

```
  Failed password for invalid user admin from 10.4.2.17 port 4122 ssh2
  Failed password for invalid user root  from 10.9.1.3  port 8891 ssh2
  Failed password for invalid user test  from 10.4.8.2  port 2201 ssh2
                    ↓
  TEMPLATE  Failed password for invalid user <*> from <*> port <*> ssh2
  VARIABLES [user, src_ip, src_port]
```

The template is the parser. Everything after this is naming and mapping the variables.

---

## 2. Position

```
  INPUT    unstructured payloads from L1, grouped by source
  OUTPUT   a template set per source, each with variable positions,
           a match count, and a stability score
  FEEDS    L3 field typing (what is each variable?)
           L4 semantic mapping (does this template assert anything?)
```

---

## 3. The algorithm — Drain, and why

The ICSE'19 benchmark (*Tools and Benchmarks for Automated Log Parsing*) evaluated the field and found **Drain the most accurate**, with high accuracy on 9 of 16 datasets against 6 for IPLoM, AEL and Spell. It is also the one that streams.

```
  DRAIN — fixed-depth parse tree

  LAYER 1   partition by TOKEN COUNT
            "Failed password for invalid user X from Y port Z ssh2"
            → 11 tokens → bucket 11

  LAYER 2+  partition by LEADING TOKENS, to a fixed depth (typically 3-4)
            → "Failed" → "password" → "for"

  LEAF      a list of candidate log groups, each with a template.
            Compare the incoming message to each by SIMILARITY:
              sim = (matching token positions) / (total tokens)
            Above threshold (typically 0.5) → merge into that group,
            replacing differing positions with the wildcard.
            Below → create a new group.

  PROPERTIES
    ONLINE     one pass, no batch phase. Fits a streaming receiver.
    BOUNDED    tree depth is fixed, so lookup is effectively
               constant work per message.
    INCREMENTAL new formats create new groups without reprocessing.
```

### 3.1 Why not the others

```
  SPELL       longest common subsequence, also streaming.
              Accurate on 6/16. LCS is more expensive per message
              than a depth-bounded tree walk.

  IPLoM       iterative partitioning by length, token position and
              mapping relation. Accurate, but BATCH — it needs the
              corpus. Wrong shape for a receiver.

  AEL         compares occurrence counts of constants vs variables.
              Also corpus-oriented.

  LogSig · LogMine · LenMa
              clustering formulations. Batch, and sensitive to
              distance-metric choice.

  Logram      n-gram dictionaries. Fast, simpler, lower accuracy.

  → Drain is chosen for the combination of accuracy AND streaming,
    not accuracy alone.
```

---

## 4. Adaptations we need

Vanilla Drain assumes a homogeneous stream. A security appliance is not one.

```
  A1  PARTITION BY SOURCE FIRST
      Never mine templates across sources. A FortiGate and a Linux
      host share token counts by coincidence and nothing else.
      → one Drain tree per (source, sub-format).

  A2  PRE-MASK OBVIOUS VARIABLES BEFORE THE TREE
      IPs, timestamps, hex strings, UUIDs, MAC addresses, numbers
      above a length threshold.
      → dramatically improves grouping, because otherwise every
        distinct IP creates tree pressure at layer 2.
      → this is the single highest-value tuning knob.

  A3  BOUND THE TEMPLATE SET
      A source producing >2,000 templates is not being parsed, it
      is being memorised. Cap it, and surface the cap.
      → usually means A2 masking is too weak, or the source is
        genuinely MIXED (01 §6) and needs sub-classification.

  A4  STABILITY BEFORE PROMOTION
      A template seen 3 times in an hour is noise. A template seen
      500 times over 24 hours across 3 restarts is a format.
      → templates carry a stability score and are not promoted to
        L3/L4 until they cross it.

  A5  VARIABLE-POSITION CONFIDENCE
      A position that is variable in 500 of 500 observations is
      certainly a variable. One variable in 3 of 500 is probably a
      constant with rare exceptions — a hostname that is almost
      always the same.
      → record the variability ratio per position.
```

---

## 5. What a mined template carries

```yaml
template:
  id: t_7f3a2c
  source: linux.rsyslog@10.4.2.40
  pattern: "Failed password for invalid user <*> from <*> port <*> ssh2"
  token_count: 11
  variable_positions: [6, 8, 10]
  variability_ratio: [1.00, 1.00, 1.00]

  observations: 12_408
  first_seen: 2026-08-10T02:14:00Z
  last_seen:  2026-08-17T09:41:00Z
  distinct_days: 7
  stability: 0.94          # promoted

  sample_values:           # RETAINED LOCALLY, never transmitted
    pos_6:  [admin, root, test, oracle, ...]
    pos_8:  [10.4.2.17, 10.9.1.3, ...]
    pos_10: [4122, 8891, 2201, ...]
```

**`sample_values` is what L3 types and what L4 maps.** It is also customer data and never leaves the appliance — it exists to inform local inference and human confirmation, nothing else.

---

## 6. Template count is a health signal

```
  A WELL-BEHAVED SOURCE
    Linux rsyslog        40-200 templates
    a firewall's events  20-80
    an application       50-500

  A BADLY-BEHAVED ONE
    >2,000 templates     masking is too weak, or the source is MIXED
    1 template           over-merged — the similarity threshold is
                         too low and everything collapsed
    template churn       new templates appearing constantly with no
                         stabilisation → the source emits genuinely
                         unbounded text (stack traces, free-form
                         messages) and is a poor parsing candidate

  Template count and churn rate belong in the Controller alongside
  parse rate. They are the earliest signal that a source will never
  parse well, and knowing that at onboarding is worth a great deal.
```

---

## 7. What template mining does NOT do

```
  ✕ it does not name anything
      "<*>" in position 6 is a variable. That it is a USERNAME is
      L3's job.

  ✕ it does not know if a template matters
      "Failed password for <*>" and "Session opened for <*>" are
      equally well-mined and only one of them may assert anything
      we care about. That is L4.

  ✕ it does not handle multi-line events
      stack traces, Windows event XML spanning lines. Multi-line
      assembly happens at framing (../ingestion/03 §3), before
      mining, or not at all.

  ✕ it is not a substitute for a vendor grammar
      where a maintained grammar exists (FortiOS, PAN-OS), USE IT.
      Mining is for the long tail, not the known.
```

That last point matters. Chronicle maintains hundreds of parsers because a hand-written grammar for a known vendor beats a mined template every time — it knows field semantics, not just positions. **Mining is what we do when nobody wrote one.**

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| No pre-masking | Every distinct IP creates tree pressure; thousands of near-identical templates | A2 masking before the tree |
| Similarity threshold too low | Over-merging — unrelated messages share a template | Tune per source; alarm at template count = 1 |
| Similarity threshold too high | Template explosion | Cap and surface; usually a masking problem |
| Mining across sources | Meaningless groups | One tree per (source, sub-format) |
| Unstable templates promoted | Parsers built on noise | Stability score gate |
| Unbounded-text source | Never stabilises, template count climbs forever | Detect churn, flag as a poor parsing candidate at onboarding |
| Multi-line events | Each line mined separately, all meaningless | Assemble at framing or not at all |

---

## 9. Considerations

**Mining runs on a sample, not on the stream.** Template mining is a *learning* activity, not an ingest activity. It runs over a bounded sample from the journal — 24 to 72 hours — in the scanner pool, producing a template set. It does not run per record at ingest. Once a parser is frozen (`06`), mining is not in the hot path at all.

**Pre-masking is where the accuracy actually comes from.** More than the algorithm choice. Masking IPs, timestamps, UUIDs, hex, MACs and long numbers before the tree is the difference between 80 templates and 8,000 for the same source.

**Sample values are retained locally and are sensitive.** They are literal fragments of customer logs. Encrypted at rest, TTL'd, never transmitted, and shown only in the Controller under RBAC.

**A source that will not stabilise should be declared early.** Some sources emit genuinely unbounded text. Discovering that at onboarding — "this source produces 4,000 templates and climbing, it will not parse well" — is far better than discovering it three months later as a mysterious coverage gap.

---

## 10. Example: Meridian, the Linux host

```
  SOURCE  linux.rsyslog @ 10.4.2.40 — 900 Linux servers forwarding
  SAMPLE  48 hours from the journal, 4.1 million records
  RUN     scanner pool, 6 minutes

  PRE-MASKING APPLIED
    IPv4/IPv6 · ISO and syslog timestamps · UUIDs · hex ≥8 chars ·
    MAC addresses · integers ≥5 digits · file paths

  RESULT
    templates mined            184
    promoted (stability ≥0.8)  141
    below threshold             43   (seen <10 times, or <2 days)
    coverage of the sample     97.2% of records match a promoted
                                     template

  TOP TEMPLATES BY VOLUME
    1  "Accepted publickey for <*> from <*> port <*> ssh2"     41%
    2  "session opened for user <*> by (uid=<*>)"              18%
    3  "Failed password for invalid user <*> from <*> port <*> ssh2"  9%
    4  "sudo: <*> : TTY=<*> ; PWD=<*> ; USER=<*> ; COMMAND=<*>"      6%
    5  "New session <*> of user <*>."                           5%
    ... 136 more

  WITHOUT PRE-MASKING (measured, for comparison)
    templates mined          6,209
    coverage                 61%
    → because every distinct source IP in template 1 and 3 spawned
      its own group

  → the masking step is worth 34× on template count and 36 points
    of coverage for this source alone.
```

### 10.1 The three that mattered

Of 141 promoted templates, L4 later determined that **three assert relationships**:

```
  t_04  "Accepted publickey for <*> from <*> port <*> ssh2"
        → IDENTITY AUTHENTICATES_TO ASSET
  t_11  "session opened for user <*> by (uid=<*>)"
        → IDENTITY AUTHENTICATES_TO ASSET (corroborating)
  t_29  "sudo: <*> : ... USER=<*> ; COMMAND=<*>"
        → IDENTITY CAN_ASSUME IDENTITY  (privilege escalation
          actually exercised)

  The other 138 assert nothing. They are correctly parsed, and
  correctly produce no observation.

  4.1 million records → 3 useful templates → a few hundred
  observations per day after aggregation.

  That ratio is not a failure. It is what "correctly nothing"
  (00 §3) looks like in practice.
```

---

*Next: [Field typing](03-field-typing.md)*

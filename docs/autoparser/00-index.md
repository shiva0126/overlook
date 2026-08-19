# Overlook — The Auto-Parser

**Version:** 0.1
**Date:** 2026-08-17
**Status:** Architecture. No implementation.
**Supersedes:** the deferral of E3 in `../10-collector-stack-and-engines.md §4.2`. See §5.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. The tension, stated first

`../14-ingestion-and-sources.md` measures log sources at **0.00003 edges per megabyte** and cloud IAM config at **17**. On that analysis, a perfect auto-parser applied to firewall telemetry produces almost nothing, and `../10 §4.2` deferred the parser engine entirely because every v1 source returns JSON.

That analysis is correct and it measures the wrong thing.

```
  THE AUTO-PARSER'S VALUE IS NOT THE VOLUME IT PARSES.
  IT IS THE NUMBER OF SOURCES IT ONBOARDS WITHOUT AN ENGINEER.

  As an MSSP with N customers, each running 5-10 devices nobody
  wrote a connector for, the choice is:

    per-source engineering    every unusual device is a ticket,
                              a release train, and a delay
                              → Stellar Cyber's model (08 §2.3),
                                which we explicitly rejected

    auto-parsing              an unusual device is an afternoon
                              of configuration by the customer's
                              own operator
```

That is a **main functional capability**, and it is the difference between a business that scales to thirty customers and one that becomes a bespoke integration shop. It is just not valuable for the reason "logs contain a lot of edges." They do not.

---

## 2. What "auto-parsing" actually means

Five distinct problems that get bundled under one word. They have different solutions and different difficulty.

```
  L0  FORMAT DETECTION      is this JSON, XML, CSV, key-value, CEF,
                            LEEF, RFC5424, RFC3164, or free text?
                            → solved. Everyone does it.

  L1  STRUCTURE EXTRACTION  for structured formats, fields come free
                            → solved.

  L2  TEMPLATE MINING       for unstructured text, separate the
                            CONSTANT parts from the VARIABLE parts
                            → solved well by Drain and successors.

  L3  FIELD TYPING          is this token an IP, a timestamp, a
                            username, a hostname, a path, a hash,
                            an ARN?
                            → partially solved. Regex plus context.

  L4  SEMANTIC MAPPING      does this field become a CANONICAL KEY?
                            does this line ASSERT A RELATIONSHIP?
                            → NOT solved by anyone. This is where
                              every vendor still uses a human.

  L5  CONFIRMATION          a human validates, and the result is
                            FROZEN into a deterministic parser
                            → the part most auto-parser designs skip,
                              and the reason they cannot be trusted
```

L0–L2 are engineering. **L4 is the actual problem**, and it is harder for us than for a SIEM.

---

## 3. Why our L4 is different — and harder

```
  A SIEM'S AUTO-PARSER
    goal: turn a log line into a SEARCHABLE EVENT
    target: "which UDM/OCSF field does this token belong in?"
    success: the field is populated and queryable
    every line has an answer

  OVERLOOK'S AUTO-PARSER
    goal: turn a log line into ENTITIES AND RELATIONSHIPS
    target: "is this token a canonical key? does this line assert
             an edge? which predicate? with what confidence?"
    success: a correct observation, or CORRECTLY NOTHING
    MOST LINES ASSERT NOTHING AT ALL
```

**"Correctly nothing" is the unusual requirement.** A SIEM parser that extracts every field from every line is doing its job. Ours must recognise that 99% of a firewall traffic stream asserts no relationship worth keeping, aggregate it (`../ingestion/03`), and emit an observation only when a line genuinely says *this identity did something to that asset*.

A parser that eagerly manufactures edges from log lines is worse than no parser, because it fills the graph with low-confidence assertions that then dilute every path score.

---

## 4. The research landscape

Recorded so it does not need re-finding.

### 4.1 Template mining — the classical algorithms

Log parsing is defined as converting each message into an event template by removing parameters and keeping keywords.

```
  HEURISTIC / TREE

  DRAIN      fixed-depth parse tree. First layer partitions by
             token count, then by leading tokens, then similarity
             match against cached log groups.
             ONLINE / STREAMING. Effectively constant work per
             message.
             → SOTA among traditional unsupervised parsers.
               High accuracy on 9 of 16 benchmark datasets.

  SPELL      longest common subsequence, streaming.
             High accuracy on 6 of 16.

  IPLoM      iterative partitioning — by message length, then token
             position, then mapping relation. 6 of 16.

  AEL        separates messages into groups by comparing occurrence
             counts of constants versus variables. 6 of 16.

  CLUSTERING

  LogSig · LogMine · LenMa
             formulate parsing as clustering, with various
             similarity/distance measures between messages.

  N-GRAM

  Logram     n-gram dictionaries. Fast, simpler, lower accuracy.
```

**Benchmark conclusion (ICSE'19, *Tools and Benchmarks for Automated Log Parsing*): Drain is the most accurate**, and it is also the one that streams. For our purposes those two properties together make it the obvious base.

### 4.2 LLM-based parsing — the 2024–2026 wave

```
  LILAC        LLM + ADAPTIVE PARSING CACHE.
               Hierarchical candidate sampling, high-quality
               demonstration selection, and a cache that stores and
               refines LLM-generated templates.
               +69.5% average F1 on template accuracy over prior SOTA.
               → THE ARCHITECTURAL IDEA WE WANT: the LLM is not
                 called per line. It generates a template once;
                 the cache serves everything after.

  LibreLog     unsupervised, open-source LLMs, no labelled logs.
               Parsing accuracy 0.8538 vs LILAC's 0.6783.

  OpenLogParser  unsupervised with open LLMs; better group accuracy
               and parsing accuracy than semi-supervised LILAC.

  LogPPT       prompt-based few-shot learning (ICSE 2023).

  MicLog       AAAI 2026 — progressive meta in-context learning.
```

The trajectory is clear: **unsupervised, open-model approaches now beat semi-supervised ones**, which matters enormously for us because of §4.4.

### 4.3 What vendors actually ship

```
  GOOGLE CHRONICLE — CBN (Config Based Normalizer)
    config files, not code. Grok patterns + regex + JSON path
    expressions extract values; each value is mapped to a UDM field.
    A PARSER EDITOR in the UI: paste raw samples, define mappings,
    see the resulting UDM event immediately.
    A CLI tool and parsers-as-code in GitHub, plus community parsers.
    → the workflow is right. The mapping is still human.

  CRIBL
    built-in parsers for JSON, CSV and syslog that auto-recognise
    structure; a Grok function for unstructured text.

  SPLUNK
    automatic extraction of key=value pairs.

  ELASTIC / DATADOG
    Grok pattern libraries — predefined regexes referenced in a
    pipeline.
```

**Nobody automates L4.** Every one of them stops at "here are the extracted fields" and asks a human which schema field each belongs to. That is the gap, and it is where an LLM genuinely helps.

### 4.4 The constraint nobody else has

```
  WE CANNOT SEND CUSTOMER LOG LINES TO A HOSTED LLM.

  It is the exact thing the architecture exists to prevent. A
  support engineer who receives a log file has broken the premise;
  an API call that ships one to a third party is worse.

  CONSEQUENCES
    · LLM assistance must run LOCALLY, on a small model on the
      collector, or
    · operate only on STRUCTURAL REPRESENTATIONS — templates with
      values stripped, field shapes, type distributions — which
      carry no customer data

  This is why §4.2's trajectory matters: unsupervised open-model
  approaches are the only ones we can actually deploy.
```

---

## 5. What this changes in the existing design

```
  ../10 §4.2 deferred E3 (the parser engine) because every v1
  source returns JSON. That deferral was correct for the ENGINE
  and wrong about the CAPABILITY.

  REVISED POSITION
    · E3 as a grammar RUNTIME is still not needed for v1 sources
    · the AUTO-PARSER is a distinct component that PRODUCES
      grammars, and it is what makes the runtime worth having
    · it is required the moment the first non-JSON source appears,
      which for the picked deployment set is FortiGate — the
      first connector on the list

  So: still not needed for cloud IAM. Needed immediately for the
  deployment set the customer actually picked.
```

---

## 6. The documents

| # | Doc | Covers |
|---|---|---|
| 01 | [Format detection](01-format-detection.md) | L0/L1 — identifying and extracting structured formats |
| 02 | [Template mining](02-template-mining.md) | L2 — Drain and successors, for unstructured text |
| 03 | [Field typing](03-field-typing.md) | L3 — what is this token? |
| 04 | [Semantic mapping](04-semantic-mapping.md) | L4 — the actual problem |
| 05 | [LLM assistance](05-llm-assist.md) | Where a model helps, and the local-only constraint |
| 06 | [Confirm and freeze](06-confirm-and-freeze.md) | Human-in-the-loop, and why the output must be deterministic |
| 07 | [Parser lifecycle](07-lifecycle.md) | Drift, versioning, regression, sharing |
| 08 | [Source manifest and parser registry](08-source-manifest-and-parser-registry.md) | Source budgets, parser selection, fallback, quarantine |

---

## 7. The design commitment that governs all of it

```
  THE AUTO-PARSER PROPOSES. A HUMAN CONFIRMS.
  THE CONFIRMED RESULT IS FROZEN INTO A DETERMINISTIC PARSER.

  NO MODEL RUNS AT INGEST TIME. EVER.

  WHY
    COST          an LLM per line at 4.1 billion lines/day is absurd
    DETERMINISM   replay must reproduce byte-identical output
                  (../ingestion/07). A model in the hot path makes
                  that impossible.
    AUDITABILITY  a customer must be able to see WHY a field mapped
                  the way it did, and a frozen grammar can be read
    PRIVACY       inference on customer data at ingest scale is the
                  thing we forbid

  This is LILAC's adaptive-cache insight taken to its conclusion:
  the model's job is to produce an artifact, not to process traffic.
```

---

*Index. Documents follow.*

**Sources:** [Tools and Benchmarks for Automated Log Parsing (ICSE'19)](https://arxiv.org/pdf/1811.03509) · [LILAC](https://arxiv.org/pdf/2310.01796) · [LibreLog / OpenLogParser](https://arxiv.org/html/2408.01585v1) · [LogPPT](https://github.com/LogIntelligence/LogPPT) · [Logram](https://arxiv.org/pdf/2001.03038) · [Chronicle CBN parsers](https://docs.cloud.google.com/chronicle/docs/ingestion/parsers-github) · [cbn-tool](https://github.com/chronicle/cbn-tool) · [Cribl parser function](https://docs.cribl.io/stream/parser-function/)

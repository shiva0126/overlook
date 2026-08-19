# Overlook — Engine Documentation

**Version:** 0.1
**Date:** 2026-08-14
**Parent:** `../10-collector-stack-and-engines.md`
**Status:** Architecture. No implementation.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## What this series is

One document per component in the collector pipeline. Each explains **how that component works**, **what has to be considered when building it**, and **how it fails** — then ends with a worked example.

Every example uses the same customer: **Meridian Financial**, defined in `../12-end-to-end-deployment-story.md`. The examples are slices of one continuous story, so reading them in order traces a single piece of data from a firewall port to a finding on a screen.

## ⚠ Where each engine runs, per handoff §3

```
  EDGE COLLECTOR        E15 orchestration · receive/journal/aggregator
                        E2 fingerprint · E3 parser · E4 normalizer
                        E5 enrichment · E6 entity resolution (LOCAL,
                        lightweight — handoff §10) · E13 fact builder
                        E14 privacy gate · sign and transport

  OVERLOOK SAAS         E7 permission closure   ⚠ ESCALATION E4
                        E8 escalation matcher
                        E9/E10/E11 posture, correlation, classification
                        E12 graph engine — the collector holds a
                        BOUNDED ENTITY CACHE, not a graph (§10.1)

  Documents for SaaS-side engines are retained: the engineering is
  the same wherever it runs, and the boundary may move (E4).
```

---

## The pipeline

```
  SOURCES ─► E15 ORCHESTRATION ─► RECEIVE + E1 ─► E2 ─► E3 ─► E4 ─► E5
                                                                    │
                                                                    ▼
                                                              E6 RESOLUTION
                                                                    │
                                                                    ▼
                                    E7 CLOSURE ─► E8 ESCALATION     │
                                    E9  E10  E11 (parallel)   ◄─────┘
                                                                    │
                                                                    ▼
                                                            E12 GRAPH ENGINE
                                                                    │
                                            ┌───────────────────────┴─────┐
                                     MODE 1 │                      MODE 2 │
                                  Controller UI              E13 ─► E14 ─► TRANSPORT
```

## The documents

| # | Doc | Engines | Needed in v1? |
|---|---|---|---|
| 01 | [Orchestration](01-orchestration.md) | E15 | **Yes** |
| 02 | [Receive, Journal, Aggregator](02-receive-journal-aggregator.md) | Receive, E1 | Partly — journal yes, aggregator no |
| 03 | [Fingerprint and Parser](03-fingerprint-and-parser.md) | E2, E3 | **No** — all v1 sources are JSON |
| 04 | [Normalizer and Enrichment](04-normalizer-and-enrichment.md) | E4, E5 | **Yes** |
| 05 | [Entity Resolution](05-entity-resolution.md) | E6 | **Yes** — hardest |
| 06 | [Permission Closure](06-permission-closure.md) | E7 | **Yes** — hardest |
| 07 | [Escalation Matcher](07-escalation-matcher.md) | E8 | **Yes** |
| 08 | [Posture, Correlation, Classification](08-posture-correlation-classification.md) | E9, E10, E11 | E9 yes; E10, E11 no |
| 09 | [Graph Engine](09-graph-engine.md) | E12 | **Yes** |
| 10 | [Fact Builder](10-fact-builder.md) | E13 | Mode 2 only |
| 11 | [Privacy Gate](11-privacy-gate.md) | E14 | Mode 2 only |
| 12 | [Sign and Transport](12-sign-and-transport.md) | — | Mode 2 only |

**v1 engine set: E15, E4, E5, E6, E7, E8, E9, E12** — eight engines, as established in `../10 §4.2`.

## Structure of each document

```
  1  Purpose            what it is for, in one paragraph
  2  Position           inputs, outputs, what it depends on
  3  Mechanics          how it actually works
  4  Considerations     the decisions and trade-offs
  5  Failure modes      what goes wrong and what happens then
  6  Contracts          interfaces it must honour
  7  Scope              what we build now, what we defer
  8  Example: Meridian  the worked case, at the end
```

## The recurring example

```
  MERIDIAN FINANCIAL
    12,000 employees · 8,500 endpoints
    CrowdStrike · Forcepoint DLP · 4 firewalls (2 PAN-OS, 2 FortiOS)
    2 AD forests · Entra ID · VMware · Oracle
    AWS 42 accounts · Azure 18 subs · GCP 6 projects

  DEPLOYMENT   COL-DC1 (on-prem, Edge L, resolution primary)
               COL-CLD (AWS private subnet, Edge M)

  THE PERSON   Priya S — developer, no admin rights anywhere,
               seen by 8 different sources under 8 different names

  THE PATH     laptop MCP config → GitHub token → OIDC trust
               → GHADeployRole → PassRole+Lambda → admin
               → prod-payments-db (4.2M records, PII+PCI)
```

---

*Index. See individual documents for detail.*

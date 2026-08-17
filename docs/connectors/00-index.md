# Overlook — Connector and Collector Catalog

**Version:** 0.1
**Date:** 2026-08-14
**Parent:** `../03-connectors.md`
**Status:** Architecture. No implementation.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## What this is

The full breakdown of every connector in the catalog, down to individual collectors. `../03-connectors.md` establishes *why* the catalog is shaped this way and gives domain-level counts. This series gives the actual contents.

**Where collector counts differ from doc 03's estimates, this series supersedes them.** Doc 03 estimated at the domain level; here each collector is enumerated, and enumeration is always messier than estimation.

## The unit definitions

```
  CONNECTOR   one authenticated integration with one source system
              one credential set · one rate-limit domain · one health
              state · one manifest

  COLLECTOR   one data-gathering routine inside a connector
              one API family or object type · one cadence · one
              coverage window · one class of entities or relationships
```

## The files

| # | Domain | Connectors | Collectors |
|---|---|---|---|
| 01 | [Cloud and infrastructure](01-cloud-infrastructure.md) | 10 | 144 |
| 02 | [Identity and access](02-identity-access.md) | 17 | 122 |
| 03 | [Code, build, artifacts](03-code-build-artifacts.md) | 12 | 90 |
| 04 | [Security tooling](04-security-tooling.md) | 18 | 96 |
| 05 | [Network and edge](05-network-edge.md) | 15 | 110 |
| 06 | [Data platforms](06-data-platforms.md) | 18 | 114 |
| 07 | [AI platforms](07-ai-platforms.md) | 15 | 75 |
| 08 | [Business context](08-business-context.md) | 8 | 40 |
| 09 | [Generic ingestion](09-generic-ingestion.md) | 7 | 25 |
| 10 | [The Overlook Agent](10-agent.md) | 1 | 10 |
| | **Total** | **121** | **826** |

Doc 03 estimated 118 connectors and 695 collectors. Enumeration produced **121 and 826** — the difference is almost entirely IAM, network and data connectors being broken into their real collectors rather than treated as single units. Enumeration is always messier than estimation, and the estimate was not wrong so much as coarse.

**None of this changes the build target.** The framework is the deliverable; the catalog is the map (`../08-connector-benchmark-and-alignment.md §7.4`). Six to ten deep connectors produce the five hero findings. The other 111 are a roadmap that gets built per customer, per deployment.

## How to read a connector entry

Each connector has a header block carrying the manifest-relevant facts, then a collector table.

```
  CONNECTOR HEADER
    api_surface     configuration | log_stream | hybrid
                    ← the single most important field. Doc 08 §3.1:
                      "supports AWS" is meaningless without this.
    functions       collect | respond
                    ← response is ALWAYS a separate manifest with
                      separate credentials
    auth            preferred method first
    least privilege what the customer must grant
    coverage        which collectors emit coverage windows
                    (i.e. which can safely drive retraction)

  COLLECTOR TABLE
    Collector       stable id used in the manifest
    Pulls           the API family or object type
    Produces        entities · relationships · properties · findings
    Cadence         default; per-instance overridable
    ★               load-bearing — without it the connector
                    contributes nothing unique
```

## Conventions used throughout

```
  ★     load-bearing collector
  ⚠     a gotcha worth reading before implementing
  →     produces
  dep:  hard dependency on another collector, in this connector
        or another one

  CADENCE VOCABULARY
    continuous   stream or push; not part of the banded cycle
    15m          high-value change detection
    1h           identity deltas
    4h           resource and IAM deltas
    12h          full enumeration of slow-changing config
    24h          full enumeration WITH coverage window
    7d           deep or expensive scans, rolling and partitioned
    on-demand    investigation only, never scheduled
```

## Which band each domain runs in

Collectors inherit their band from the dependency rules in `../03-connectors.md §5.2`, not from their domain. But as a rule of thumb:

```
  BAND 1  identity authorities        → domain 02, plus cloud org structure
  BAND 2  platform inventory + grants → domain 01
  BAND 3  workload and supply chain   → domain 03, Kubernetes
  BAND 4  data, AI, network           → domains 05, 06, 07
  BAND 5  overlays                    → domain 04, domain 08
  CONTINUOUS                          → domain 09, domain 10, syslog/flow
```

## The three things that recur

Worth stating once here rather than in every entry.

**1 · Configuration beats logs, by orders of magnitude.**
A firewall's syslog stream is 1.24 TB/day and yields ~40 edges. Its rulebase is 40 MB and yields 14,000. Wherever a connector has both an `api_surface: configuration` and an `api_surface: log_stream` path, the configuration path is worth ~350× more per byte.

**2 · Response is always separate.**
A connector with a respond function carries a **second manifest and second credentials**. A customer must be able to deploy every connector read-only, with zero write capability, and that must be provable. Response collectors are listed but never share a credential with collection.

**3 · Coverage windows decide retraction.**
Only collectors marked as emitting coverage windows can drive tombstoning. A collector that samples, pages without a completion guarantee, or streams cannot ever safely say "this no longer exists" — it can only add.

---

*Index. Domain files follow.*

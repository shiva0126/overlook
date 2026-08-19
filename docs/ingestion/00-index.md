# Overlook — Ingestion

**Version:** 0.1
**Date:** 2026-08-17
**Scope:** How data physically enters the collector. Mechanics only.
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

## Scope

This folder covers **ingestion only** — from a source emitting or being fetched, to a durable record handed to the pipeline. It stops at stage 2.

```
  IN SCOPE
    the four ingress classes and how each works
    the ingest journal
    flow control, backpressure, shedding
    journal replay

  NOT IN SCOPE — and where it lives
    what our sources are worth        ../14-ingestion-and-sources.md
    the ten pipeline stages          ../04-data-flow-to-security-facts.md
    parsing, normalization, resolution  ../engines/03, 04, 05
    collector behaviour and specs    ../collectors/
    connector catalog                ../connectors/
```

## The documents

| # | Doc | Covers |
|---|---|---|
| 01 | [Pull ingestion](01-pull.md) | Connector polling — dispatch, fetch loop, pagination, cursors |
| 02 | [Push ingestion](02-push.md) | Webhooks — signature, replay guard, the ack contract |
| 03 | [Stream ingestion](03-stream.md) | Syslog, NetFlow, IPFIX — framing, templates, aggregation |
| 04 | [Agent ingestion](04-agent.md) | The agent gateway — enrollment, batching, ack-and-prune |
| 05 | [The ingest journal](05-journal.md) | Segments, fsync policy, retention, crash recovery |
| 06 | [Flow control](06-flow-control.md) | Backpressure, priority shedding, disk pressure |
| 07 | [Journal replay](07-replay.md) | The only debugging mechanism the architecture permits |

---

## The organising idea

Four ingress classes, and they differ in exactly one property that determines everything else about them:

```
                    RECOVERABLE?     DURABILITY CONTRACT
  PULL              yes, by cursor   cursor only, no fsync
  PUSH              NO               journal + fsync BEFORE ack
  STREAM            no, but cheap    aggregate first, journal the aggregate
  AGENT             yes, at source   journal + fsync, then ack, agent prunes
```

Everything downstream of stage 2 treats all four identically. Everything upstream treats them completely differently. Getting that boundary wrong is the most common way this kind of pipeline loses data or wastes disk.

## The numbers

```
  2.2 TB/day    arrives
    187 MB/day  journaled
    4.2 MB/day  fsync'd

  89% of arriving volume never touches disk in any form.
```

## The contract every class honours

```
  ✓ nothing is acknowledged before it is durable   (PUSH, AGENT)
  ✓ nothing is processed before it is acknowledged
  ✓ provenance travels with every record
  ✓ no record is silently discarded
  ✓ shedding is by declared priority class, never uniform
  ✓ a coverage window is emitted only on a complete enumeration
  ✓ a stream NEVER emits a coverage window
```

## The recurring example

Meridian Financial, continuing from `../12-end-to-end-deployment-story.md`:

```
  4 firewalls · 6 core switches · 8,500 agents · 3 GitHub orgs
  41 AWS accounts · 2 AD forests · Entra · CrowdStrike · FortiEDR
  Scalefusion · Forcepoint DLP · VMware · Oracle

  COL-DC1  on-prem, Edge L
  COL-CLD  AWS private subnet, Edge M
```

---

*Index. Documents follow.*

# Overlook - AI Security Research Index

**Version:** 0.1
**Date:** 2026-08-18
**Purpose:** entry point for the AI-security research set that compares Overlook with Palo Alto and CrowdStrike, then turns that comparison into a collector plan.

**Companion to:** `../07-competitive-landscape.md`, `../19-collector-industry-comparison-and-plan.md`, `../20-collector-end-to-end-architecture.md`, `../connectors/07-ai-platforms.md`

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## What is in this set

| # | Doc | Covers |
|---|---|---|
| 01 | [Landscape and vendor comparison](01-landscape-and-vendor-comparison.md) | Palo Alto AI Access Security, Prisma AIRS, CrowdStrike AIDR, and adjacent tool patterns |
| 02 | [End-to-end Overlook AI collector](02-end-to-end-overlook-ai-collector.md) | Source classes, collector layout, pipelines, and sequence diagrams |
| 03 | [Product plan](03-product-plan.md) | What Overlook should copy, what it should not copy, and the phased build plan |
| 04 | [Implementation backlog](04-implementation-backlog.md) | Work items mapped to Overlook modules, contracts, dependencies, and acceptance gates |
| 05 | [Executive brief](05-executive-brief.md) | Short product and investment summary for decision-makers |
| 06 | [Mermaid diagram pack](06-mermaid-diagram-pack.md) | Standalone architecture, flow, sequence, and failure diagrams |

---

## Why this exists

The existing collector docs explain the generic collector architecture. The AI-security market adds a second layer of requirements:

- workforce GenAI access control
- runtime AI gateway inspection
- agent and MCP visibility
- model and posture discovery
- prompt/response telemetry with privacy constraints

The market leaders already split this into source classes and policy surfaces. Overlook should do the same, but keep the collector privacy-bounded and source-local.

---

## Reading order

1. Read [Landscape and vendor comparison](01-landscape-and-vendor-comparison.md) to understand where Palo Alto and CrowdStrike are strong.
2. Read [End-to-end Overlook AI collector](02-end-to-end-overlook-ai-collector.md) to see the architecture that fits Overlook's constraints.
3. Read [Product plan](03-product-plan.md) to see the implementation priorities and product adjustments.
4. Use the [Implementation backlog](04-implementation-backlog.md) to turn the plan into engineering work.
5. Use the [Executive brief](05-executive-brief.md) and [Mermaid diagram pack](06-mermaid-diagram-pack.md) for reviews and handoffs.

# Overlook - AI Security Research Index

**Version:** 0.1
**Date:** 2026-08-18
**Purpose:** entry point for the AI-security research set that compares Overlook with Palo Alto and CrowdStrike, then turns that comparison into a collector plan.

**Companion to:** `../07-competitive-landscape.md`, `../19-collector-industry-comparison-and-plan.md`, `../20-collector-end-to-end-architecture.md`, `../connectors/07-ai-platforms.md`

---

## What is in this set

| # | Doc | Covers |
|---|---|---|
| 01 | [Landscape and vendor comparison](01-landscape-and-vendor-comparison.md) | Palo Alto AI Access Security, Prisma AIRS, CrowdStrike AIDR, and adjacent tool patterns |
| 02 | [End-to-end Overlook AI collector](02-end-to-end-overlook-ai-collector.md) | Source classes, collector layout, pipelines, and sequence diagrams |
| 03 | [Product plan](03-product-plan.md) | What Overlook should copy, what it should not copy, and the phased build plan |

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


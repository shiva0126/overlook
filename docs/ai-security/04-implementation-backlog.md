# Overlook AI Security - Implementation Backlog

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---


**Version:** 0.1  
**Date:** 2026-08-18  
**Purpose:** map the AI-security product plan to Overlook modules, contracts, tests, and delivery gates.

**Inputs:** [Product plan](03-product-plan.md), [AI collector architecture](02-end-to-end-overlook-ai-collector.md), [canonical event schema](../15-canonical-event-schema.md), [collector anatomy](../collectors/00-anatomy.md), [engine index](../engines/00-index.md)

## Delivery rule

Build the smallest complete vertical slice first: one source, one durable ingress path, one deterministic parser, one canonical AI observation, one privacy decision, and one signed fact export. Do not add breadth until this slice survives replay, duplicates, malformed input, source silence, and export failure.

## Work breakdown

| ID | Work item | Existing module or contract | Depends on | Done when |
|---|---|---|---|---|
| AI-01 | Add AI source classes and capability flags | `docs/connectors/07-ai-platforms.md`, source manifest | None | Browser, endpoint, application, gateway, agentic, cloud, model, and local-runtime sources are declaratively identifiable |
| AI-02 | Define canonical AI observation fields | `docs/15-canonical-event-schema.md` | AI-01 | Actor, provider, model, agent, tool, data class, action, outcome, timestamps, provenance, and evidence references have typed fields |
| AI-03 | Define privacy and export policy | E14 privacy gate, `docs/engines/11-privacy-gate.md` | AI-02 | Every field has a disposition: export, tokenize, redact, aggregate, or local-only; decisions are explainable |
| AI-04 | Extend source manifests | `docs/autoparser/08-source-manifest-and-parser-registry.md` | AI-01, AI-02 | Manifests declare ingress, parser, cursor, retry, rate budget, sensitivity, and health expectations |
| AI-05 | Implement concurrent source isolation | E15 orchestration, receive/journal/aggregator | AI-04 | One source can be throttled, quarantined, or unavailable without starving another source |
| AI-06 | Persist-before-ack for push and stream | journal, replay, flow control | AI-05 | Acknowledgement occurs only after durable write; crash recovery replays without loss |
| AI-07 | Add AI parser registry entries | E2 fingerprint, E3 parser, autoparser registry | AI-04 | Known vendor payloads select a pinned parser version; unknown formats enter quarantine |
| AI-08 | Add parser drift detection | parser health baseline, quarantine | AI-07 | Missing fields, type changes, volume shifts, and unknown versions create health signals |
| AI-09 | Normalize AI telemetry | E4 normalizer, E5 enrichment | AI-02, AI-07 | Equivalent vendor records produce equivalent canonical observations with provenance preserved |
| AI-10 | Resolve AI entities locally | E6 entity resolution, bounded cache | AI-09 | Users, devices, applications, models, agents, tools, and data assets link without requiring raw prompts |
| AI-11 | Build AI facts and evidence references | E13 fact builder | AI-09, AI-10 | Facts are idempotent, retractable, attributable, and point to bounded local evidence |
| AI-12 | Enforce local privacy gate | E14 privacy gate | AI-03, AI-11 | Raw prompts, secrets, and sensitive responses never leave the collector unless an explicit policy permits them |
| AI-13 | Sign, queue, and transport reduced facts | sign and transport, outbound queue | AI-12 | Export is authenticated, replay-safe, backpressured, and resumable |
| AI-14 | Add AI discovery collectors | cloud/model/agent inventory collectors | AI-04, AI-07 | Registered models, agents, gateways, tools, and posture findings are represented even without runtime traffic |
| AI-15 | Add unmanaged AI discovery | local-runtime and workstation collectors | AI-10, AI-12 | Local MCP servers, IDE assistants, runtime processes, and credential-bearing configs become findings without exporting file contents |
| AI-16 | Add controller health views | controller UI, collector health contracts | AI-05, AI-08, AI-13 | Operators see lag, drops, quarantine, parser drift, privacy decisions, and coverage per source |
| AI-17 | Add replay and golden fixtures | test harness, journal, parser registry | AI-06, AI-07, AI-09 | Every supported source has fixtures for valid, malformed, duplicate, reordered, and drifted records |
| AI-18 | Run capacity and failure validation | deployment ceiling, flow control | AI-05 through AI-13 | Mixed-source load proves bounded memory, fair scheduling, recovery, and no cross-source starvation |

## Recommended vertical slices

### Slice 1: Gateway or application telemetry

Implement `AI-01` through `AI-13` for one JSON source. This validates the full privacy boundary and should be the first production-shaped slice.

### Slice 2: Agentic and MCP telemetry

Reuse the same pipeline for agent, tool, server, and authorization facts. Add explicit tool-call, server-identity, credential-use, and approval outcomes without changing the transport contract.

### Slice 3: Discovery and posture

Add pull-based inventory and posture collectors. These should emit coverage windows and inventory facts, not pretend that absence of runtime traffic means absence of AI.

### Slice 4: Unmanaged AI

Add workstation-local discovery only after privacy and evidence handling are stable. The collector should export metadata and risk facts, never arbitrary configuration contents.

## Non-functional acceptance gates

- **Durability:** no acknowledged record is lost across process restart; replay is idempotent.
- **Isolation:** a high-volume stream cannot consume another source's journal, parser, or outbound budget.
- **Privacy:** export tests prove that prompt text, response text, tokens, and credentials are local-only by default.
- **Explainability:** every fact identifies source, parser version, observation time, collection time, and privacy disposition.
- **Drift:** schema or volume changes are visible as health state, not silent parse failure.
- **Backpressure:** queues expose age, depth, rejection, and quarantine metrics; the system degrades by policy.
- **Scale:** mixed-source tests remain within the documented collector ceiling and scale out when exceeded.
- **Security:** credentials are scoped, short-lived, never stored in event payloads, and excluded from evidence references.

## Explicit deferrals

- Inline blocking or proxying of every AI request.
- Raw prompt or response search in SaaS.
- LLM-dependent source classification or parser selection.
- Full graph ownership inside the edge collector.
- Automatic remediation based only on an unverified AI observation.


# Overlook - Collector Industry Comparison and Plan

**Version:** 0.1
**Date:** 2026-08-18
**Purpose:** compare the Overlook collector plan with industry-standard products, identify where they are stronger, and turn that into a build plan for the collector.

**Companion to:** `07-competitive-landscape.md`, `10-appliance-stack-and-engines.md`, `14-ingestion-and-sources.md`, `collectors/00-anatomy.md`, `autoparser/00-index.md`

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Executive summary

The collector work should not try to out-feature established vendors.
It should absorb the parts they already do well and keep Overlook's unique advantages intact.

What the industry is doing better today:

- **Stellar Cyber** is better at packaging source-side collection as a deployable, resource-bounded appliance with explicit modular feature controls.
- **Google SecOps** is better at parser operationalization, default parser coverage, and self-service field-to-schema mapping.
- **Cribl** is better at source routing, ordered transformation pipelines, and fan-out without turning every step into a monolith.
- **Microsoft Defender for Cloud** is better at integrating attack-path analysis into a cloud security graph with remediation context.

What Overlook must do better:

- preserve the privacy boundary
- unify many source classes in one collector
- keep the collector source-local and budget-aware
- reduce raw vendor records into canonical events and facts
- treat parser drift and quarantine as first-class operations
- avoid an LLM hot path in ingest

The plan below turns those observations into work items.

---

## 2. What each vendor does better, and why

| Vendor | They do better | How they do it | Overlook response |
|---|---|---|---|
| Stellar Cyber | Source-side appliance ergonomics | Modular sensors can enable only the features needed, and sensor/data-processor placement is explicit | Build manifest-driven collector profiles with hard budgets and placement flags |
| Google SecOps | Parser operations at scale | Default parsers, parser extensions, UDM mapping, and a documented parser workflow | Build a source manifest + parser registry + quarantine flow, with deterministic parser versioning |
| Cribl | Routing and transformation control | Routes select a pipeline first; pipelines run ordered functions; outputs can be cloned and cascaded | Keep source router separate from parsing and normalization; use per-source queues and ordered stages |
| Microsoft Defender for Cloud | Graph-backed attack-path remediation | Cloud security graph, attack-path analysis, and remediation steps are tightly linked | Keep collector scope limited to local facts; let SaaS do graph analysis and remediation context |

---

## 3. What the collector should copy

### 3.1 From Stellar Cyber

Stellar Cyber's strength is not one magical parser.
It is the operational shape of the appliance:

- modular feature enablement
- explicit sizing constraints
- separation of collection roles
- predictable deployment profiles

The useful lesson is that a collector should know what it is allowed to do before it starts running.

**Collector implication**

- every source gets a manifest
- every source gets a budget
- every source has a recoverability class
- the collector can disable a source cleanly if it exceeds its resource contract

### 3.2 From Google SecOps

Google SecOps is strong because parsing is a product surface, not a hidden implementation detail.

It has:

- maintained default parsers
- customer-specific parser extensions
- a UDM normalization target
- self-service mapping tools
- a clear expectation that raw logs are transformed into a canonical model

**Collector implication**

- the parser registry must be explicit and versioned
- parser updates must be reviewable and deterministic
- a parser miss must go to quarantine, not disappear silently
- a local assistant may propose parser changes, but the hot path must remain deterministic

### 3.3 From Cribl

Cribl's strong pattern is routing before transformation.

Routes decide where an event should go, and pipelines perform ordered functions.

That separation prevents the ingest layer from becoming an unmaintainable knot.

**Collector implication**

- route first, parse second, normalize third
- isolate source classes early
- allow fan-out to quarantine, metrics, and evidence storage without duplicating parsing logic

### 3.4 From Microsoft Defender for Cloud

Microsoft's advantage is the tight loop between graph view and remediation.

It is strong at showing attack paths and then linking them to fixes.

**Collector implication**

- the collector should not try to compute the full graph
- the collector should emit high-quality canonical events and facts
- graph correlation and remediation live downstream in SaaS

---

## 4. What Overlook should not copy

- a giant event lake on the collector
- a message-broker dependency in the customer environment
- a model-in-the-hot-path parser
- silent fallback behavior when parsing fails
- one shared queue for every source type
- a collector that cannot explain what left the environment

The Overlook collector only works if the privacy boundary stays obvious and auditable.

---

## 5. Overlook collector plan

### Phase 0 - Freeze the contracts

Deliverables:

- source manifest schema
- parser registry schema
- canonical event schema
- source budgets and recoverability classes
- quarantine and replay semantics

Acceptance criteria:

- every source can be described declaratively
- every parser can be versioned
- every source has a failure mode
- every source has an ownership and health state

### Phase 1 - Build the ingress backbone

Deliverables:

- source adapters for pull, push, stream, and agent
- source router
- raw journal
- replay mechanism
- per-source queues
- backpressure and rate budgets

Acceptance criteria:

- push sources are journaled before ack
- pull sources can refetch from cursor
- stream sources aggregate on receive
- one noisy source cannot starve the others

### Phase 2 - Build the parser system

Deliverables:

- parser registry
- generic parsers for syslog, key-value, JSON, CSV
- source-specific parsers for the first priority vendors
- quarantine flow
- parser drift detection

Acceptance criteria:

- parser selection is deterministic
- fallback is explicit
- unknown formats land in quarantine
- parser versions are auditable

### Phase 3 - Build the canonical event path

Deliverables:

- canonical event v1
- normalization layer
- entity resolver hooks
- local analytics inputs
- evidence references

Acceptance criteria:

- vendor syntax disappears after normalization
- source provenance remains visible
- sensitive fields are tagged for privacy control
- events can be merged into facts without raw replay dependence

### Phase 4 - Build fact reduction and privacy gating

Deliverables:

- fact builder
- dedupe and merge
- privacy gate
- tokenization / redaction policy
- outbound queue and signing

Acceptance criteria:

- only minimal facts leave the environment
- raw payloads stay local
- repeated observations collapse into stable facts
- outbound batches are signed and replay-safe

### Phase 5 - Build operational control

Deliverables:

- source health dashboard
- freshness / backlog / coverage states
- parser miss reporting
- per-source budget reporting
- local controller actions for enable, disable, quarantine, replay

Acceptance criteria:

- operators can tell what is healthy, stale, or blocked
- the collector makes resource pressure visible
- the customer can manage the collector without SaaS

### Phase 6 - Expand to the long tail

Deliverables:

- long-tail vendor playbooks
- additional parser families
- AI gateway and agent enrichment
- source-specific heuristics only where the manifest demands them

Acceptance criteria:

- new sources can be onboarded without rewriting the collector core
- long-tail parsing does not destabilize the main path

For the full end-to-end flow and diagrams, see [Collector End-to-End Architecture](20-collector-end-to-end-architecture.md).

---

## 6. Collector architecture decisions implied by the comparison

1. Source routing must happen before heavy parsing.
2. Every source must have a manifest.
3. Parsing must be versioned, deterministic, and quarantine-aware.
4. Stream telemetry must be aggregated on receive.
5. Pull sources should prefer cursor-based refetch over durable hoarding.
6. The collector must expose explicit budgets and backpressure.
7. The collector must reduce to canonical events and facts before egress.
8. SaaS owns graph correlation, attack paths, and remediation context.

---

## 7. Recommended first build slice

If the goal is the fastest production-relevant collector slice, build this first:

1. FortiGate and FortiManager config plus syslog
2. Active Directory / LDAP sweeps
3. Cloud IAM configuration
4. FortiEDR endpoint reports
5. Generic syslog and JSON fallback

Why this slice:

- it exercises all ingress classes
- it includes high-value config and high-volume stream sources
- it covers the identity closure that powers the graph
- it stresses parser versioning and quarantine
- it gives an immediate comparison point against the industry products above

---

## 8. Practical success metrics

The collector should be judged on these metrics:

- source registration success rate
- parser hit rate
- parser drift rate
- quarantine rate
- backlog depth
- freshness lag
- coverage completeness
- replay success rate
- source isolation under overload
- number of facts emitted per raw MB

If these are not measured, the collector will drift back toward a log lake.

---

## 9. References

### Stellar Cyber

- https://docs.stellarcyber.ai/6.6.xs/Common/Stellar-Architecture.htm
- https://docs.stellarcyber.ai/prod-docs/6.5.xs/Common/Stellar-Architecture.htm?TocPath=GETTING+STARTED%7C_____3

### Google SecOps

- https://docs.cloud.google.com/chronicle/docs/event-processing/using-parser-extensions
- https://docs.cloud.google.com/chronicle/docs/event-processing/parser-extension-examples
- https://docs.cloud.google.com/chronicle/docs/ingestion/parser-list/supported-default-parsers
- https://docs.cloud.google.com/chronicle/docs/event-processing/parsing-overview
- https://docs.cloud.google.com/chronicle/docs/event-processing/self-service-parser-options

### Cribl

- https://docs.cribl.io/stream/routes
- https://docs.cribl.io/stream/pipelines
- https://docs.cribl.io/stream/tour/

### Microsoft Defender for Cloud

- https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-attack-path
- https://learn.microsoft.com/en-us/azure/defender-for-cloud/how-to-manage-attack-path
- https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction

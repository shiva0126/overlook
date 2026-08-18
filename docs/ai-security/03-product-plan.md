# Overlook - AI Security Product Plan

**Version:** 0.1
**Date:** 2026-08-18
**Purpose:** convert the AI-security comparison into a concrete product plan for Overlook.

**Companion to:** `01-landscape-and-vendor-comparison.md`, `02-end-to-end-overlook-ai-collector.md`, `../19-collector-industry-comparison-and-plan.md`

---

## 1. Product position

Overlook should not compete as "another AI firewall."
That is the wrong category for the product.

The better position is:

- a privacy-bounded AI collector
- a canonical reduction layer for AI telemetry
- a trust-graph input for identity, data, network, model, and agent relationships
- a system that can see unmanaged AI, not just registered AI

That position keeps Overlook aligned with its architecture.

---

## 2. What to copy from Palo Alto

Palo Alto's useful ideas are architectural, not cosmetic.

Copy these:

- lifecycle thinking: access security, runtime security, posture, model security, discovery
- AI gateway as a distinct control point
- agent discovery as a first-class feature
- MCP support as a real attack surface
- posture plus runtime linkage

Adapt them to Overlook by turning each one into source classes and facts instead of inline enforcement surfaces.

---

## 3. What to copy from CrowdStrike

CrowdStrike's useful ideas are operational.

Copy these:

- collector taxonomy
- policy types mapped to collector types
- prompt, response, and metadata treated as first-class telemetry
- browser, endpoint, application, gateway, agentic, cloud, and OTel coverage
- a single view that correlates AI activity with the rest of the security stack

Adapt them to Overlook by keeping the collector local and the export reduced.

---

## 4. What not to copy

Do not copy these assumptions:

- that the product owns the whole inline traffic path
- that raw prompt content should be freely exported
- that every AI source will appear in a vendor registry
- that a hosted LLM belongs in the ingest path
- that one gateway covers all AI use cases

Those assumptions are valid for some AI-security vendors and wrong for Overlook.

---

## 5. The gaps Overlook should close

### 5.1 Unmanaged AI

This is the most important gap.

Overlook should detect and model:

- developer-laptop MCP servers
- local model runtimes
- IDE assistant extensions
- ad hoc agent frameworks
- credentials stored in local configuration files

This is the layer most platforms still miss because they start from a registry.

### 5.2 Cross-domain AI facts

Overlook should link AI telemetry back to:

- identity
- cloud IAM
- data sensitivity
- network reachability
- repository and CI/CD access

That is where the trust graph becomes useful.

### 5.3 Privacy-bounded reduction

The collector should reduce AI telemetry locally into facts.

That means:

- raw payloads stay local
- only necessary fields leave the environment
- sensitive prompt text is redacted or tokenized before export
- large evidence stays attached as local references

### 5.4 Deterministic parser behavior

The AI parser path must be deterministic.

No LLM should be required for:

- source classification
- parser selection
- canonical field selection
- export decisions

An LLM can assist offline or locally, but not as the ingest dependency.

---

## 6. Recommended product phases

### Phase 0 - Define the AI contracts

Deliverables:

- AI source-class taxonomy
- canonical AI event schema
- evidence reference schema
- privacy-gate rules
- AI source manifest extensions

Acceptance criteria:

- every AI source can be described declaratively
- every source has a provenance contract
- every export decision is explainable

### Phase 1 - Build the AI source router

Deliverables:

- browser, endpoint, application, gateway, agentic, cloud, model, and local-runtime routing
- per-source queues
- recovery and replay
- health states and backpressure

Acceptance criteria:

- one noisy source cannot starve the others
- source failures are isolated

### Phase 2 - Build the AI parser registry

Deliverables:

- deterministic parser registry
- schema and field typing for AI telemetry
- quarantine for unknown or drifting formats
- source-specific parser versions

Acceptance criteria:

- parser drift is visible
- parser misses do not silently vanish
- parser changes are reviewable

### Phase 3 - Build AI fact reduction

Deliverables:

- canonical event normalization
- entity resolution
- evidence linking
- local dedupe and merge
- privacy gate

Acceptance criteria:

- raw telemetry is reduced before egress
- facts remain attributable to a source
- sensitive data is protected locally

### Phase 4 - Add AI discovery and posture ingestion

Deliverables:

- cloud AI control-plane collectors
- model inventory collectors
- agent inventory collectors
- posture and red-team scan ingestion

Acceptance criteria:

- Overlook can describe what AI exists even when it is not actively talking

### Phase 5 - Add unmanaged AI coverage

Deliverables:

- local MCP discovery
- developer workstation config discovery
- local runtime discovery
- IDE assistant inventory

Acceptance criteria:

- the product sees AI that has no central registry

---

## 7. Product decisions that should stay fixed

1. The collector is source-local and budget-aware.
2. The ingest path stays deterministic.
3. The privacy boundary is enforced before egress.
4. The output is facts, not raw telemetry dumps.
5. The AI layer is one domain of the trust graph, not the whole product.

If any of these change, the product should revisit the entire architecture.

---

## 8. Success criteria

The AI-security part of Overlook is working when:

- multiple AI source classes can be active at once
- the collector can explain what it saw and why it exported it
- unmanaged AI appears as a first-class finding
- model, agent, data, identity, and network context are linked
- the SaaS receives reduced facts instead of raw conversations

---

## 9. Implementation order

1. Finalize contracts and schema
2. Build routing and durability
3. Build parser registry and quarantine
4. Build canonical event normalization
5. Build local privacy gate and fact builder
6. Add discovery and posture collectors
7. Expand to unmanaged AI

That is the order that preserves the Overlook boundary and avoids rework.


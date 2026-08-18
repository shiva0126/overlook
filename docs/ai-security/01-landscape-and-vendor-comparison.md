# Overlook - AI Security Landscape and Vendor Comparison

**Version:** 0.1
**Date:** 2026-08-18
**Purpose:** compare the AI-security products that matter for Overlook, with focus on Palo Alto and CrowdStrike, and extract the product lessons that should shape the collector.

**Companion to:** `../07-competitive-landscape.md`, `../19-collector-industry-comparison-and-plan.md`, `../connectors/07-ai-platforms.md`

---

> **Scope note**
> This document is about AI security controls and telemetry collection. It does not repeat the generic collector architecture already covered in `../20-collector-end-to-end-architecture.md`.

---

## 1. What the market is doing

The AI-security market is splitting into four practical product shapes:

1. **Workforce AI control**
   - controls employee use of public GenAI tools
   - focuses on prompts, responses, data loss, and browser coverage

2. **AI runtime security**
   - inspects traffic between applications, models, agents, and gateways
   - focuses on prompt injection, data leakage, and misuse of tools

3. **AI posture and discovery**
   - inventories agents, models, datasets, and AI assets
   - focuses on what exists, where it runs, and what it can reach

4. **AI collector and telemetry fabric**
   - captures the AI workflow at browser, endpoint, app, gateway, cloud, and agent layers
   - focuses on normalized telemetry and policy routing

Palo Alto leans hard into 2 and 3, with 1 as an access-security entry point.
CrowdStrike leans into 1, 2, and the collector fabric itself.
Overlook should learn from both, but stay collector-first and privacy-bounded.

---

## 2. Palo Alto Networks

### 2.1 AI Access Security

Palo Alto positions AI Access Security as the workforce GenAI control plane.
It is aimed at employee use of public AI services and is deployed through the Palo Alto network/browser stack.

What it emphasizes:

- safe adoption of GenAI by employees
- mitigation of inadvertent prompt data leakage
- malicious content in responses
- visibility and policy enforcement across NGFW, Prisma Access, and Prisma Browser
- DLP integration and logging

What that means architecturally:

- the product is not just a log viewer
- it is an access control and policy enforcement surface
- the policy layer is tied to network and browser controls

### 2.2 Prisma AIRS

Prisma AIRS is Palo Alto's broader AI-runtime platform.
Official docs describe it as covering:

- AI applications
- AI models
- AI data protection
- AI agents

The platform also includes:

- AI Runtime Firewall
- AI Runtime API / API intercept
- AI Model Security
- AI Red Teaming
- posture management
- AI Gateway
- MCP visibility and protection

What this means in practice:

- Palo Alto is treating AI security as a full lifecycle problem
- it separates discovery, runtime inspection, posture, and model security
- it is comfortable with both network-level and application-level enforcement

### 2.3 AI agent discovery and model security

Palo Alto's agent-discovery story is strong because it starts with inventory.
The product explicitly tracks enterprise and SaaS agents built on cloud platforms such as AWS Bedrock and Azure AI Foundry/OpenAI.

Its model-security story is also important:

- models are treated as security-scannable assets
- assessments are automated
- audit trails exist for reviews

That is a significant difference from products that only inspect prompts in motion.

---

## 3. CrowdStrike AIDR

### 3.1 The main product idea

CrowdStrike AIDR is built around prompt-layer visibility and protection.
It focuses on collecting AI activity across:

- browser
- endpoint
- application
- gateway
- agentic / MCP
- cloud
- OpenTelemetry

That collector taxonomy is one of the strongest things in the market.

### 3.2 What CrowdStrike does especially well

CrowdStrike is unusually explicit about the telemetry model:

- each collector category has a clear job
- policy types match collector categories
- the console can correlate prompts, responses, metadata, and detections
- actions include logging, redaction, blocking, and transformation
- telemetry can be forwarded into Falcon Next-Gen SIEM

That is valuable because it turns "AI security" into a control surface rather than a marketing label.

### 3.3 The collector model

The collector approach is more important than the policy wording.
CrowdStrike has already shown that AI security needs multiple collection paths:

- browser extension for managed web usage
- endpoint sensor for desktop apps and coding assistants
- SDK/API for application integration
- gateway policy for API gateways
- agentic MCP proxy for tool traffic
- OTel for general telemetry forwarders

That is the closest market proof that Overlook should not define AI security as one connector.

---

## 4. Adjacent tools that matter

These are not the primary AI-security comparators, but they show the implementation patterns Overlook should steal.

| Product | What it does better | Lesson for Overlook |
|---|---|---|
| Google SecOps | Parser operations, UDM mapping, entity context | Parsing must be a product surface, not hidden code |
| Stellar Cyber | Appliance ergonomics and feature gating | Collector budgets and feature flags must be explicit |
| Cribl | Routing before transformation | Separate source routing from parsing and normalization |
| Microsoft Defender for Cloud | Graph-backed attack-path remediation | Keep the graph and remediation downstream from ingest |

For broader AI-security market context, see `../07-competitive-landscape.md`.

---

## 5. What each vendor is actually better at

### Palo Alto is better at

- full-lifecycle AI coverage
- posture plus runtime plus model security
- discovery of agents and AI assets
- network and browser policy enforcement
- a clear AI gateway concept

### CrowdStrike is better at

- collector taxonomy
- prompt-layer telemetry unification
- policy-to-collector alignment
- practical deployment across browser, endpoint, app, gateway, agentic, and cloud
- SIEM adjacency for correlation

### Overlook should be better at

- privacy-bounded collection
- local reduction before egress
- source manifest and parser registry discipline
- unmanaged AI coverage
- combining AI telemetry with the rest of the trust graph

---

## 6. The comparison that matters for Overlook

| Dimension | Palo Alto | CrowdStrike | Overlook target |
|---|---|---|---|
| Primary focus | AI security platform lifecycle | Prompt-layer telemetry and control | Source-local AI collector that feeds the trust graph |
| Enforcement style | Network, browser, runtime, gateway | Policy at collector and gateway points | Privacy gate, redaction, and fact reduction before egress |
| Collectors | AI gateway, runtime, discovery, model security | Browser, endpoint, app, gateway, agentic, cloud, OTel | Browser, endpoint, app, gateway, agentic, cloud, model, posture, local runtime |
| Discovery | Strong agent and model discovery | Strong telemetry and usage visibility | Strong unmanaged AI discovery plus identity and reachability |
| MCP support | Strong runtime and tool protection | Explicit agentic MCP proxy | MCP support without trusting the registry alone |
| Data model | Security platform logs and findings | Prompt/response telemetry plus detections | Canonical event plus evidence refs plus security facts |

---

## 7. What Overlook should borrow, adapt, and reject

### Borrow

- Palo Alto's split between access security, runtime security, model security, and posture
- Palo Alto's agent discovery and MCP awareness
- CrowdStrike's collector taxonomy
- CrowdStrike's policy alignment with collector categories
- CrowdStrike's attention to prompt, response, and metadata as first-class telemetry

### Adapt

- turn "policy" into a local privacy-gate and export policy, not a full inline AI firewall
- turn "collector categories" into source classes and manifests
- turn "runtime protection" into local fact extraction and risk signaling

### Reject

- a design that depends on a hosted LLM in the ingest path
- a design that assumes every AI source is registered in a vendor console
- a design that treats prompts as safe to export in raw form
- a monolithic gateway that tries to own every AI traffic path

---

## 8. Bottom line

Palo Alto is strongest when the problem is framed as a full AI security lifecycle.
CrowdStrike is strongest when the problem is framed as prompt-layer telemetry across many collection points.

Overlook should not copy either product directly.
It should combine the useful parts:

- Palo Alto's lifecycle thinking
- CrowdStrike's collector taxonomy
- Overlook's privacy boundary and cross-domain trust graph

That combination is what justifies the collector design.

---

## 9. Official sources

- [Palo Alto Prisma AIRS overview](https://docs.paloaltonetworks.com/ai-runtime-security/administration/prisma-airs-overview)
- [Palo Alto AI Access Security introduction](https://docs.paloaltonetworks.com/ai-access-security/getting-started/introducing-ai-access-security)
- [Palo Alto AI Gateway configuration](https://docs.paloaltonetworks.com/ai-runtime-security/administration/configure-ai-gateway)
- [Palo Alto AI Agent Discovery](https://docs.paloaltonetworks.com/ai-runtime-security/administration/agent-discovery)
- [Palo Alto API Intercept overview](https://docs.paloaltonetworks.com/ai-runtime-security/activation-and-onboarding/ai-runtime-security-api-intercept-overview)
- [Palo Alto MCP server for centralized AI agent security](https://docs.paloaltonetworks.com/ai-runtime-security/activation-and-onboarding/prisma-airs-mcp-server-for-centralized-ai-agent-security)
- [CrowdStrike AIDR overview](https://aidr-docs.crowdstrike.com/docs/aidr)
- [CrowdStrike collector types](https://aidr-docs.crowdstrike.com/docs/aidr/collectors)
- [CrowdStrike policies](https://aidr-docs.crowdstrike.com/docs/aidr/policies)
- [CrowdStrike browser collector](https://aidr-docs.crowdstrike.com/docs/aidr/collectors/browser)
- [CrowdStrike gateway collector](https://aidr-docs.crowdstrike.com/docs/aidr/collectors/gateway)
- [CrowdStrike agentic collector](https://aidr-docs.crowdstrike.com/docs/aidr/collectors/agentic)
- [CrowdStrike OTel collector](https://aidr-docs.crowdstrike.com/docs/aidr/collectors/otel)


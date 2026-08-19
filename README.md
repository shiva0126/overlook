# Overlook

A privacy-preserving cyber exposure intelligence platform.

Overlook correlates cloud, application, data, identity, network, runtime, and AI
relationships into a unified TrustGraph to identify how attackers — or AI agents —
can ultimately reach and control critical business assets.

**Core design principle:** sensitive customer telemetry is processed inside the
customer's environment. Overlook SaaS receives only the minimum Security Facts,
graph relationships, risk metadata, and evidence references required for
correlation and attack-path analysis.

## Components

| Component | Role |
|---|---|
| Overlook Edge Collector | Customer-side ingestion, parsing, normalization, enrichment, Security Fact generation, privacy minimization, forwarding |
| Overlook Agent | Host runtime context, local AI/MCP discovery, host response actions |
| Overlook AI Gateway | Prompt, tool-call, MCP and RAG inspection for AI traffic |
| Overlook SaaS / TrustGraph | Cross-domain correlation, attack paths, risk, investigation, response orchestration |

## Status

Early architecture and design. Nothing implemented yet.

## Repository layout

```
docs/
  LLD-edge-collector-v1.0.md   ← THE IMPLEMENTATION BOUNDARY
  edge-collector/       ← component by component (start here)
  engines/              engine-level detail
  connectors/           the connector catalog and framework
  collectors/           per-source collection mechanics
  ingestion/            ingress mechanics
  autoparser/           the auto-parser, L0–L5
  analytics/            SaaS-side analytics and intelligence
```

## Where to start

`docs/LLD-edge-collector-v1.0.md` is the Low Level Design and the
implementation boundary. It supersedes the earlier Engineering Handoff for
collector internals.

`docs/edge-collector/` explains each component underneath it — inputs, outputs,
mechanics, resource budget and failure modes — and records the five open
escalations against the LLD in
[`13-escalations.md`](docs/edge-collector/13-escalations.md).

The rest of `docs/` is the reasoning underneath both.

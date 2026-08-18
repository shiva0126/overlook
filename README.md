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
| Overlook Edge Analytics Node | Customer-side ingestion, normalization, entity resolution, local analytics, Security Fact generation |
| Overlook Agent | Host runtime context, local AI/MCP discovery, host response actions |
| Overlook AI Gateway | Prompt, tool-call, MCP and RAG inspection for AI traffic |
| Overlook SaaS / TrustGraph | Cross-domain correlation, attack paths, risk, investigation, response orchestration |

## Status

Early architecture and design. Nothing implemented yet.

## Repository layout

```
docs/                   architecture and design documents
  edge-collector/       ← the collector, service by service (start here)
  engines/              engine-level detail
  connectors/           the connector catalog and framework
  collectors/           per-source collection mechanics
  ingestion/            ingress mechanics
  autoparser/           the auto-parser, L0–L5
  analytics/            SaaS-side analytics and intelligence
```

## Where to start

`docs/edge-collector/` specifies the seven services of the Edge Collector as
handed over — inputs, outputs, mechanics, resource budget and failure modes for
each. It is the buildable layer; the rest of `docs/` is the reasoning underneath
it.

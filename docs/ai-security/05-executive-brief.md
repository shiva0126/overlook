# Overlook AI Security - Executive Brief

**Version:** 0.1  
**Date:** 2026-08-18

## The decision

Overlook should build an AI-security collector and evidence-reduction layer, not attempt to become a universal inline AI firewall. The collector should accept telemetry from many vendors at the same time, understand each source locally, and export only privacy-approved security facts into the existing Overlook trust graph.

## Why this matters

AI activity is distributed across browsers, endpoints, applications, gateways, cloud control planes, model platforms, agents, MCP servers, and developer workstations. Palo Alto demonstrates the value of lifecycle coverage and runtime control. CrowdStrike demonstrates the value of broad collector coverage and operational policy alignment. Overlook can differentiate by combining that breadth with source-local privacy and visibility into unmanaged AI.

## Target operating model

```text
Many vendor sources
        |
        v
Source-aware edge collector
  journal -> route -> parse -> normalize -> resolve
        |
        v
Local facts and evidence references
  privacy gate -> sign -> queue -> export
        |
        v
Overlook SaaS trust graph and risk paths
```

## Product advantages to preserve

- Raw AI content stays local by default.
- Parser and export behavior is deterministic and reviewable.
- Multiple sources run concurrently with isolated budgets and recovery.
- AI facts connect to identity, data, cloud, network, repository, and CI/CD context.
- Unmanaged MCP servers, local runtimes, IDE assistants, and credential-bearing configurations are visible as a distinct risk surface.

## Investment sequence

1. Lock the AI schema, source manifests, and privacy contract.
2. Prove a durable end-to-end path with one gateway or application source.
3. Add agentic/MCP telemetry using the same contracts.
4. Add cloud, model, agent, and posture discovery.
5. Add unmanaged workstation and local-runtime discovery.

## Measures of success

- Concurrent mixed-source ingestion with no cross-source starvation.
- Zero acknowledged-event loss across restart and replay.
- No raw prompts, responses, secrets, or tokens exported by default.
- Every exported fact is attributable, deduplicated, and explainable.
- AI observations materially improve existing Overlook identity, exposure, and blast-radius analysis.

## Main risks

- Privacy failure from exporting raw content or credentials.
- Silent parser drift from vendor format changes.
- Overbuilding inline enforcement before the collector contracts are stable.
- Treating registry inventory as complete and missing unmanaged AI.
- Resource contention when high-volume streams and API collectors share one appliance.

## Positioning

Overlook's AI-security promise should be: **discover AI everywhere, reduce it safely at the edge, and explain its impact in the existing trust graph.**


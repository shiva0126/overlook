# Overlook AI Security - Mermaid Diagram Pack

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
**Purpose:** reusable diagrams for architecture reviews, implementation tickets, and operator documentation.

## System context

```mermaid
flowchart LR
    V[Vendor connectors] --> I[Ingress adapters]
    I --> C[Overlook edge collector]
    C --> F[Canonical security facts]
    F --> S[Overlook SaaS]
    S --> G[Trust graph]
    G --> R[Risk paths and investigations]
```

## Collector layout

```mermaid
flowchart TD
    subgraph Sources
        B[Browser]
        E[Endpoint]
        A[Application]
        W[Gateway]
        M[Agent / MCP]
        D[Cloud and model APIs]
        L[Local runtime and workstation]
    end
    subgraph Edge[Overlook edge collector]
        X[Adapters]
        Q[Source router and budgets]
        J[Durable journal]
        P[Parser registry]
        N[Normalizer and enrichment]
        R[Local entity resolution]
        T[Fact builder]
        G[Privacy gate]
        O[Signed outbound queue]
    end
    B --> X
    E --> X
    A --> X
    W --> X
    M --> X
    D --> X
    L --> X
    X --> Q --> J --> P --> N --> R --> T --> G --> O
```

## Multi-source concurrency and isolation

```mermaid
flowchart LR
    F[Fortinet firewall] --> RF[Firewall queue]
    E[FortiEDR] --> RE[EDR queue]
    A[AI gateway] --> RA[AI queue]
    M[MCP discovery] --> RM[Discovery queue]
    RF --> PF[Firewall parser]
    RE --> PE[EDR parser]
    RA --> PA[AI parser]
    RM --> PM[Discovery parser]
    PF --> Z[Shared canonical fact path]
    PE --> Z
    PA --> Z
    PM --> Z
    RF -. budget and backpressure .- CF[Firewall control]
    RE -. budget and backpressure .- CE[EDR control]
    RA -. budget and backpressure .- CA[AI control]
    RM -. budget and backpressure .- CM[Discovery control]
```

## Runtime event sequence

```mermaid
sequenceDiagram
    participant Source
    participant Adapter
    participant Journal
    participant Parser
    participant Facts
    participant Privacy
    participant SaaS
    Source->>Adapter: Send or expose event
    Adapter->>Journal: Write provenance envelope
    Journal-->>Adapter: Durable offset
    Adapter-->>Source: Acknowledge when required
    Journal->>Parser: Read source-scoped record
    Parser->>Facts: Emit canonical observation
    Facts->>Privacy: Request export decision
    Privacy-->>Facts: Redact, tokenize, aggregate, or local-only
    Facts->>SaaS: Sign and enqueue reduced fact
    SaaS-->>Facts: Acknowledge fact
    Facts->>Journal: Commit outbound checkpoint
```

## Pull collector sequence

```mermaid
sequenceDiagram
    participant Scheduler
    participant Collector
    participant API
    participant Journal
    participant SaaS
    Scheduler->>Collector: Lease scope and cursor
    Collector->>API: Preflight permissions
    API-->>Collector: Reachability and scope
    loop Each page
        Collector->>API: Fetch page with rate budget
        API-->>Collector: Page and continuation cursor
        Collector->>Journal: Durable raw page envelope
    end
    Collector->>Journal: Commit cursor after durable write
    Collector->>Journal: Emit complete coverage window
    Journal->>SaaS: Export reduced inventory facts
```

## Failure and recovery

```mermaid
stateDiagram-v2
    [*] --> Healthy
    Healthy --> Degraded: lag, 429s, partial parse
    Degraded --> Healthy: recovery window passes
    Degraded --> Quarantined: unknown format or unsafe payload
    Healthy --> Quarantined: parser or privacy violation
    Degraded --> Failed: repeated transport or auth errors
    Failed --> Recovering: retry policy permits
    Recovering --> Healthy: replay and health checks pass
    Recovering --> Failed: retry exhausted
    Quarantined --> Recovering: operator or signed parser update
```

## Privacy decision path

```mermaid
flowchart TD
    R[Raw observation] --> C{Contains sensitive content?}
    C -->|No| E[Export canonical fact]
    C -->|Prompt or response| H[Hash or tokenize identifiers]
    C -->|Credential or secret| X[Remove and retain local evidence reference]
    C -->|Personal or regulated data| D[Redact or aggregate]
    H --> E
    D --> E
    X --> L[Local-only evidence]
    E --> S[Sign and send]
```

## Discovery-to-risk path

```mermaid
flowchart LR
    I[AI inventory] --> A[Agent and model entities]
    R[Runtime observations] --> A
    M[MCP tools and servers] --> A
    U[Identity and IAM] --> C[Entity correlation]
    D[Data sensitivity] --> C
    N[Network and reachability] --> C
    A --> C --> P[Trust graph path]
    P --> F[Finding and remediation context]
```


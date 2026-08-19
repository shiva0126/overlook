# Overlook — Connectors: Inventory, Framework and Orchestration

**Version:** 0.1 (design draft)
**Date:** 2026-08-12
**Companion to:** `01-system-design.md`, `02-iam-deep-dive.md`
**Status:** For brainstorming. Nothing here is implemented.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## The two questions this document answers

1. **How many connectors do we actually need, and what are they?**
2. **How do we build and run all of them as one thing, in one go?**

There is no phased connector roadmap in this document. Connectors are not delivered in waves over quarters — they are a **fleet**, built against one framework and orchestrated by one control loop. The only sequencing that exists anywhere in this design is *within a single run cycle*, where identity sources must resolve before the sources that depend on them. That is dependency ordering measured in minutes, not a release plan measured in quarters.

---

## 1. What counts as a connector

Without a strict definition the number is meaningless. Two candidate units:

```
  WRONG UNIT — "one integration per API"
     AWS would be 28 connectors (IAM, EC2, S3, RDS, Lambda, KMS, ...)
     Inflates the count, hides the real work, breaks credential modelling.

  RIGHT UNIT — "one authenticated source system"
     AWS is ONE connector with 28 COLLECTORS inside it.
     One credential, one rate-limit domain, one health state,
     one coverage window, one failure blast radius.
```

**Definition used throughout:**

> A **connector** is one authenticated integration with one source system: one credential set, one rate-limit domain, one health state, one manifest.
>
> A **collector** is one data-gathering routine inside a connector: one API family or object type, producing one class of entities or relationships.

So the counts below are given as **connectors (collectors)**. The connector count drives credential management, customer onboarding effort, and the health model. The collector count drives engineering effort.

---

## 2. The inventory

### 2.1 Cloud and infrastructure platforms

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 1 | AWS | 28 | Accounts, IAM, resources, network, audit, findings |
| 2 | Microsoft Azure | 22 | Subscriptions, RBAC, resources, network, activity |
| 3 | Google Cloud | 18 | Org/folder/project, IAM, resources, audit |
| 4 | Oracle Cloud (OCI) | 10 | Compartments, IAM, compute, storage |
| 5 | Alibaba Cloud | 8 | RAM, ECS, OSS |
| 6 | Kubernetes (generic) | 12 | RBAC, workloads, service accounts, secrets, nodes |
| 7 | OpenShift | 6 | Projects, SCCs, routes |
| 8 | VMware vCenter | 8 | VMs, hosts, datastores, permissions |
| 9 | Microsoft Hyper-V / SCVMM | 5 | VMs, hosts |
| 10 | Nutanix | 5 | VMs, clusters |

**Subtotal: 10 connectors, 122 collectors**

AWS breakdown, as the reference for how a large connector decomposes:

```
  Organizations, IAM, Access Analyzer, STS, CloudTrail, Config,
  EC2, VPC, ELB/ALB, Route53, S3, EBS, RDS, DynamoDB, Redshift,
  Lambda, ECS, EKS, ECR, SSM, Secrets Manager, KMS, SNS/SQS,
  CloudFormation, GuardDuty, Security Hub, Inspector, Macie
```

### 2.2 Identity and access

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 11 | Microsoft Entra ID | 14 | Users, groups, roles, app regs, SPs, consent, CA, PIM, sign-ins |
| 12 | Active Directory (LDAP) | 11 | Users, groups, computers, OUs, GPOs, ACLs, trusts, delegation, SPNs |
| 13 | AD Certificate Services | 4 | CAs, templates, enrollment rights, issued certs |
| 14 | Okta | 8 | Users, groups, apps, policies, roles, logs |
| 15 | Ping Identity | 6 | Users, groups, apps, policies |
| 16 | OneLogin | 5 | Users, groups, apps |
| 17 | JumpCloud | 5 | Users, groups, devices |
| 18 | Google Workspace | 8 | Users, groups, OUs, OAuth apps, domain-wide delegation |
| 19 | AWS IAM Identity Center | 5 | Permission sets, assignments, groups |
| 20 | CyberArk | 6 | Vaults, safes, accounts, sessions |
| 21 | HashiCorp Vault | 6 | Auth methods, policies, roles, secret engines |
| 22 | Delinea / Thycotic | 5 | Secret server folders, permissions |
| 23 | BeyondTrust | 5 | Password safe, sessions |
| 24 | SailPoint | 6 | Identities, entitlements, certifications, roles |
| 25 | Saviynt | 5 | Identities, entitlements |
| 26 | Duo | 3 | MFA enrolment, policies, bypass |

**Subtotal: 16 connectors, 102 collectors**

This is the densest and most important block in the entire catalog. Per `02-iam-deep-dive.md`, ~73% of graph edges come from here.

### 2.3 Code, build and artifacts

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 27 | GitHub | 12 | Repos, teams, collaborators, OIDC trusts, Actions, secrets, code scan |
| 28 | GitLab | 10 | Projects, members, CI, OIDC, variables |
| 29 | Bitbucket | 8 | Repos, permissions, pipelines |
| 30 | Azure DevOps | 10 | Projects, repos, pipelines, service connections |
| 31 | Jenkins | 7 | Jobs, credentials, agents, plugins |
| 32 | CircleCI | 5 | Projects, contexts, env vars |
| 33 | Terraform Cloud/Enterprise | 6 | Workspaces, state, variables, run tasks |
| 34 | Harness | 5 | Pipelines, connectors, secrets |
| 35 | ArgoCD | 5 | Applications, projects, RBAC |
| 36 | JFrog Artifactory | 5 | Repos, permissions, artifacts |
| 37 | Sonatype Nexus | 4 | Repos, permissions |
| 38 | Container registries (DockerHub/Quay) | 4 | Images, permissions |

**Subtotal: 12 connectors, 81 collectors**

### 2.4 Security tooling (findings and runtime overlays)

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 39 | CrowdStrike Falcon | 8 | Hosts, processes, detections, vulns, RTR for response |
| 40 | Microsoft Defender for Endpoint | 8 | Devices, alerts, software inventory, TVM |
| 41 | Microsoft Defender for Cloud | 6 | Posture findings, secure score |
| 42 | SentinelOne | 6 | Agents, threats, applications |
| 43 | Palo Alto Cortex XDR | 6 | Endpoints, incidents |
| 44 | Tenable | 5 | Assets, vulnerabilities, scans |
| 45 | Qualys | 5 | Assets, vulnerabilities, policy compliance |
| 46 | Rapid7 InsightVM | 5 | Assets, vulnerabilities |
| 47 | Wiz | 4 | Cloud findings, inventory |
| 48 | Orca Security | 4 | Cloud findings |
| 49 | Snyk | 5 | Projects, issues, SBOM |
| 50 | Checkmarx | 4 | SAST results, projects |
| 51 | Veracode | 4 | Scans, flaws |
| 52 | Semgrep | 3 | Findings, rules |
| 53 | Aqua Security | 4 | Images, runtime policies |
| 54 | Prisma Cloud | 5 | Cloud + container findings |
| 55 | Black Duck | 3 | SCA, licences |

**Subtotal: 17 connectors, 85 collectors**

These are almost entirely **overlay connectors**: they do not create nodes, they attach properties and findings to nodes that already exist. That distinction matters for orchestration — overlays run last in a cycle and are cheap.

### 2.5 Network and edge

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 56 | Palo Alto NGFW / Panorama | 8 | Rules, zones, NAT, objects, logs |
| 57 | Fortinet FortiGate / FortiManager | 8 | Policies, interfaces, VPN, logs |
| 58 | Check Point | 7 | Rulebase, objects, gateways |
| 59 | Cisco ASA / Firepower | 7 | ACLs, objects, NAT |
| 60 | Juniper SRX | 5 | Policies, zones |
| 61 | F5 BIG-IP | 5 | Virtual servers, pools, iRules |
| 62 | Citrix ADC | 4 | vServers, policies |
| 63 | VMware NSX | 6 | Segments, DFW rules, groups |
| 64 | Zscaler | 6 | Policies, app segments, traffic, AI app usage |
| 65 | Netskope | 6 | Policies, CASB app usage, DLP events |
| 66 | Cisco Umbrella | 4 | DNS policy, requests |
| 67 | Infoblox / core DNS-DHCP | 5 | DNS records, DHCP leases, IPAM |
| 68 | NetFlow / IPFIX / sFlow | 4 | Flow aggregates |
| 69 | Cloudflare | 5 | Zones, WAF, Access, tunnels |
| 70 | Akamai | 4 | Properties, WAF |

**Subtotal: 15 connectors, 84 collectors**

### 2.6 Data platforms

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 71 | Oracle Database | 6 | Schemas, users, grants, classification |
| 72 | Microsoft SQL Server | 6 | Databases, logins, roles, classification |
| 73 | PostgreSQL | 5 | Databases, roles, grants |
| 74 | MySQL / MariaDB | 5 | Databases, users, grants |
| 75 | MongoDB | 5 | Databases, users, roles |
| 76 | Elasticsearch / OpenSearch | 5 | Indices, roles, mappings |
| 77 | Cassandra | 4 | Keyspaces, roles |
| 78 | Snowflake | 7 | Databases, roles, grants, shares, masking |
| 79 | Databricks | 7 | Workspaces, Unity Catalog, clusters, tokens |
| 80 | BigQuery | 5 | Datasets, IAM, classification |
| 81 | Amazon Redshift | 5 | Clusters, schemas, grants |
| 82 | Azure Synapse | 5 | Pools, permissions |
| 83 | SMB / NFS file shares | 5 | Shares, ACLs, classification |
| 84 | SharePoint / OneDrive | 7 | Sites, libraries, sharing links, permissions |
| 85 | Box | 5 | Folders, collaborations, shared links |
| 86 | Dropbox Business | 4 | Teams, shared folders |
| 87 | NetApp / Dell Isilon | 5 | Volumes, exports, ACLs |
| 88 | Google Drive | 5 | Files, sharing, permissions |

**Subtotal: 18 connectors, 96 collectors**

Data connectors are the only ones that perform **content classification** (DSPM), which makes them the most resource-hungry and the ones most in need of isolation from the rest of the fleet.

### 2.7 AI platforms and services

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 89 | OpenAI (org/admin API) | 6 | Projects, API keys, members, usage, models |
| 90 | Anthropic (org admin API) | 5 | Workspaces, API keys, members, usage |
| 91 | Google AI / Gemini | 4 | Projects, keys, usage |
| 92 | Hugging Face | 5 | Orgs, models, tokens, spaces |
| 93 | GitHub Copilot (admin) | 4 | Seats, usage, policies |
| 94 | Pinecone | 4 | Indexes, API keys, namespaces |
| 95 | Weaviate | 4 | Classes, schemas, keys |
| 96 | Qdrant | 4 | Collections, keys |
| 97 | Milvus / Zilliz | 4 | Collections, users |
| 98 | Chroma | 3 | Collections |
| 99 | LangSmith / LangChain platform | 5 | Traces, agents, tools, datasets |
| 100 | MCP registry / server discovery | 6 | Servers, tools, credentials, permissions |
| 101 | Glean | 4 | Sources, permissions, retrieval |
| 102 | Writer / Jasper / enterprise AI apps | 4 | Users, docs, connectors |
| 103 | Local model runtimes (Ollama, vLLM, LM Studio) | 5 | Models, endpoints, hosts |

**Subtotal: 15 connectors, 67 collectors**

Bedrock, Azure OpenAI and Vertex AI are *not* separate connectors — they are collectors inside AWS, Azure and GCP respectively. This is the counting unit doing useful work: they share the cloud credential and rate-limit domain.

### 2.8 Business context

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 104 | ServiceNow | 7 | CMDB, ownership, changes, incidents, tickets |
| 105 | Jira | 5 | Projects, issues, ticket creation for remediation |
| 106 | Workday | 4 | Workers, org structure, joiner/mover/leaver |
| 107 | BambooHR / HiBob | 3 | Employees, departments |
| 108 | Slack | 5 | Users, channels, apps, bots, AI usage |
| 109 | Microsoft Teams | 5 | Users, teams, apps, bots |
| 110 | Confluence | 4 | Spaces, permissions, content classification |
| 111 | PagerDuty | 3 | Services, ownership, on-call |

**Subtotal: 8 connectors, 36 collectors**

Business-context connectors are undervalued. ServiceNow and Workday supply **ownership** — and ownership is what turns a finding into an assigned task. A platform that cannot say who owns an asset generates work for the security team instead of for the team that can actually fix it.

### 2.9 Generic ingestion

| # | Connector | Collectors | Produces |
|---|---|---|---|
| 112 | Syslog receiver (CEF / LEEF / JSON / RFC5424) | 4 | Anything that speaks syslog |
| 113 | Webhook receiver | 3 | Push-based sources |
| 114 | Kafka / Event Hubs / Pub-Sub | 4 | Customers who already centralise |
| 115 | Cloud log-bucket puller (S3/Blob/GCS) | 3 | Archived logs |
| 116 | Generic REST poller (customer-defined) | 3 | Long-tail internal systems |
| 117 | CSV / file drop | 2 | Asset lists, crown-jewel designations, exclusions |
| 118 | SNMP / network device inventory | 3 | Unmanaged network gear |

**Subtotal: 7 connectors, 22 collectors**

The **generic REST poller** matters more than it looks. Every enterprise has 5–10 internally-built systems that will never get a bespoke connector. A declarative, customer-configurable poller with a mapping DSL converts an infinite long tail into a configuration exercise.

---

## 3. The number

```
   ┌──────────────────────────────────────────────────────────┐
   │  TOTAL CONNECTOR CATALOG                                 │
   │                                                          │
   │     118 connectors                                       │
   │     695 collectors                                       │
   └──────────────────────────────────────────────────────────┘

   By domain:
     Identity & access        16 connectors   102 collectors   <- densest
     Data platforms           18              96
     Security tooling         17              85
     Network & edge           15              84
     Cloud & infrastructure   10             122   <- most collectors each
     AI platforms             15              67
     Code, build, artifacts   12              81
     Business context          8              36
     Generic ingestion         7              22
     ---------------------------------------------------------
                             118             695
```

### 3.1 The coverage curve

Not all 118 carry equal weight. Measured by *what fraction of enterprise environments a connector applies to*:

```
   12 connectors   ->  produces the complete IAM spine and all five
                       hero findings, in ~85% of enterprises
                       AWS, Azure, Entra, GCP, AD, Okta, GitHub,
                       Kubernetes, CrowdStrike, Palo Alto,
                       ServiceNow, syslog

   34 connectors   ->  ~90% of what a typical enterprise runs

   60 connectors   ->  ~97%

  118 connectors   ->  ~99.5%, plus credible RFP coverage
```

The curve is steep and then very flat. That is not an argument for stopping at 34 — the last 84 connectors are what wins competitive RFPs, and each one is cheap once the framework exists. It *is* an argument for making sure the first 12 are excellent rather than making 118 mediocre.

### 3.2 The effort reality

```
   Bespoke connector, hand-built, no framework:       15-20 days each
   118 × 17 days = ~2,000 dev-days = 8 engineers × 1 year

   With a proper SDK, manifest-driven, generated scaffolding:
     large/complex (AWS, Azure, Entra, AD):           20-30 days   (8 of these)
     medium (GitHub, K8s, Snowflake, Palo Alto):       6-10 days   (40 of these)
     small overlay (Tenable, Snyk, Duo):               2-4 days    (70 of these)

   8×25 + 40×8 + 70×3 = 200 + 320 + 210 = 730 dev-days
   ≈ 6 engineers × 6 months for the full catalog
```

**The framework is worth roughly 1,270 dev-days.** That is the entire argument for building the connector SDK before building connectors, and it is why the catalog can be treated as one body of work rather than something rationed out over years.

---

## 4. The connector framework

### 4.1 Design goal

> A new small connector should take one engineer three days, of which two are reading the vendor's API documentation.

Everything below exists to make that true.

### 4.2 The manifest

Declarative, executable, and the single source of truth for the connector's behaviour, health model, UI presentation, and least-privilege policy.

```yaml
connector:
  id: github
  version: 3.2.0
  vendor: GitHub
  category: [code, identity]
  applies_to_pct: 71                # drives coverage reporting

  auth:
    methods:
      - id: github_app
        preferred: true
        scopes: [read:org, repo:read, actions:read, security_events:read]
      - id: pat
        deprecated: true
    least_privilege_doc: policies/github-app-permissions.md

  rate_limits:
    primary:   { requests: 5000, per: hour, scope: installation }
    secondary: { concurrent: 10 }
    strategy: token_bucket_with_backoff

  collectors:
    - id: repositories
      produces:
        entities: [REPOSITORY]
      schedule: { interval: 6h, delta_cursor: updated_at }
      full_enumeration: true
      cost: "1 call per 100 repos"

    - id: oidc_trusts
      produces:
        relationships:
          - { predicate: CAN_ASSUME, from: PIPELINE, to: ROLE, confidence: 0.95 }
      depends_on: [aws.iam, azure.entra]      # cross-connector dependency
      schedule: { interval: 12h }

    - id: secret_scanning
      produces:
        entities: [SECRET]
        relationships:
          - { predicate: CONTAINS, from: REPOSITORY, to: SECRET }
      schedule: { interval: 4h }
      requires_scope: security_events:read
      degrades_gracefully: true    # absent scope -> skip, don't fail

  health:
    success_criteria: "repositories >= 1 AND auth_errors == 0"
    parse_rate_baseline: 0.98
    staleness_threshold: 18h

  emits_coverage_window: true      # enables safe tombstoning
```

Everything the runtime needs is here: scheduling, rate limits, dependencies, health, coverage, and what the connector is allowed to claim about the graph.

### 4.3 The SDK contract

Each collector implements a small interface. The framework supplies everything else.

```
  Framework provides (connector author never writes these):
    credential acquisition and rotation
    HTTP client with retry, backoff, jitter, circuit breaking
    pagination helpers (cursor, offset, link-header, token)
    rate-limit governance and 429 handling
    cursor persistence for delta collection
    coverage-window emission
    entity/relationship emit API with schema validation
    normalization helpers (timestamps, IPs, ARNs, DNs, case folding)
    canonical-key derivation helpers
    error classification and health reporting
    metrics, tracing, structured logging
    sandboxing and resource limits
    test harness with recorded fixtures

  Connector author writes:
    fetch()    -> pull raw objects from the source
    map()      -> raw object -> entities + relationships
    key()      -> raw object -> canonical key candidates
    (optional) verify() -> confirm a high-value inference
```

### 4.4 Conformance harness

Every connector must pass the same suite before it ships. This is what makes 118 connectors maintainable by a small team.

```
  1. SCHEMA        every emitted entity/relationship validates against
                   the Security Fact schema and the closed predicate enum
  2. CANONICAL KEY every entity emits at least one key from the priority
                   list, correctly normalized
  3. IDEMPOTENCY   running twice against a fixture produces identical facts
  4. PAGINATION    a 3-page fixture yields all records, no duplicates
  5. RATE LIMIT    a 429 fixture triggers backoff, not failure
  6. AUTH FAILURE  a 401/403 fixture opens the circuit breaker and does
                   NOT retry into an account lockout
  7. PARTIAL       a mid-enumeration failure does NOT emit a coverage window
  8. DEGRADATION   a missing optional scope skips a collector cleanly
  9. PRIVACY       no raw values in emitted facts; all tokenizable
                   fields tokenized
 10. RESOURCE      completes a 10k-object fixture within the declared budget
```

Requirement 6 deserves emphasis: a connector that retries a bad credential in a loop will **lock the customer's service account**, which turns a monitoring tool into an outage. This has happened to real products and it is worth a dedicated test.

---

## 5. Orchestration: the single run cycle

### 5.1 The model

All connectors are governed by one control loop on the Edge Collector. There is no notion of some connectors being "live" and others "not yet built" — the fleet is a single scheduled system, and a connector that a customer has not configured is simply idle.

Within each cycle there is **dependency ordering**, because some connectors need the output of others to resolve entities correctly. This is the only sequencing in the design, and it operates on the timescale of a run.

```
   ┌───────────────────────────────────────────────────────────┐
   │                  CONNECTOR CONTROL LOOP                    │
   │                     (Edge Collector, R1/R2)                     │
   │                                                            │
   │   SCHEDULER ──► DEPENDENCY GATE ──► DISPATCHER             │
   │       ▲                                   │                │
   │       │                                   ▼                │
   │  HEALTH STATE ◄── RESULT SINK ◄──── WORKER POOL            │
   │       ▲                                   │                │
   │       │                                   ▼                │
   │  RATE GOVERNOR ◄──────────────── CREDENTIAL BROKER         │
   └───────────────────────────────────────────────────────────┘
```

### 5.2 The five bands of a cycle

Ordering within a cycle is driven by what entity resolution needs, not by product priority.

```
  BAND 0 — PREFLIGHT                        ~30 seconds
    credential validity checks (cheap, non-mutating)
    connector reachability probes
    content/manifest version check
    -> connectors failing preflight are skipped, not retried into lockout

  BAND 1 — IDENTITY AUTHORITIES             runs first, always
    Entra ID, Active Directory, Okta, Google Workspace,
    AWS Organizations, GCP Resource Manager, Azure management groups
    WHY FIRST: canonical keys and the Resolution Directory come from here.
    Every downstream connector resolves its identities against this output.
    Collecting AWS IAM before Entra means AWS identities resolve to
    weaker keys and may fail to merge.

  BAND 2 — PLATFORM INVENTORY AND GRANTS    the bulk of the work
    AWS, Azure, GCP, OCI, vCenter, Hyper-V, Nutanix
    resources + IAM grants + constraints
    -> feeds permission closure (02-iam §5)

  BAND 3 — WORKLOAD AND SUPPLY CHAIN        needs Band 2 identifiers
    Kubernetes (needs cloud IDs for the IRSA/WI bridge)
    GitHub, GitLab, Azure DevOps, Jenkins, Terraform Cloud
    registries, ArgoCD
    -> OIDC trust edges need both the cloud role AND the repo

  BAND 4 — DATA, AI AND NETWORK             independent, heaviest
    database connectors, file shares, SharePoint, DSPM classification
    AI platforms, vector DBs, MCP discovery
    firewalls, proxies, SWG, DNS, flow aggregation
    -> isolated resource pool; must never starve Bands 1-3

  BAND 5 — OVERLAYS                         last, cheap, attach-only
    vulnerability scanners, EDR, CNAPP findings, SAST/SCA,
    ServiceNow ownership, Workday org data
    -> these create almost no nodes; they attach properties to
       nodes that already exist. Running them last means they
       always find their target.
```

### 5.3 Why bands, not a strict DAG

A strict per-collector dependency DAG is more precise and much harder to operate. Bands give 95% of the benefit:

- A connector declares `depends_on` at the **collector** level for genuine hard dependencies (GitHub `oidc_trusts` needs `aws.iam`).
- The dependency gate defers just those collectors, not the whole connector.
- Everything else runs as soon as its band opens.
- A band opens when the previous band reaches **quorum**, not completion — typically 80% of its connectors healthy — so one slow source cannot stall the cycle.

```
   Band opens when:  prior_band_healthy_pct >= 80%
                     OR prior_band_elapsed > band_timeout
   Late arrivals from a prior band still emit; their facts merge
   normally on the next cycle. Nothing is lost, only delayed.
```

### 5.4 Cadence

Not everything runs at the same rate. Each collector declares its own interval; the scheduler jitters within it.

```
  CONTINUOUS   syslog, webhooks, flow, agent telemetry, gateway facts
               (not part of the banded cycle at all — always streaming)

  15 min       cloud audit trails (CloudTrail, Activity Log, Cloud Audit)
               high-value change detection

  1 hour       identity changes (Entra delta query, Okta events,
               AD uSNChanged delta)

  4 hours      cloud resource inventory delta, IAM policy delta,
               repo/CI state, K8s state, MCP config from agents

  12 hours     full IAM enumeration + permission closure
               network device configs
               AI platform inventory

  24 hours     full resource enumeration with coverage windows
               overlay connectors (scanners, EDR inventory)
               business context (ServiceNow, Workday)

  7 days       DSPM deep classification (rolling, partitioned —
               never scans everything at once)
```

**DSPM must be partitioned.** A full classification pass over petabytes cannot be a scheduled job; it is a rolling crawl that covers a slice per night and reports coverage as a percentage of the estate with an age distribution.

---

## 6. Rate governance

### 6.1 The problem

118 connectors against shared API quotas will, without governance, exhaust the customer's rate limits and break *their* production automation. That is the fastest possible way to get uninstalled.

### 6.2 The governor

```
   Token buckets, hierarchical, one per rate-limit domain:

     tenant
       └── provider (aws)
             └── account (123456789012)
                   └── service (iam)
                         └── operation class (list vs get vs write)

   Each level has a configurable ceiling. A collector must acquire
   tokens at EVERY level before issuing a call.

   Defaults are deliberately conservative — target 30% of the
   published quota — leaving 70% for the customer's own workloads.
   The customer can raise it; they will never be surprised by it.
```

### 6.3 Adaptive behaviour

```
   429 / throttle response
        -> exponential backoff with full jitter
        -> reduce that bucket's ceiling by 50%
        -> recover 10% per successful minute
        -> if throttled 5× in 10 min: open circuit, mark degraded,
           surface in local UI with the affected coverage

   Cost-aware collection (per manifest `cost` field):
        expensive collectors run at lower frequency automatically
        when the governor is under pressure — rather than failing
```

### 6.4 Fair scheduling

```
   Without fairness, one 400-account AWS org starves everything else.

   Weighted fair queueing across connectors:
       each connector gets a guaranteed minimum share of worker slots
       leftover capacity distributed by priority class
       no connector may hold more than 40% of the pool

   Priority classes:
       P0  identity authorities (Band 1)
       P1  cloud IAM and inventory (Band 2)
       P2  workload and supply chain (Band 3)
       P3  data, AI, network (Band 4)
       P4  overlays (Band 5)
```

---

## 7. Concurrency and sizing

### 7.1 Worker pools

Separate pools by resource profile, because one pool means a DSPM scan starves the identity connectors — the single most common way a collector like this falls over.

```
   POOL A — API/IO bound          most connectors
     concurrency = 4 × vCPU, capped at 64
     each worker is mostly waiting on network

   POOL B — CPU bound             parsing, classification, closure
     concurrency = vCPU − 2
     protects the box from saturation

   POOL C — SCAN                  DSPM crawling, file shares
     concurrency = 4, hard-capped
     lowest priority, yields to A and B
     ISOLATED: cannot borrow from other pools

   POOL D — REALTIME              resolve API, response, agent gateway
     reserved capacity, never yielded
     protects the analyst-facing latency budget
```

### 7.2 Per-profile expectations

Against the Edge Collector profiles in `01-system-design.md` §10.3:

| Profile | Connectors configured | Concurrent workers | Full cycle | Incremental |
|---|---|---|---|---|
| S (<500 hosts) | 8–15 | 16 | ~20 min | ~2 min |
| M (<5k hosts) | 15–30 | 32 | ~45 min | ~4 min |
| L (<25k hosts) | 30–60 | 64 | ~90 min | ~8 min |
| *(beyond Edge L)* | add another collector — there is no XL | | | |

These are targets to validate under load, not measurements. The number that actually matters is the **incremental** column — that is what determines whether the change feed feels live.

---

## 8. Failure isolation

### 8.1 Blast radius rules

```
   One collector fails      -> that collector's data is stale.
                               Other collectors in the connector continue.
   One connector fails      -> that source is stale and marked.
                               NOTHING is tombstoned (§8.2).
                               Other connectors unaffected.
   One credential expires   -> circuit opens after 2 auth failures.
                               NO retry loop. Customer alerted in local UI.
   A source is unreachable  -> exponential backoff to a 1h ceiling,
                               connector marked degraded, coverage reported.
   A connector crashes      -> sandboxed worker dies alone; supervisor
                               restarts with backoff; 3 crashes in 10 min
                               quarantines it until manual reset.
   A connector goes rogue   -> resource limits (CPU, memory, call rate)
   (memory leak, hot loop)     enforced per worker; killed on breach.
```

### 8.2 Never tombstone on partial data

The most damaging failure mode in a graph product, restated here because it is the connector layer's responsibility to prevent it:

```
   Connector completed FULL enumeration successfully
       -> emit coverage window
       -> anything not seen within it is genuinely gone -> tombstone

   Connector failed, partially completed, or was throttled
       -> DO NOT emit a coverage window
       -> DO NOT tombstone anything
       -> mark the affected subgraph STALE with the reason
       -> surface staleness at the point of use in the UI,
          not on a separate health page
```

Without this, a broken connector silently "resolves" thousands of findings and the customer's exposure score improves while their actual exposure does not. That single bug can end the product's credibility.

### 8.3 Health as a first-class product surface

```
   PER CONNECTOR
     state          healthy | degraded | failed | not_configured
     last_success   timestamp
     coverage       "38 of 42 AWS accounts"      <- honest gaps
     parse_rate     0.98  (baseline 0.98)
     freshness      "resources 2h, IAM 40m, audit 3m"
     errors         classified, with remediation text
     quota          "using 22% of the AWS IAM quota"

   PER TENANT (rolled up into the CISO dashboard)
     "Cloud 94% | Identity 97% | Network 62% | Data 88% | AI 78%"
```

Coverage percentages tell the customer **what Overlook cannot see**. That builds trust, and it drives expansion honestly: "Network 62%" is both an admission and a sales motion.

---

## 9. Credential brokering

118 connectors means up to 118 credential sets — often more, since large customers have many cloud accounts. The Edge Collector is therefore one of the most valuable targets in the environment.

```
   BROKER MODEL

   Connector worker                  Credential Broker (separate process)
        │                                       │
        │  request(connector_id, purpose)       │
        ├──────────────────────────────────────►│
        │                                       │ policy check
        │                                       │ unwrap from KMS/HSM
        │  scoped handle, TTL 5 min             │ audit log entry
        │◄──────────────────────────────────────┤
        │                                       │
        │  worker never sees the raw secret     │
        │  at rest; it is injected into the     │
        │  HTTP client at call time and zeroed  │
```

Preference order, enforced by the manifest and surfaced during onboarding:

```
   1. Workload identity (IRSA, Managed Identity, GCP WIF)   no secret at all
   2. OIDC federation with short-lived tokens
   3. Vendor app model (GitHub App, Entra app with cert)
   4. Static credential with automatic rotation
   5. Static credential                                     last resort
```

Least-privilege policies ship **with** the connector as applyable IaC, and read privileges are always separate from response privileges so a customer can deploy the entire fleet with zero write capability.

---

## 10. Delta collection

Full enumeration of everything, every cycle, does not scale past a mid-size customer. Delta strategy per source type:

```
   NATIVE CHANGE FEED    best — use wherever available
     Entra delta query, AWS CloudTrail/EventBridge, GCP asset feed,
     Azure Event Grid, K8s watch, GitHub webhooks, AD uSNChanged/DirSync

   CURSOR / WATERMARK    updated_at or sequence-based
     store cursor per collector, resume from it
     periodic full sweep to catch missed deletions and drift

   CONTENT HASH          for config-style sources
     hash the fetched config; skip processing if unchanged
     firewall rulebases and IAM policies benefit enormously

   FULL ONLY             when the source offers nothing better
     run at the lowest cadence that meets the freshness target
```

**A full sweep is still required periodically even with a change feed**, because change feeds miss deletions, backfills, and anything that happened during an outage. Cadence: full sweep daily for identity and IAM, weekly for bulk resources. The full sweep is also what legitimately emits the coverage window.

---

## 11. Onboarding

A customer with 30 connectors to configure will stall unless onboarding is engineered as carefully as collection.

```
   1. DISCOVER      Overlook detects what exists before asking.
                    Cloud org APIs enumerate accounts. AD reveals trusts.
                    Entra reveals federated apps. DNS and flow reveal
                    which SaaS and security tools are in use.
                    -> "We detected 42 AWS accounts, Okta, CrowdStrike,
                        Palo Alto and GitHub. Connect them?"

   2. PRIORITISE    Order the setup list by graph value, not alphabetically.
                    Identity authorities first — they unlock resolution
                    for everything else.

   3. GUIDE         Per connector: exact least-privilege policy as
                    copy-paste IaC, a validation button, and a plain
                    statement of what it will and will not read.

   4. VERIFY        Immediately after connecting, run a scoped test
                    collection and show real results:
                    "Connected. Found 1,204 identities, 38 roles,
                     12 with admin. First finding in 4 minutes."

   5. SHOW THE GAP  Persistent, honest coverage reporting:
                    "Network coverage 62% — 3 firewalls not connected.
                     Connecting them would reveal an estimated
                     14 additional attack paths."
```

Step 5 is the growth loop. It is honest, it is specific, and it makes the next connector the customer's idea.

---

## 12. Building the fleet as one program

No sequencing, no waves. The catalog is one body of work, parallelised by structure rather than by calendar.

### 12.1 Parallel tracks

```
   TRACK A — FRAMEWORK          2 engineers, continuous
     SDK, manifest runtime, scheduler, rate governor, credential broker,
     conformance harness, fixture recorder, health model
     Everything else depends on this. It is never "done"; it absorbs
     every capability that would otherwise be duplicated in a connector.

   TRACK B — DEEP CONNECTORS    3 engineers
     AWS, Azure, Entra, GCP, Active Directory, AD CS, Kubernetes, Okta
     These are genuinely hard, carry the IAM semantics, and cannot be
     rushed or templated. They also define what the SDK must support.

   TRACK C — BREADTH            3 engineers
     The ~70 small and medium connectors. Highly parallel, template-driven,
     scaffolding generated from OpenAPI specs where they exist.
     Throughput target: 2 engineers × 1 connector per 3 days sustained.

   TRACK D — CONTENT            2 engineers
     iam-semantics, escalation primitives, AI fingerprints, MCP reputation,
     parser grammars, classification patterns.
     Ships independently of code via the content pipeline.

   TRACK E — INGESTION          1 engineer
     Syslog, webhook, Kafka, flow aggregation, generic REST poller.
     Different shape from API connectors; deserves its own owner.
```

### 12.2 The dependency between tracks

```
   Track A must lead Track C by about 6 weeks — the SDK needs to be
   stable before breadth work compounds, or 70 connectors inherit
   the same defects and every fix becomes 70 pull requests.

   Track B informs Track A continuously. The hard connectors reveal
   what the framework is missing. Build AWS and Entra first
   specifically to stress the SDK before Track C scales up.

   Track D is independent of all of them and can start immediately.
   Escalation primitives need no connector to exist — they need
   only the capability schema.
```

### 12.3 Definition of done for a connector

```
   [ ] manifest complete and validated
   [ ] all conformance tests pass
   [ ] least-privilege policy documented and applyable as IaC
   [ ] recorded fixtures committed for regression
   [ ] coverage window emission verified
   [ ] delta strategy implemented (not just full enumeration)
   [ ] health criteria defined with a real baseline
   [ ] onboarding text written, with a plain statement of what it reads
   [ ] entity/relationship output reviewed against the graph model
   [ ] canonical key priority verified against a second source
```

That last item is the one most likely to be skipped and the most expensive to skip: a connector that emits weak canonical keys silently fragments the graph, and the symptom appears far away, as a missing attack path nobody knows to look for.

---

## 13. Summary

```
   118 connectors, 695 collectors — the full catalog

   12 connectors produce the complete IAM spine and all five
      hero findings in ~85% of enterprises
   34 cover ~90% of what a typical enterprise runs
   60 cover ~97%

   Framework first: it is worth ~1,270 dev-days across the catalog
   Total build: ~730 dev-days ≈ 6 engineers × 6 months, five parallel tracks

   ONE control loop on the Edge Collector governs the whole fleet.
   Ordering exists only WITHIN a run cycle — five bands, gated on
   quorum rather than completion, because entity resolution needs
   identity authorities to land before anything that references them.

   Rate governance targets 30% of customer quota by default.
   Four isolated worker pools so DSPM can never starve identity.
   Coverage windows are the contract that makes tombstoning safe.
   Health and coverage are product surfaces, not diagnostics.
```

---

*End of document.*

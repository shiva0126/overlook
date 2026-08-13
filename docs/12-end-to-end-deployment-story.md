# Overlook — End-to-End Deployment Story

**Version:** 0.1
**Date:** 2026-08-13
**Synthesises:** docs 01–11
**Status:** A worked scenario. Numbers are engineering estimates for sizing, not measurements.

---

## How to read this

One fictional customer, fully deployed. Every source, every component, every engine, traced from the wire to the finding. The point is to test whether the architecture in docs 01–11 actually holds under a realistic flood.

| Part | Covers |
|---|---|
| I | The customer, and where every component physically sits |
| II | The flood, quantified — what actually arrives per day |
| III | Eight component stories, one per source type |
| IV | The engines: what runs where, and when |
| V | Convergence — how eight sources become one person |
| VI | What the customer sees |
| VII | What breaks, and what happens then |

---

# PART I — THE CUSTOMER AND THE DEPLOYMENT

## 1. Meridian Financial

```
  PEOPLE        12,000 employees, 3 continents
  ENDPOINTS     8,500  — 6,200 Windows, 1,400 macOS, 900 Linux servers

  ON-PREMISES
    Active Directory      2 forests, 3 domains, 1 AD CS deployment
    VMware vSphere        4 clusters, 1,100 VMs
    Oracle + MS SQL       payments, customer records
    File shares           SMB, 40 TB
    Firewalls             2× Palo Alto (data centre), 2× FortiGate (branch)
    NetFlow               from 6 core switches

  IDENTITY & SECURITY
    Microsoft Entra ID    12,000 users, 400 app registrations, 900 SPs
    CrowdStrike Falcon    8,500 sensors
    Forcepoint DLP        endpoint + network

  CLOUD
    AWS      42 accounts    8,400 IAM roles, 2,100 users, 21,000 policies
    Azure    18 subs        12,000 RBAC assignments
    GCP       6 projects       800 IAM bindings
```

This is the **Hybrid archetype** (`09 §4, Archetype 3`) — the case Overlook exists for, and the one where the attack path crosses boundaries no single competitor can see across.

## 2. Where the components sit

```
 ┌─ MERIDIAN DATA CENTRE ────────────────────────────────────────────┐
 │                                                                    │
 │  Active Directory ──LDAPS──┐                                       │
 │  AD CS ────────────────────┤                                       │
 │  Palo Alto ×2 ──syslog/TLS─┤                                       │
 │  FortiGate ×2 ──syslog/TLS─┤                                       │
 │  Core switches ──NetFlow───┤     ┌──────────────────────────────┐  │
 │  VMware vCenter ──API──────┼────►│      EDGE-DC1                │  │
 │  Oracle / MSSQL ──JDBC─────┤     │      profile L               │  │
 │  File shares ──SMB─────────┤     │      32 vCPU / 128 GB / 8 TB │  │
 │  Forcepoint DLP ──API──────┤     │                              │  │
 │  CrowdStrike ──API─────────┤     │  ROLE: resolution PRIMARY    │  │
 │                            │     │  (identity authority is AD)  │  │
 │  8,500 AGENTS ──mTLS───────┘     └───────────┬──────────────────┘  │
 │                                              │                     │
 └──────────────────────────────────────────────┼─────────────────────┘
                                                │ replicate:
                                                │ resolution directory
                                                │ + deployment_key
 ┌─ MERIDIAN AWS (private subnet, no public IP) ─┼─────────────────────┐
 │                                               ▼                     │
 │  AWS 42 accts ──IRSA──┐        ┌──────────────────────────────┐    │
 │  Azure 18 subs ──MI───┤        │      EDGE-CLD                │    │
 │  GCP 6 projects ──WIF─┼───────►│      profile M               │    │
 │  Entra ID ──Graph─────┤        │      16 vCPU / 64 GB / 2 TB  │    │
 │  GitHub ──App─────────┤        │                              │    │
 │  EKS/AKS/GKE ──API────┘        └───────────┬──────────────────┘    │
 │                                            │                       │
 └────────────────────────────────────────────┼───────────────────────┘
                                              │
                              MODE 2 only:    │  Security Facts
                                    mTLS 443  │  ~12 MB/day
                                              ▼
                                     OVERLOOK MSSP CONSOLE
                                        (tokens only)
```

**Why two Edge Nodes.** Not for capacity — for **reach and trust**. `EDGE-DC1` needs LDAP line-of-sight to domain controllers and must receive syslog and NetFlow from devices that will not route to the internet. `EDGE-CLD` uses workload identity (IRSA, Managed Identity, Workload Identity Federation) so **no cloud credentials are ever stored anywhere**. Splitting them means neither holds credentials it does not need.

**Why DC1 is resolution primary.** Active Directory is the authoritative source of workforce identity here. Canonical keys derive from it, so the Resolution Directory lives beside it and `EDGE-CLD` queries it (`01 §8.3`).

**Both share one `deployment_key`.** Generated at enrollment on DC1, wrapped by Meridian's KMS, distributed to CLD through customer-controlled enrollment. If they did not share it, the same person would tokenize differently on each node and the hybrid graph would silently split into two disconnected halves (`09 §2.2`).

---

# PART II — THE FLOOD

## 3. What actually arrives, per day

```
  SOURCE                    VOLUME/DAY   INGRESS   JOURNALED?
  ────────────────────────  ──────────   ───────   ───────────────────
  Firewall syslog ×4         1.24 TB     STREAM    aggregate first
    ~36,000 EPS sustained    3.1B events
  NetFlow (6 switches)        900 GB     STREAM    aggregate first
                             4.1B records
  CrowdStrike API              40 GB     PULL      cursor only
  Cloud policy + config        18 GB     PULL      cursor only
  VMware / Oracle / shares     12 GB     PULL      cursor only
  Active Directory            2.8 GB     PULL      cursor only
    (full sweep nightly)
  Forcepoint DLP              1.1 GB     PULL      cursor only
  Entra ID (Graph + delta)     400 MB    PULL      cursor only
  8,500 agents                 102 MB    AGENT     yes, fsync
  GitHub webhooks               18 MB    PUSH      yes, fsync
  ────────────────────────  ──────────
  TOTAL                       ~2.2 TB/day
```

**Note what is journaled.** Only ~120 MB of the 2.2 TB gets durably journaled before acknowledgement — the agent and webhook data, because it is unrecoverable if dropped (`04 §2.1`). Everything else is either re-fetchable by cursor (PULL) or aggregated at receive (STREAM). This is why the appliance survives the flood: **we do not try to durably store 2.2 TB/day.**

## 4. The two regimes

```
  INITIAL LOAD — first 72 hours          STEADY STATE — every day after
  ──────────────────────────────         ─────────────────────────────────
  full enumeration of everything          delta collection on cursors
  AD: all 24,000 objects + 720,000 ACEs   AD: uSNChanged delta, ~400 objects
  AWS: all 21,000 policies                AWS: policy delta, ~40 changed
  permission closure from scratch         incremental closure on change
  entity resolution from scratch          resolution for new entities only
  ~14 hours of processing                 ~8 minutes of processing
  resolution review queue fills           queue trickles, 2-6 items/day
  ~180,000 facts emitted                  ~2,900 facts emitted
```

Two regimes, one pipeline. The first is a batch job that must not fall over; the second is a near-real-time loop. Sizing must satisfy both — the initial load defines peak memory, the steady state defines latency.

---

# PART III — EIGHT COMPONENT STORIES

Each story: what arrives, which ingress class, what each stage does, which engines fire, and what reaches the graph.

---

## 5. The firewall story — 1.24 TB becomes 15 KB

The flood, and the clearest illustration of the architecture.

```
  WHAT ARRIVES
    4 devices, syslog over TCP/TLS to EDGE-DC1:6514
    ~36,000 events/sec sustained, 3.1 billion/day
    traffic logs, threat logs, system logs, config-change logs
```

**Stage 1 — RECEIVE.** The syslog listener terminates TLS, frames records, and immediately applies **per-source priority classification**. Traffic logs are the lowest-value, highest-volume class here. They are handed straight to the Stream Aggregator without touching disk.

**Engine E1 — Stream Aggregator.** In-memory tumbling windows, 15 minutes, keyed on `(src_subnet, dst_subnet, dst_port, protocol, action)`.

```
  3.1 billion traffic events
        ↓  aggregate
  ~120,000 aggregate records/day        ≈ 26,000 : 1
```

Only the aggregate is journaled. The individual traffic events are gone within 15 minutes and were never written to disk. **We do not need to know that host A talked to host B at 14:22:07. We need to know that this subnet reaches that port on that subnet.**

**Stage 2 — IDENTIFY.** For a known source the registry answers instantly: `10.4.0.9:6514 → paloalto / PAN-OS 11.1`.

**Stage 3 — PARSE.** Engine E3 runs the PAN-OS grammar. Parse rate is monitored per device against a 98% baseline — the defence against a firmware upgrade silently changing the log format (`04 §11.3`).

**Stage 4/5 — NORMALIZE, ENRICH.** Fields map to the Overlook schema; subnets resolve to known network segments.

**Stage 6 — RESOLVE.** Subnets become `NETWORK` entities; destination IPs resolve to `ASSET` entities via time-bounded IP bindings.

**Stage 8 — FACT BUILD.** 120,000 aggregates collapse into roughly **2,000 distinct `CONNECTS_TO` edges**, of which ~1,960 already exist and only bump `last_seen`. Per the emission policy (`04 §23`), unchanged facts do not transmit.

```
  DAILY OUTCOME FROM THE FIREWALLS
    ingested             1.24 TB
    facts emitted        ~40 new or changed edges  ≈ 15 KB
    reduction            ~80,000,000 : 1
```

### 5.1 But the traffic logs are not the valuable part

The far more valuable firewall data arrives on a completely different path: a **PULL connector** reading the **rulebase** every 12 hours.

```
  4 devices × ~4,000 rules = ~16,000 policy rules
        ↓  E7 (a reachability closure, structurally similar to
           permission closure — evaluate rule order, zones, NAT,
           address objects, service groups)
  ~14,000 ROUTES_TO edges with conditions

  These are CONFIGURED reachability — what is possible.
  The CONNECTS_TO edges from traffic are OBSERVED reachability —
  what happened.

  Both matter, and they disagree usefully:
    configured but never observed  → an unused rule, attack surface
    observed but not configured    → a shadowed rule or a bypass
```

**This is doc 08's PaaS trap in miniature.** The syslog stream is 1.24 TB and yields 40 edges. The config pull is 40 MB and yields 14,000 edges. *The configuration API is worth 350× more per byte than the log stream.*

---

## 6. The NetFlow story — 4.1 billion records

Same shape, larger numbers.

```
  6 core switches → NetFlow v9 → EDGE-DC1:2055
  4.1 billion records/day

  E1 STREAM AGGREGATOR
    key: (src_subnet, dst_subnet, dst_port, protocol)
    window: 15 minutes
    → 180,000 aggregates/day        ≈ 23,000 : 1

  → ~4,000 CONNECTS_TO edges, ~3,950 unchanged
  → ~50 emitted facts/day
```

**What NetFlow uniquely gives us:** east-west reachability *inside* segments the firewalls never see. The Palo Altos see traffic crossing zones; NetFlow sees the VLAN where 400 servers talk to each other freely. That is where lateral movement lives, and it is invisible to firewall logs.

**One engineering note.** Aggregation windows are held in memory: 180,000 keys × ~200 bytes ≈ 36 MB. Trivial. But during the initial load, before subnets are known, key cardinality can spike — hence spill-to-disk in the aggregator design.

---

## 7. The Active Directory story — where the real graph comes from

The single richest source in this deployment, and nothing about it resembles the flood.

```
  WHAT ARRIVES  (PULL, LDAPS, nightly full sweep + hourly delta)
    12,000 users
     8,500 computers
     3,200 groups
       180 GPOs
        42 OUs
         3 domains, 2 forests, 4 trusts
   ~720,000 ACEs across all objects
    2.8 GB per full sweep
```

**Stage 6 — RESOLVE.** Each user yields a canonical key by priority (`01 §6.2`): `mail` attribute wins → `email:priya.s@meridian.com`. Aliases recorded: `adguid:…`, `sam:MERIDIAN\priyas`, `upn:…`. All of it lands in the **Resolution Directory** so `EDGE-CLD` can resolve the same person later.

**Stage 7 — DERIVE.** This is where AD becomes a graph rather than a directory.

```
  E7 — CLOSURE (the AD variant)
    nested group expansion: 3,200 groups, max depth 9
    → 95,000 direct MEMBER_OF edges
    → 412,000 effective memberships after transitive closure
       (materialized — 02 §5.4)

  ACL EVALUATION over 720,000 ACEs
    ~698,000 are default or inherited → discarded
    ~22,000 are non-default
    ~4,100 are security-relevant and become edges:
        GenericAll, GenericWrite, WriteDacl, WriteOwner,
        ForceChangePassword, AddMember,
        WriteProperty on msDS-KeyCredentialLink,
        WriteProperty on msDS-AllowedToActOnBehalfOfOtherIdentity,
        GetChanges + GetChangesAll

  DELEGATION
    unconstrained delegation      12 computers
    constrained delegation        40 accounts
    resource-based constrained     8 computers
    MachineAccountQuota = 10      ← default, never changed

  E8 — ESCALATION MATCHER
    WriteDacl on Domain Admins held by a non-admin
      → synthesize CAN_ASSUME (transitive to full control)
    RBCD writable + MachineAccountQuota > 0
      → synthesize CAN_ASSUME on 8 computers, CRITICAL
    DCSync rights on a service account
      → synthesize CAN_ASSUME on the domain itself
    Unconstrained delegation on a non-DC
      → synthesize CAN_ASSUME on any authenticating identity

  RESULT
    ~340 synthesized edges that appear in NO policy document
    and that no scanner in Meridian's environment reports
```

**Daily outcome:** 2.8 GB in, ~24,000 entities and ~416,000 edges on first load, then ~400 changed objects per day producing ~180 facts.

### 7.1 The collection problem nobody warns you about

Enumerating 720,000 ACEs over LDAP looks exactly like reconnaissance, and Meridian's own SOC will alert on it.

```
  MITIGATION, from day one
    dedicated read-only service account, documented
    paged queries, configurable rate limit
    a published collection profile handed to their SOC to allowlist
    full sweep in a nightly window; uSNChanged delta hourly
    optional: consume their existing SharpHound output instead
```

This belongs in the onboarding runbook, not in a support ticket after the first night.

---

## 8. The Entra ID story — and the moment the graph converges

```
  WHAT ARRIVES  (PULL, Microsoft Graph, delta queries hourly)
    12,000 users            ← the SAME 12,000 people as AD
       900 service principals
       400 app registrations
        60 directory role assignments
       140 OAuth consent grants
        24 Conditional Access policies
```

**Stage 6 — RESOLVE, and the convergence.** Entra user `priya.s@meridian.com` produces canonical key `email:priya.s@meridian.com` — **identical to the key AD produced**. Deterministic match, confidence 1.0. One entity, now with two sources.

That merge is the entire architecture working. Without it, Meridian has an AD graph and an Entra graph and no path between them.

**Stage 7 — DERIVE.** The Entra-specific escalation surface (`02 §14.2`):

```
  E8 — ESCALATION MATCHER finds
    3 app registrations holding RoleManagement.ReadWrite.Directory
      → synthesize CAN_ASSUME Global Administrator      CRITICAL
    11 service principals whose OWNERS are ordinary users
      → owner can add credentials → become the SP       HIGH
    1 dynamic group whose membership rule reads a
      user-writable attribute, and which holds a role
      → any user can grant themselves that role         HIGH
    Global Admin → elevateAccess → User Access Administrator
      at root management group
      → synthesize CAN_ASSUME Owner on ALL 18 Azure subs  CRITICAL

  CONDITIONAL ACCESS becomes PROTECTS edges, and its GAPS
  become findings:
    4 accounts excluded from the MFA policy, 2 of which
    can reach Global Administrator
```

That last `elevateAccess` chain is one edge that connects the identity plane to 18 subscriptions. Miss it, and every Azure attack path is understated.

---

## 9. The cloud story — 42 accounts, and the closure that matters

```
  WHAT ARRIVES  (PULL, IRSA/MI/WIF — no stored credentials)
    AWS      42 accounts:  8,400 roles, 2,100 users,
                           21,000 policies, ~180,000 statements,
                           4 SCPs, 62 permission boundaries
    Azure    18 subs:      12,000 role assignments, 40 custom roles,
                           6 deny assignments
    GCP       6 projects:    800 bindings, 12 custom roles
    Total    ~18 GB of policy documents per full sweep
```

**Stage 7 — E7 PERMISSION CLOSURE.** The most expensive computation in the appliance, and the one that produces the most value.

```
  FOR EACH PRINCIPAL, evaluate in strict precedence (02 §3.1):
    explicit DENY anywhere → stop
    SCP → resource policy → permission boundary
    → session policy → identity policy

  SCALE CONTROL (02 §5)
    action grouping     18,000 AWS actions → 40 groups → 12 capabilities
    pattern preservation  keep "arn:aws:s3:::prod-*", expand only for
                          the 47 crown-jewel buckets
    bounded chains      CAN_ASSUME closure to depth 6
    equivalence classes 380 identical Terraform-deployed roles across
                        accounts collapse to 1 class × 380

  OUTPUT
    ~2.1 million capability entries
    → ~310,000 graph edges after grouping and pattern preservation

  TIMING
    initial full closure   ~26 minutes on EDGE-CLD (profile M)
    incremental on change  ~600 ms for a typical policy edit
```

**Stage 7 — E8 ESCALATION MATCHER** then runs over the capability sets:

```
  AWS
    iam:PassRole(*) + lambda:CreateFunction + InvokeFunction   41 principals
    iam:CreatePolicyVersion on an attached policy               6
    ssm:SendCommand on instances with privileged roles         28
    secretsmanager:GetSecretValue on DB credential secrets      94
    s3:GetObject on Terraform state buckets                     12  ←
    GitHub OIDC trust with "repo:meridian/*" subject condition   3  ← CRITICAL
  AZURE
    Microsoft.Authorization/roleAssignments/write                9
    virtualMachines/runCommand on VMs with managed identities   61
  GCP
    iam.serviceAccounts.getAccessToken                          14
    cloudbuild.builds.create (default SA is project Editor)      2

  → ~270 synthesized escalation edges
```

**Every one of these is invisible to a log-based tool.** None appears in CloudTrail. They are standing conditions derived from evaluating policy documents — precisely what `11 §6.1` argues Stellar Cyber's machine structurally cannot produce.

---

## 10. The agent story — 8,500 endpoints without a flood

This is where the thin-agent decision (`01 §12.1`) pays for itself.

```
  WHAT WE DO NOT COLLECT
    ✗ full process trees        8,500 × 1,440 snapshots × 200 procs
                                = 2.4 BILLION records/day.
                                We take this from the CrowdStrike API
                                instead — better coverage, an already
                                approved kernel driver, zero new agent risk.

  WHAT WE DO COLLECT — only what no API can provide
    ✓ MCP client/server configs   ~/.claude/, ~/.cursor/,
                                  claude_desktop_config.json
    ✓ local model runtimes        ollama, LM Studio, vLLM
    ✓ IDE AI extensions           and the endpoints they call
    ✓ AI SDK / framework presence in local environments
    ✓ credential PRESENCE in AI configs — type only, never the value
    ✓ browser→AI-domain relationships with owning user and process

  CADENCE
    AI config scan   every 4h  → 8,500 × 6 = 51,000 reports/day
    heartbeat        every 60s → small
    report size      ~2 KB
    TOTAL            ~102 MB/day from 8,500 endpoints
```

102 MB from 8,500 endpoints. Compare with the 1.24 TB from four firewalls. **Scope discipline is worth more than any compression.**

**What it finds at Meridian:**

```
    340 endpoints with MCP server configurations
        of which 47 are mcp-filesystem rooted at a home or work
        directory, and 12 of those directories contain files
        already classified as sensitive
    120 endpoints running local models (ollama)
  4,200 endpoints with IDE AI extensions
     31 MCP configs holding a credential — GitHub tokens,
        AWS keys, database strings — presence recorded, value never read
```

**Stage 7 — E9 POSTURE RULES** fires on the combination:

```
  MCP filesystem server rooted at a directory containing PII
    + the agent using it can invoke a tool with external egress
    → FINDING: unmanaged AI data-exposure path
```

**And this is the finding nobody else in Meridian's stack can produce.** CrowdStrike sees a process. Forcepoint sees a file. Entra sees a user. Only the graph sees *the developer's laptop runs an MCP server holding a GitHub token whose repository has an OIDC trust into an AWS role that reads the payments database.*

---

## 11. The EDR and DLP stories — overlays

Both are **overlay connectors** (`03 §2.4`): they create almost no nodes. They attach properties and findings to nodes that already exist, which is why they run last in the cycle (Band 5).

```
  CROWDSTRIKE  (PULL, ~40 GB/day)
    8,500 host records          → attach to existing ASSET nodes:
                                   OS, patch level, sensor health,
                                   last seen, installed software
    process + network context   → on demand, for investigation
    detections                  → FINDING facts
    PROTECTS edges              → an asset with a healthy EDR sensor
                                   reduces the weight of CAN_EXECUTE
                                   edges into it (01 §21.3)
    RESPONSE                    → RTR is our response path; we do NOT
                                   build host response in our own agent

  FORCEPOINT DLP  (PULL, ~1.1 GB/day)
    policy definitions          → what Meridian considers sensitive
    incidents                   → attach to DATASTORE and IDENTITY nodes
    classification results      → DATA_CLASS properties, which feed
                                   crown-jewel designation

    DLP is genuinely valuable here: it tells us WHICH datastores hold
    regulated data, without us running our own DSPM scan. Deferring
    classification (10 §4.2) costs far less when the customer already
    owns a DLP.
```

**Both are Band 5** — they run after entities exist, so they always find their target. Running an overlay before the cloud connector has created the asset would produce orphaned properties.

---

# PART IV — THE ENGINES, WHERE AND WHEN

## 12. Physical placement

```
  ENGINE                          EDGE-DC1        EDGE-CLD
  ──────────────────────────────  ─────────────   ─────────────
  E1  Stream Aggregator           ✓ heavy         —
  E2  Source Fingerprint          ✓               —
  E3  Parser                      ✓ heavy         ✓ light (JSON)
  E4  Normalizer                  ✓               ✓
  E5  Enrichment                  ✓               ✓
  E6  Entity Resolution           ✓ PRIMARY       ✓ queries DC1
  E7  Permission Closure          ✓ AD variant    ✓ cloud variant
  E8  Escalation Matcher          ✓ AD catalog    ✓ cloud catalog
  E9  Posture Rules               ✓               ✓
  E10 Correlation                 ✓               —
  E11 Classification              — (DLP instead) —
  E12 Graph                       ✓ local         ✓ local
  E13 Fact Builder                ✓               ✓        Mode 2
  E14 Privacy Gate                ✓               ✓        Mode 2
  E15 Orchestration               ✓               ✓
  E16 Response Executor           ✓ disabled      ✓ disabled
```

Two graphs, two closures, one resolution authority. Each Edge computes closure for the sources it can reach, and both emit into one merged graph — either the local one in Mode 1 or the console's in Mode 2.

## 13. A day in the cycle

```
  00:00  BAND 0  PREFLIGHT — credential validity, reachability
                 (non-mutating; a failing credential is skipped,
                  never retried into an account lockout)

  00:02  BAND 1  IDENTITY AUTHORITIES — must land first
                 AD full sweep (DC1) · Entra delta (CLD)
                 AWS Organizations · Azure mgmt groups · GCP hierarchy
                 → canonical keys and the Resolution Directory update
                 → E6 runs continuously through this band
                 duration ~38 min; band opens the next at 80% quorum

  00:34  BAND 2  PLATFORM INVENTORY AND GRANTS
                 AWS IAM (42 accts) · Azure RBAC · GCP IAM · VMware
                 → E7 permission closure begins as data lands
                 duration ~26 min

  01:00  BAND 3  WORKLOAD AND SUPPLY CHAIN
                 Kubernetes (needs cloud IDs for the IRSA bridge)
                 GitHub (OIDC trusts need both repo AND cloud role)
                 duration ~9 min

  01:09  BAND 4  DATA, AI, NETWORK  (isolated pool — cannot starve 1-3)
                 Oracle/MSSQL grants · file share ACLs
                 firewall rulebases · AI platform APIs
                 duration ~22 min

  01:31  BAND 5  OVERLAYS
                 CrowdStrike · Forcepoint · vuln scanners
                 duration ~14 min

  01:45  E8 ESCALATION MATCHER over the completed closure
  01:52  E9 POSTURE RULES
  01:58  E12 GRAPH — coverage windows applied, retractions processed
  02:04  E13/E14 FACT BUILD + PRIVACY GATE  (Mode 2)
  02:07  sync complete. ~2,900 facts, 11.8 MB.

  CONTINUOUS, ALL DAY — outside the banded cycle entirely
    syslog + NetFlow receivers  → E1 aggregator
    agent gateway               → 8,500 endpoints
    GitHub webhooks
    cloud audit trails          → 15-minute cadence
    incremental closure on any IAM change → ~600 ms
```

**The bands are not a roadmap.** They are dependency ordering inside one 2-hour window, gated on quorum rather than completion so one slow source cannot stall the cycle (`03 §5.3`).

---

# PART V — CONVERGENCE

## 14. Eight sources become one person

The moment everything exists for.

```
  AD          MERIDIAN\priyas, objectGUID 8f14e45f, mail priya.s@meridian.com
  Entra       priya.s@meridian.com, id 00u1a2b3, 3 app role assignments
  AWS         arn:aws:iam::1234:user/priya.s, tag email=priya.s@meridian.com
  Azure       priya.s@meridian.com, Contributor on 2 subscriptions
  GitHub      psharma-meridian, verified email priya.s@meridian.com
  CrowdStrike CORP\priyas on LT-4471
  Agent       priyas on LT-4471, VS Code + Copilot, mcp-filesystem
  DLP         priya.s@meridian.com, 2 policy events

                            ↓  E6 ENTITY RESOLUTION

  CANONICAL KEY   email:priya.s@meridian.com
  ONE NODE        8 sources, resolution confidence 1.0
  ALIASES         adguid, sam, upn, ARN, GitHub login, EDR user string
```

Seven of the eight matched deterministically on a verified corporate email. The eighth — `CORP\priyas` from CrowdStrike — matched via the **Resolution Directory**, which knew the alias because AD had registered it hours earlier on a different Edge Node.

**That is why the Resolution Directory is mandatory rather than optional** (`01 §8.3`). Without it, CrowdStrike's view of Priya is a separate node and every path through her endpoint breaks.

## 15. And the path assembles itself

No single connector saw this. It exists only because eight of them fed one graph.

```
  Priya S  (Developer, Engineering — no admin rights anywhere)
    │
    │  agent: laptop LT-4471 runs mcp-filesystem rooted at ~/work
    │         config holds a GitHub token (presence detected)
    ▼
  GitHub token → repo meridian/payments-api
    │
    │  GitHub connector: repo has an OIDC trust to an AWS role
    │  with subject condition "repo:meridian/*"        ← too broad
    ▼
  AWS role GHADeployRole
    │
    │  E7 closure: can assume EC2AppRole
    │  E8 escalation: iam:PassRole(*) + lambda:CreateFunction
    ▼
  EC2AppRole  →  effectively administrator in account 1234
    │
    │  E7: s3 + rds read across the account
    ▼
  RDS prod-payments-db
    │
    │  Forcepoint DLP: classified PII + PCI
    │  crown jewel, criticality 95
    ▼
  4.2M customer records

  CONFIDENCE 0.91   (weakest link: MCP credential presence inferred,
                     not the credential value — we never read it)
  AGE       existed 84 days
  SEVERITY  CRITICAL

  CHOKE POINT
    the OIDC subject condition "repo:meridian/*"
    appears in 3 trusts and 1,240 paths.
    Tightening it to a specific repo and ref eliminates all 1,240.
    Estimated remediation: 40 minutes.
```

Meridian owns CrowdStrike, Forcepoint, four firewalls, AD, Entra and three clouds. **Not one of those tools can produce that sentence**, because each sees one hop.

---

# PART VI — WHAT THE CUSTOMER SEES

## 16. The Controller, 08:30

```
  ⚠ ATTENTION                                                  3 items

  ● NEW CRITICAL PATH                                     detected 02:04
    Developer laptop → MCP credential → GitHub OIDC → AWS admin
    → prod-payments-db (4.2M records, PII+PCI)
    Choke point: 1 OIDC subject condition, 1,240 paths
    [ View path ]  [ Break this path ]  [ Export for ticket ]

  ● PARSE RATE COLLAPSE                                          06:12
    fortigate / fw-branch-02 — 98% → 4%
    Firmware upgraded overnight; log format changed.
    41,000 records quarantined, samples retained.
    [ View samples ]  [ Replay after content update ]

  ● RESOLUTION REVIEW                                       6 pending
    Oldest 2 days. 2 involve privileged accounts.
    [ Review ]

  COVERAGE
    Identity 97% · Cloud 94% · Network 88% · Data 71% · AI 82%
    Runtime 91% (CrowdStrike) · Code 64% — 2 GitHub orgs unconnected
    → 128 path fragments dead-end at the missing GitHub org
```

## 17. The outbound inspector, if in Mode 2

```
  OUTBOUND · last 24h · 11.8 MB · 2,914 facts

  02:04:11  RELATIONSHIP  CAN_ASSUME
    IDN-9f3a7c21e845b0d6 → ROL-2b8e4f19a70c5d33
    [ before / after ▾ ]
      BEFORE  priya.s@meridian.com → arn:aws:iam::1234:role/GHADeployRole
              evidence: 4.2 KB trust policy document
      AFTER   IDN-9f3a… → ROL-2b8e…
              evidence: sha256:8a1f…c4d2  (hash only)
      STRIPPED  email, ARN, account id, role name, policy document
```

2.2 TB arrived. 11.8 MB left. Nothing identifying crossed the boundary.

---

# PART VII — WHEN THINGS BREAK

## 18. Six failures, and what actually happens

```
  1  AD CREDENTIAL EXPIRES AT 02:00
     circuit opens after 2 auth failures — NO retry loop, because a
     retry loop locks Meridian's service account and turns a
     monitoring tool into an outage.
     NO coverage window emitted → NOTHING tombstoned.
     24,000 AD entities marked STALE with the reason, visible on the
     path itself rather than buried in a health page.
     Attention inbox raises it with the graph consequence stated.

  2  FIREWALL FIRMWARE CHANGES THE LOG FORMAT
     parse rate 98% → 4%. Records QUARANTINED, never dropped.
     Samples retained locally — we cannot ask Meridian to email us
     logs, so the operator replays the journal against updated
     content once a parser fix ships.

  3  AWS THROTTLES US DURING THE CLOSURE
     governor backs off, halves that bucket's ceiling, recovers 10%
     per successful minute. Closure completes late rather than
     failing. Default ceiling is 30% of Meridian's quota — the
     other 70% is theirs, always.

  4  EDGE-CLD LOSES ITS LINK TO EDGE-DC1
     resolution falls back to a cached alias set. New cloud entities
     resolve with lower confidence and are marked "resolution
     degraded" rather than silently mis-merged. Bias is toward
     under-merge: a missed finding beats a false accusation.

  5  MSSP CONSOLE UNREACHABLE FOR 9 HOURS  (Mode 2)
     facts buffer on disk — 7-day capacity, 5% used.
     Collection, resolution and closure all continue.
     The Controller keeps working: local findings, evidence, config.
     On reconnect, drain oldest-first, rate-limited. Facts are
     idempotent, so replay is safe.

  6  A DSPM-STYLE SCAN TRIES TO EAT THE BOX
     it cannot. The scanner runs in its own process with a hard
     ceiling and the lowest priority, and cannot borrow capacity
     from the identity or realtime pools. This is the single most
     common way appliances like this fall over.
```

---

## 19. The whole thing in one frame

```
  IN     2.2 TB/day across 10 source types, 2 Edge Nodes,
         8,500 agents, 4 ingress classes

  KEPT   ~2.9 million graph edges, bitemporal, confidence-scored,
         evidence-referenced

  OUT    11.8 MB/day of tokenized facts  (Mode 2)
         or nothing at all               (Mode 1)

  FOUND  6 critical paths, 3 choke points, 1 of which is a
         40-minute fix that eliminates 1,240 paths

  THE POINT
    Every one of those 2.2 TB was seen. Almost none of it was kept.
    What was kept is not events — it is what is POSSIBLE.
    And the finding that matters came from combining eight sources
    that Meridian already owned, none of which could see past
    its own hop.
```

---

*End of document.*

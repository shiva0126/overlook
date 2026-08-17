# Overlook — Competitive Landscape and Feasible Positions

**Version:** 0.1
**Date:** 2026-08-13
**Companion to:** `06-prior-art-secops-stellarcyber.md`
**Purpose:** List who else is doing this, assess honestly what is left unoccupied, and identify what is actually feasible to build.

**Confidence note:** items marked ✔ were verified in this session's searches. Items marked ○ are from prior knowledge and should be verified before being used in any external material.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. The short version

The three claims kept in `01 §2.3` after the Chronicle survey were:

1. Residency is not blindness
2. Hybrid permission-closure depth
3. AI agents inside that closure

**After this survey, claim 2 is heavily contested and claim 3 is partially occupied.** Claim 1 stands alone and unchallenged.

```
   CLAIM 1  vendor blindness         ████████████  nobody else
   CLAIM 2  hybrid identity depth    ██░░░░░░░░░░  XM Cyber, BloodHound
   CLAIM 3  AI agents in closure     ████░░░░░░░░  BloodHound (Entra),
                                                    Zenity, Noma
```

This does not mean the project is dead. It means the *shape* of what is feasible has changed, and Part V proposes four honest options.

---

## 2. The landscape

### 2.1 Cloud-native exposure / CNAPP

| Vendor | What they do | Relevance |
|---|---|---|
| **Wiz** ✔ | Agentless CNAPP. Security Graph correlating misconfigurations, vulns, identities, exposure, data sensitivity and secrets into attack paths and toxic combinations. CSPM + CWPP + CIEM + DSPM + AI-SPM. **Acquired by Google, $32B, March 2026.** | The market leader, now inside Google |
| **Google SCC** ✔ | Risk Engine: attack path simulation, exposure scores, toxic combinations, chokepoints. GCP + AWS. Org-level activation. | Attack paths from a hyperscaler |
| **Orca Security** ○ | Agentless side-scanning CNAPP with attack path analysis | Direct Wiz competitor |
| **Palo Alto Prisma Cloud** ○ | CNAPP; absorbed Dig (DSPM) and others | Platform play |
| **Microsoft Defender for Cloud** ○ | CNAPP with attack path analysis, Cloud Security Explorer graph | Bundled with Azure |
| **Tenable Cloud Security** ○ | Formerly Ermetic. CIEM-strong CNAPP | Identity-centric cloud |
| **Upwind, Sysdig, Aqua, CrowdStrike Falcon Cloud Security** ○ | Runtime-centric CNAPP | Runtime angle |

### 2.2 Exposure / attack path management

| Vendor | What they do | Relevance |
|---|---|---|
| **XM Cyber** ✔ | Continuous Exposure Management. Hybrid attack paths validated **from internet-facing exposure → cloud AI models → on-prem databases → industrial systems**. May 2026: Identity Exposure Management with granular visibility into permissions **and their actual usage** across AD, Entra and cloud. March 2026: extended to AI attack surfaces. | **The closest direct competitor** |
| **Tenable One** ○ | Exposure management platform; includes Tenable Identity Exposure (formerly Alsid) for AD/Entra | Broad exposure play |
| **Rapid7 Exposure Command** ○ | Attack path and exposure management | Mid-market |
| **Qualys ETM** ○ | Enterprise TruRisk Management | Vuln-centric |
| **Cymulate / Picus / SafeBreach / Pentera** ○ | Breach and attack simulation — validation rather than graph | Adjacent |
| **Axonius** ○ | Asset intelligence and correlation across tools | Inventory layer |

### 2.3 Identity attack path / ISPM / authorization graph

This category is the closest to Overlook's IAM thesis and the most crowded.

| Vendor | What they do | Relevance |
|---|---|---|
| **BloodHound Enterprise (SpecterOps)** ✔ | Identity Attack Path Management across **AD, Entra, AWS, Okta and GitHub**. Jan 2026: **on-premises deployment option**. Jul 2026: AWS attack path management. **Extension for Microsoft Entra Agent ID — attack path management for Copilot agents and AI identities**, showing how AI agents, delegated identities and service principals create indirect paths to privileged access. Feb 2026: Scentry advisory service. | **The most direct threat to claims 2 and 3** |
| **Veza** ○ | Authorization graph — "who can do what" across cloud, SaaS, data systems | Closest to permission closure as a concept |
| **Silverfort** ○ | Identity security across on-prem and cloud, MFA everywhere, ITDR | Hybrid identity |
| **Semperis** ○ | AD/Entra security, recovery, attack path visibility | AD specialist |
| **Microsoft Defender for Identity** ○ | AD/Entra ITDR with attack path elements | Bundled |
| **Oleria, Sharelock, Push Security** ○ | Identity posture / SaaS identity | Emerging |

### 2.4 CIEM

| Vendor | Relevance |
|---|---|
| **Sonrai, Britive, Permiso, Entra Permissions Management, Tenable (Ermetic)** ○ | Granted-vs-used entitlement analysis. Our `02 §19` CIEM chapter is a well-served category |

### 2.5 AI security posture and agent security

Crowded, well funded, consolidating fast.

| Vendor | What they do | Relevance |
|---|---|---|
| **Zenity** ✔ | AI agent discovery, real-time inventory with **ownership and dependency mapping**, posture management, least-privilege policies pre-deployment, runtime step-level monitoring for privilege escalation and prompt injection, inline containment. **MCP: every server and tool in one view, including shadow servers from public registries** | Direct overlap with our AI chapters |
| **Noma Security** ✔ | $132M raised incl. $100M Series B. **Agent Access Control — discover, govern and enforce access policies for AI agents and MCP servers across the enterprise.** Targets homegrown AI apps, RAG pipelines, agents, MCP estates | Direct overlap |
| **Prompt Security** ✔ | Acquired by SentinelOne | Consolidated |
| **Aim Security** ✔ | Acquired by Cato Networks | Consolidated |
| **Lakera** ✔ | Acquired by Check Point | Consolidated |
| **Protect AI** ○ | Acquired by Palo Alto Networks | Consolidated |
| **Robust Intelligence** ○ | Acquired by Cisco | Consolidated |
| **HiddenLayer, Pillar, Straiker, Operant, Knostic, Witness AI** ○ | Model security, runtime, AI gateway, knowledge-access control | Fragmented remainder |

**The pattern:** every independent AI-security vendor of note has been acquired by a platform vendor, or is racing to be. Zenity and Noma are the significant remaining pure-plays, and both are using our vocabulary — "identity blast radius," "MCP servers," "least privilege for agents."

### 2.6 Data security posture

| Vendor | Relevance |
|---|---|
| **Cyera, Varonis, BigID, Sentra, Securiti** ○ | DSPM. Our `01 §22` is a well-served category |

### 2.7 SIEM / XDR with graph ambitions

| Vendor | Relevance |
|---|---|
| **Google SecOps** ✔ | Entity Context Graph, aliasing, risk analytics, data residency — see doc 06 |
| **Stellar Cyber** ✔ | Distributed sensors, edge filtering, multi-tenancy, residency — see doc 06 |
| **Microsoft Sentinel, Splunk (Cisco), Panther, Exabeam** ○ | Event-centric |

---

## 3. The honest scorecard

Overlook's three claims, tested against what shipped.

### Claim 1 — Residency is not blindness

```
   STATUS: UNCONTESTED

   No surveyed vendor offers a model where the vendor CANNOT read
   the customer's security data.

   Everyone offers residency. BloodHound Enterprise now offers
   full on-premises deployment — but that is self-hosting, which
   solves the problem by removing the vendor from the loop rather
   than by making the vendor blind. Different trade: the customer
   takes on all the operational burden and loses cross-tenant
   intelligence.

   Our model — vendor-hosted graph, customer-held key, vendor
   structurally unable to resolve tokens — remains unique.
```

### Claim 2 — Hybrid permission-closure depth

```
   STATUS: HEAVILY CONTESTED

   XM Cyber           hybrid paths from internet → cloud AI models →
                      on-prem databases → OT. Identity exposure across
                      AD, Entra AND cloud, with granted-vs-USED
                      permission analysis. That is our 02 §2 shipped.

   BloodHound Ent.    AD + Entra + AWS + Okta + GitHub attack paths,
                      with an on-premises deployment option.
                      That is our hybrid identity story shipped.

   What is still thin in both: Kubernetes RBAC bridging into cloud IAM,
   network reachability as a graph edge, data classification as a
   crown-jewel input, and the full escalation-primitive breadth across
   three clouds. But "we do hybrid identity paths" is no longer
   a differentiating sentence.
```

### Claim 3 — AI agents inside the closure

```
   STATUS: PARTIALLY OCCUPIED — but with a real gap

   OCCUPIED
     BloodHound Entra Agent ID extension — Copilot agents and AI
       identities, delegated identities, service principals, indirect
       paths to privileged access. Microsoft-managed agents.
     Zenity — agent inventory with ownership and dependency mapping,
       MCP servers and tools including shadow servers, least-privilege
       policy, runtime monitoring.
     Noma — agent and MCP access control across the enterprise.

   THE GAP THEY SHARE
     All three are oriented to MANAGED, enterprise-registered agents:
     Entra Agent ID, Copilot Studio, enterprise agent platforms,
     public MCP registries.

     None covers the UNMANAGED layer:
       - MCP servers configured on a developer's laptop
         (~/.claude/, ~/.cursor/, claude_desktop_config.json)
       - self-hosted agent frameworks a team wrote themselves
       - local model runtimes (ollama, vLLM, LM Studio)
       - IDE assistant extensions and what they can reach
       - the credentials those local configurations hold
       - RAG retrieval identity broader than its query audience

     That is the layer where a developer's laptop holds an MCP server
     with a GitHub token that reaches production — and no registry
     knows it exists.
```

---

## 4. What is actually left

Stripping out everything occupied, three things remain genuinely unoccupied:

```
  1. VENDOR BLINDNESS
     A vendor-hosted exposure graph the vendor cannot read.
     Nobody. Not contested at all.

  2. THE UNMANAGED AI LAYER
     Local MCP servers, self-hosted agents, local models, IDE
     assistants — and the credentials and reachability they carry.
     Everyone else starts from a registry. This layer has no registry.

  3. THE COMBINATION AT BREADTH
     BloodHound is identity-only. XM Cyber is vendor-hosted.
     Zenity/Noma are AI-only. Wiz is cloud-only and now Google-owned.
     Nobody combines: hybrid identity + cloud + data + network + the
     unmanaged AI layer + a privacy boundary.

     Caveat: #3 is exactly the 730-dev-day problem. It is what a
     funded team builds, not what one person builds.
```

### 4.1 The market opening nobody planned

Google now owns **Wiz + SCC + Chronicle**. That is the most complete security graph stack in the industry, under one vendor, who reads everything.

That creates a real, structural opening:

```
   Customers who will NOT consolidate on Google security:
     - AWS-primary and Azure-primary enterprises
     - anyone with a multi-cloud mandate
     - regulated institutions in jurisdictions wary of US hyperscalers
     - organisations already uncomfortable with Google's data posture
     - EU / India / Gulf / Indonesia sovereignty requirements

   For those buyers, "vendor-neutral AND vendor-blind" is not a
   feature list — it is the whole reason to take the meeting.
```

---

## 5. Feasible positions

Four honest options, given: **one person, Go, Postgres, no code yet.**

### Option A — The unmanaged AI exposure map

```
  WHAT      Discover and map the AI layer nobody has a registry for:
            local MCP servers, self-hosted agents, local models, IDE
            assistants, the credentials they hold, and what those
            credentials reach in cloud IAM.

  UNIQUE    Yes. Zenity, Noma and BloodHound all start from managed
            registries. This layer is invisible to all of them.

  BUILD     thin agent (Go, userland, no kernel) — config file parsing
            + 3-4 connectors (AWS, Entra, GitHub) for the reachability half
            + small graph + escalation matching
            NO gateway, NO DSPM, NO network, NO response

  EFFORT    4-6 months solo, realistically

  RISK      Small market on its own. Might be a feature, not a company.
            Zenity or Noma could add laptop discovery in a quarter.

  DEMO      "This developer's laptop runs an MCP filesystem server
             rooted at /Users/x/work, and a GitHub token in that config
             assumes a role that reaches your production database.
             Nothing in your enterprise registry knows this exists."
```

### Option B — Privacy-first identity exposure for sovereign markets

```
  WHAT      The full IAM closure (AD + Entra + cloud), deployed with
            the Edge/tokenization architecture, sold specifically to
            buyers who cannot use Wiz, BloodHound SaaS or XM Cyber.

  UNIQUE    The architecture is. The findings are not — BloodHound and
            XM Cyber produce similar output for buyers who can use them.

  BUILD     Edge Collector + Security Fact + tokenization + de-tokenization
            + AD/Entra/AWS connectors + permission closure
            + escalation primitives + path engine + minimal UI

  EFFORT    12-18 months solo. This is the doc 01 + 02 core.

  RISK      Competing on architecture against products with years of
            content depth. BloodHound's on-prem option partially
            neutralises the "can't use SaaS" argument.

  DEMO      "Your Overlook SaaS instance holds tokens. Here is the
             outbound inspector proving it. We could not read your
             graph if we wanted to."
```

### Option C — Open-source the escalation primitive engine

```
  WHAT      Publish the escalation primitive catalog and closure engine
            as open source. The "known privilege escalation paths across
            AWS, Azure, GCP, K8s, AD" as a community-maintained,
            machine-readable dataset with a reference evaluator.

  UNIQUE    Partially. Individual catalogs exist (Rhino Security's AWS
            privesc research, BloodHound's AD edges, various GCP lists).
            A unified, versioned, multi-cloud, machine-readable catalog
            with a working evaluator does not.

  BUILD     content + a Go evaluator. No collector, no SaaS, no UI.

  EFFORT    2-4 months solo for a credible v1

  RISK      Not a business by itself.

  UPSIDE    Highest credibility-per-effort of any option. Becomes the
            content moat referenced in 01 §17 and 02 §7. Community
            contributes the long tail. A commercial product can be
            built on top later, and the catalog makes that product
            immediately deeper than a from-scratch competitor.
```

### Option D — The full platform

```
  WHAT      Docs 01-05 as written.
  EFFORT    ~730 dev-days of connectors alone. 3+ years solo.
  VERDICT   Not feasible as a solo build. Keep as the target
            architecture that the feasible options grow toward.
```

### 5.1 The combination worth considering

A and C are complementary and both solo-feasible:

```
   C first  (2-4 months)   escalation primitive engine, open source
                            → credibility, content moat, a working
                              closure evaluator you need anyway

   A second (4-6 months)   the unmanaged AI layer, built on top of C
                            → the reachability half of Option A IS
                              the closure engine from C
                            → a demo nobody else can produce

   Together this is ~9 months solo, produces something genuinely
   unoccupied, and leaves the Edge/tokenization architecture (Option B)
   as the deployment model to add when there is a buyer who needs it —
   rather than building the privacy machinery before anyone has asked.
```

The argument against building Option B first: the privacy architecture is the most engineering-expensive part of the design, and its value is entirely conditional on a buyer who cannot use the alternatives. Building it before finding that buyer is a large bet on a market hypothesis. Building A+C first produces something demonstrable, then B becomes a deployment option for the customers who require it.

---

## 6. What this changes in the other documents

```
  01 §2.3   Claim 2 must be softened — XM Cyber and BloodHound ship
            hybrid identity paths. Claim 3 must be narrowed to the
            UNMANAGED layer specifically.
  01 §2.5   Add BloodHound Enterprise, Veza, Zenity and Noma to the
            landscape table. Note the Google/Wiz acquisition.
  02        Unaffected — the IAM depth is still correct and still needed
            by every option.
  03        Connector counts unaffected, but Options A and C need
            far fewer than 118.
  04, 05    Unaffected as architecture; scope depends on option chosen.
```

---

## 7. Open questions

```
  Q1  Which option? A+C, B, or something else.
  Q2  Is there an identified buyer for the privacy architecture, or is
      that a hypothesis? The answer changes Option B's risk profile
      completely.
  Q3  Is MSSP a channel? (carried over from doc 06, still unanswered)
  Q4  Verify the ○ items before any external use.
  Q5  Does BloodHound Enterprise's on-prem option actually neutralise
      the sovereignty argument, or do those buyers still object to
      self-hosting the operational burden?
```

---

*End of document.*

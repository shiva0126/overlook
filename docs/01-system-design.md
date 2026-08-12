# Overlook — Complete System Design

**Version:** 0.1 (design draft)
**Date:** 2026-08-12
**Status:** For brainstorming. Nothing here is implemented.

---

## How to read this document

This document exists to answer one question: **if I had to build Overlook, what exactly would I build, and how would data move through it?**

It is organised in seven parts:

| Part | Chapters | What it covers |
|---|---|---|
| I — Foundations | 1–4 | The mental model. What Overlook is, what problem it solves, the three planes. |
| II — The Spine | 5–9 | The data contracts everything else depends on: Security Facts, entities, tokenization, entity resolution, the TrustGraph. |
| III — Components | 10–18 | Internal architecture of the Edge Node, Agent, AI Gateway, and SaaS. |
| IV — Intelligence | 19–23 | Correlation, attack paths, risk, change, blast radius. |
| V — AI Security | 24–29 | How the AI-specific capabilities actually work. |
| VI — Action | 30–31 | Response and Break Attack Path. |
| VII — Operations | 32–38 | Investigation UX, self-security, product surfaces, scale, open decisions, pre-mortem, build model. |

Chapters 8, 20, 27 and 32 contain the four **end-to-end traces** — a single piece of data followed from source to screen. If you only read four chapters, read those.

---

# PART I — FOUNDATIONS

---

## 1. What Overlook is

### 1.1 The one-sentence definition

> Overlook is a **privacy-preserving exposure intelligence platform**. It builds a graph of everything in your environment and the relationships between them, then answers: *how can an attacker — or an AI agent — reach the things that matter?*

### 1.2 The mental model: a map, not a camera

The clearest way to hold Overlook in your head is the difference between a **map** and a **camera**.

A **camera** records what happened. That is a SIEM, an EDR, an NDR. It stores events, searches them, alerts on patterns. Its unit of work is the *event*. Its cost model is *volume of data stored*. Its failure mode is *drowning in alerts*.

A **map** records what is *possible*. That is Overlook. It stores entities and the relationships between them, then computes reachability. Its unit of work is the *relationship*. Its cost model is *number of entities*. Its failure mode is *a stale or incomplete map*.

This distinction drives every architectural decision in this document:

- Overlook does **not** store raw logs long-term. It reads them, extracts relationships, and discards them.
- Overlook does **not** alert on events. It reports **exposure** — a condition that persists until someone fixes it.
- A finding in Overlook has a **lifetime**, not a **timestamp**. "This path has existed for 41 days" is a more useful statement than "this happened at 04:12."
- Overlook does not need every event. It needs *enough* events to establish that a relationship exists. Sampling is acceptable. This is why the privacy model works at all.

### 1.3 The three questions Overlook answers

Every feature must serve one of these:

1. **What do I have?** — Asset, identity, data, and AI inventory. Unified, deduplicated, across cloud and on-prem.
2. **How is it all connected?** — The TrustGraph. Who can access what, through what, with what privilege.
3. **What should I fix first?** — Attack paths ranked by what they can reach, and the single change that breaks the most paths.

If a proposed feature doesn't serve one of those three, it belongs in a different product.

### 1.4 What Overlook is explicitly not

This list is a scope defence. Refer back to it whenever the roadmap starts expanding.

| Not | Why not | What we do instead |
|---|---|---|
| A SIEM | Log storage is a commodity business with brutal economics; incumbents are entrenched | We read logs, extract facts, discard raw data |
| An EDR | Requires kernel drivers, a detection research team, and 24/7 threat intel | We read the customer's existing EDR via API; our agent is thin |
| A SOAR | Requires hundreds of integrations and a workflow engine | We do a handful of high-value, tightly-scoped response actions |
| An NDR | Requires packet capture and traffic decryption at scale | We consume flow metadata and firewall configs for *reachability*, not detection |
| A vulnerability scanner | Commodity; customers already have one | We ingest their scanner's output as graph properties |
| A DLP | Requires content inspection everywhere | We inspect content in one place — the AI Gateway — for one purpose |

The pattern: **Overlook consumes other tools' output and turns it into relationships.** The tools produce findings; Overlook produces *context and consequence*.

### 1.5 The positioning contradiction (resolved)

The original architecture brief says Overlook should not be positioned as "SIEM + EDR + SOAR + CSPM + DSPM + ASPM + NDR + AI DLP" — and then lists 25 components that are essentially that list.

The resolution: those 25 items are **capability areas of the graph**, not products. CSPM in Overlook is not a CSPM product; it is *the cloud connector set plus the misconfiguration rules that generate cloud nodes and edges*. DSPM is not a DSPM product; it is *the data classification pass that attaches a sensitivity label to DATASTORE nodes*.

Stated precisely:

> Overlook has one product — the TrustGraph. Everything else is a source of nodes and edges for it.

Hold to that and the scope stays sane. Drift from it and you are building eight products with one engineering team.

---

## 2. The problem being solved

### 2.1 The customer's actual situation

A mid-to-large enterprise in 2026 typically has:

- 3 cloud providers, 40–200 cloud accounts/subscriptions/projects
- An on-prem data centre that is not going away
- 15–40 security tools, each with its own console and its own idea of what an "asset" is
- 60,000–400,000 findings open across those tools
- An identity estate spanning AD, Entra ID, Okta, three cloud IAMs, Kubernetes RBAC, and thousands of service accounts nobody owns
- And now, an explosion of AI: staff pasting into ChatGPT, developers running Copilot, teams building agents with production credentials, MCP servers appearing on laptops

They cannot answer the simple question: **which of these findings can actually get someone to the payroll database?**

### 2.2 Why existing tools don't answer it

- **Per-tool consoles** see one domain. The cloud tool doesn't know the AD trust. The identity tool doesn't know the firewall rule.
- **CNAPPs** are cloud-only. The attack path stops at the VPN. Real attack paths cross on-prem/cloud boundaries constantly.
- **Attack path tools** exist but require shipping everything to the vendor's cloud.
- **AI security tools** are standalone dashboards. They can tell you a prompt contained a secret. They cannot tell you that the agent which received it can assume a role that reaches production.

### 2.3 The two wedges

Overlook has exactly two defensible positions. Everything else is table stakes.

**Wedge 1 — Privacy-preserving architecture.**
The analysis happens inside the customer's environment. Only derived, tokenized facts leave. This wins deals that are structurally unwinnable for SaaS-only competitors: regulated banking, defence, healthcare, government, and any jurisdiction with data-residency law (EU, India, Gulf states, Indonesia, China).

This is not a feature. It is an architecture. It cannot be retrofitted by a competitor without rewriting their platform, which is why it is defensible.

**Wedge 2 — AI as a first-class citizen of the exposure graph.**
`AI_AGENT` and `MCP_SERVER` are node types alongside `ROLE` and `DATASTORE`. This lets Overlook answer questions no standalone AI-security tool can:

- *This developer has read-only AWS. But the agent they can prompt is admin. That is a privilege escalation path with no CVE and no misconfiguration — it is an architecture problem.*
- *This MCP server is on a laptop, reads `/finance/`, and the agent using it can send external email.*
- *An indirect prompt injection in a document indexed by this RAG app reaches an agent whose service account can modify production.*

### 2.4 The demo that sells the product

Every good security product has one slide. Overlook's is the **AI Privilege Gap** (Chapter 29):

```
  Priya (Developer)
     |
     |  direct AWS permission:  s3:GetObject  (read-only)
     |
     +--> can prompt --> DevOps-Agent
                              |
                              | runs as: svc-devops-ai
                              |
                              +--> AWS role: DevOpsAdmin (Administrator)
                                        |
                                        +--> RDS: prod-payments-db
                                                  |
                                                  +--> 4.2M records, PII + PCI

  FINDING:  AI PRIVILEGE GAP
  User privilege:  READ
  Effective privilege via agent:  ADMIN
  Severity: CRITICAL
  Break this path:  remove iam:PassRole from svc-devops-ai  (breaks 7 other paths too)
```

Everything in this document exists to make that screen possible, accurate, and trustworthy.

---

## 3. The three planes

Overlook decomposes into three planes with different trust levels, different failure modes, and different scaling characteristics. Keeping them mentally separate prevents most architectural mistakes.

```
+---------------------------------------------------------------+
|  ACTION PLANE                                                  |
|  Response commands, approvals, signing, execution, rollback    |
|  Trust: highest. Failure mode: breaks production.              |
|  Scale: tens of operations per day.                            |
+---------------------------------------------------------------+
                              ^
                              |  signed, approved, TTL-bounded
                              |
+---------------------------------------------------------------+
|  INTELLIGENCE PLANE  (Overlook SaaS)                           |
|  TrustGraph, correlation, attack paths, risk, investigation    |
|  Trust: medium. Sees tokens and relationships, never raw data. |
|  Scale: millions of nodes, tens of millions of edges/tenant.   |
+---------------------------------------------------------------+
                              ^
                              |  Security Facts (tokenized, signed)
                              |
+---------------------------------------------------------------+
|  COLLECTION PLANE  (Edge Node, Agent, AI Gateway)              |
|  Ingestion, parsing, normalization, resolution, local analysis |
|  Trust: highest. Sees everything in the clear.                 |
|  Scale: billions of events/day per large customer.             |
+---------------------------------------------------------------+
```

### 3.1 Why the boundary sits where it does

The single most important line in the system is the boundary between the Collection Plane and the Intelligence Plane. Data crossing upward is reduced by roughly **10,000:1** in volume and **100%** in identifiability.

A concrete illustration. One day at a mid-size customer:

| Stage | Volume | Contains |
|---|---|---|
| Raw events ingested at Edge | ~2.4 TB | Everything. Usernames, IPs, hostnames, queries, file paths |
| After parsing + normalization | ~600 GB | Same information, structured |
| After entity resolution + dedup | ~40 GB | Unique entities and observations |
| Security Facts generated | ~180 MB | Relationships only |
| **Sent to SaaS after privacy gate** | **~12 MB/day steady state** | Tokens, edge types, confidence, timestamps |

That last number is the product's economics *and* its privacy story in one figure. A customer's entire security posture, updated daily, is a few megabytes — because a *relationship* is tiny compared to the evidence that proves it.

### 3.2 What each plane may know

| | Collection Plane | Intelligence Plane | Action Plane |
|---|---|---|---|
| Raw log lines | Yes | Never | Never |
| Usernames, emails, hostnames | Yes | Never (tokens only) | Token + local resolution |
| File contents, prompt text | Yes | Never | Never |
| Credentials | Yes (vaulted) | Never | Never |
| Entity relationships | Yes | Yes | Yes |
| Risk scores, confidence | Computes | Computes | Reads |
| Attack paths | Partial (local only) | Yes (global) | Reads |
| Cross-customer intel | No | Yes (aggregate only) | No |

### 3.3 Failure isolation

Each plane fails independently and degrades rather than stops:

- **SaaS down** → Edge continues collecting and buffering. Local UI still serves inventory and local findings. No global attack paths, no response.
- **Edge down** → Agents buffer locally (bounded). SaaS graph goes stale, marked with a staleness indicator per source. Nothing is deleted; the graph ages rather than emptying.
- **Agent down** → That host's runtime context ages out; the host remains in the graph from API-derived sources.
- **AI Gateway down** → *Critical decision:* fail-open or fail-closed. Default fail-open (AI traffic passes uninspected, a gap is recorded in the graph) because failing closed breaks the customer's business. Fail-closed must be an explicit per-policy opt-in.

---

## 4. Vocabulary

Precise terms, used consistently for the rest of the document.

| Term | Definition |
|---|---|
| **Entity** | A thing that exists: a user, a role, a host, a bucket, an AI agent. Becomes a node. |
| **Canonical Key** | The globally-stable, source-independent string that identifies an entity. See 6.2. |
| **Observation** | A single sighting of an entity or relationship from one source at one time. Raw material. |
| **Security Fact** | The unit of information that crosses from Edge to SaaS. A tokenized, signed, evidence-referenced assertion about entities or relationships. See Chapter 5. |
| **Edge (graph)** | A relationship between two nodes: `CAN_ASSUME`, `CAN_READ`, `CONNECTS_TO`. |
| **Edge Node** | The customer-side appliance. Unfortunate collision with "edge (graph)" — context disambiguates; this document says "Edge Node" for the appliance. |
| **TrustGraph** | The unified graph in SaaS. Nodes + edges + properties + time. |
| **Attack Path** | An ordered sequence of graph edges from a starting condition to a crown jewel. |
| **Crown Jewel** | An asset the customer designates as business-critical. Paths are scored by which crown jewels they reach. |
| **Blast Radius** | Everything reachable *from* a compromised node. The inverse of an attack path. |
| **Choke Point** | An edge appearing in many distinct paths. Removing it breaks all of them. The basis of Break Attack Path. |
| **Privacy Gate** | The Edge Node component that decides what may leave and strips/tokenizes everything else. |
| **Token** | A deterministic, tenant-scoped pseudonym for a sensitive value. See Chapter 7. |
| **Evidence Reference** | A hash pointing at raw evidence retained *at the Edge*, fetchable on demand by an authorised analyst. |

---

# PART II — THE SPINE

Part II defines the data contracts. These are the highest-leverage decisions in the entire system: they are extremely expensive to change once connectors and the graph depend on them. Everything in Part III is replaceable. Nothing in Part II is.

---

## 5. The Security Fact

### 5.1 Why it is the centre of the system

The Security Fact is the **only** thing that crosses from customer environment to Overlook SaaS. That makes it:

- The **privacy boundary** — if it isn't in the Fact schema, it cannot leak.
- The **compatibility boundary** — Edge and SaaS version independently; the Fact is the contract between them.
- The **audit boundary** — a customer auditor reviews the Fact schema to know exactly what leaves.
- The **cost boundary** — Facts are what you store and index in SaaS.

**Therefore: define the Fact schema formally, version it, sign it, and publish it to customers.** A schema registry with backwards-compatibility enforcement is not over-engineering here; it is the thing that lets you ship Edge 2.4 to a customer still running SaaS features from 2.1.

### 5.2 Anatomy

```jsonc
{
  // --- identity of this fact ---
  "fact_id":      "01JC8Q2K7M4N5P6R7S8T9V0W1X",   // ULID, sortable by time
  "schema":       "overlook.fact.v1",
  "fact_type":    "RELATIONSHIP",                  // ENTITY | RELATIONSHIP | PROPERTY | FINDING | EVENT_SUMMARY
  "tenant_id":    "TNT-7742",
  "edge_node_id": "EDGE-ap-south-1-a",

  // --- what is being asserted ---
  "subject":   { "type": "IDENTITY", "token": "IDN-9f3a7c21e845b0d6" },
  "predicate": "CAN_ASSUME",
  "object":    { "type": "ROLE",     "token": "ROL-2b8e4f19a70c5d33" },

  // --- qualifiers that make the assertion useful ---
  "attributes": {
    "mechanism":        "aws_sts_assume_role",
    "conditions":       ["aws:MultiFactorAuthPresent=true"],
    "privilege_level":  "ADMIN",
    "is_transitive":    true,
    "hop_count":        1
  },

  // --- trust and lineage ---
  "confidence":  0.97,
  "sources":     ["aws.iam.get_role_policy", "aws.cloudtrail.assume_role_event"],
  "collection_method": "API_POLL",

  // --- time ---
  "first_seen":  "2026-06-02T09:14:22Z",
  "last_seen":   "2026-08-12T04:00:11Z",
  "observation_count": 148,
  "valid_until": "2026-08-19T04:00:11Z",           // staleness horizon

  // --- evidence, retained at the Edge, never sent ---
  "evidence": {
    "ref":        "sha256:8a1f...c4d2",
    "kind":       "iam_policy_document",
    "location":   "edge://EDGE-ap-south-1-a/evidence/2026/08/12/8a1f...c4d2",
    "retention_expires": "2026-11-10T00:00:00Z"
  },

  // --- integrity ---
  "signature": "ed25519:MEUCIQD..."
}
```

### 5.3 The five fact types

**`ENTITY`** — asserts a thing exists.
> `IDN-9f3a…` exists, type IDENTITY, subtype human_user, attributes {department: FINANCE, mfa_enabled: true, last_active: …}

**`RELATIONSHIP`** — asserts an edge between two entities. The most important type; this is what makes attack paths possible.
> `IDN-9f3a…` `CAN_ASSUME` `ROL-2b8e…`

**`PROPERTY`** — asserts a fact *about* an entity that isn't a relationship.
> `DST-1c4b…` has {classification: PII+PCI, record_count_bucket: "1M-10M", encryption: AES256, public: true}

Note `record_count_bucket`, not `record_count`. Exact counts can be identifying and are rarely decision-relevant. **Bucket everything that doesn't need precision.** This principle repeats throughout the design.

**`FINDING`** — asserts a condition worth attention, produced by a local rule.
> Rule `AWS-S3-PUBLIC-SENSITIVE` fired on `DST-1c4b…`, severity HIGH, evidence `sha256:…`

**`EVENT_SUMMARY`** — an aggregate of behaviour over a window. Never individual events.
> `IDN-9f3a…` performed `PRIVILEGED_ACCESS` to `AST-77c2…` 14 times between 09:00–17:00, first 09:14, last 16:52, anomaly_score 0.81

### 5.4 Lifecycle

```
   Observation(s) at Edge
            |
            v
   [ Fact Builder ]  -- merges repeat observations into one fact,
            |           bumping last_seen and observation_count
            v
   [ Privacy Gate ]  -- tokenize, bucket, strip, validate against schema
            |
            v
   [ Signer ]        -- Ed25519, Edge Node key
            |
            v
   [ Outbound Queue ] -- durable, ordered, compressed batches
            |
            v
   ================ mTLS 443 ================
            |
            v
   [ SaaS Ingest ]   -- verify signature, verify schema, dedupe by fact_id
            |
            v
   [ Graph Writer ]  -- upsert node/edge, update temporal fields
            |
            v
   [ Change Feed ]   -- emits diff events for the correlation engine
```

### 5.5 Idempotency and dedup

Facts must be **idempotent on replay**. After a network partition the Edge will resend; after an Edge restart it may rebuild from local state. The rule:

- `fact_id` is a ULID, unique per emission — used for transport-level dedup only.
- The **semantic identity** of a fact is `hash(tenant_id, fact_type, subject.token, predicate, object.token, canonical(attributes))`.
- SaaS upserts on semantic identity, taking `max(last_seen)`, `min(first_seen)`, `sum(observation_count)` where sensible, and the highest-confidence source.

Getting this wrong produces duplicate edges, which inflate path counts and destroy trust in the numbers. Design it in from the first line of code.

### 5.6 Schema evolution rules

1. Fields may be **added**; never removed or retyped within a major version.
2. Unknown fields are **preserved, not rejected**, by SaaS ingest (forward compatibility — a newer Edge talking to older SaaS must not break).
3. Predicate vocabulary is a **closed enum**, versioned. New predicates require a schema version bump. This prevents connector authors inventing `can_read_maybe`.
4. Every schema version ships with a **conformance test suite** that connectors must pass in CI.
5. The current schema is **published to the customer**, in the local UI, as a human-readable document. This is a sales asset: "here is literally everything that leaves your network."

---

## 6. The entity model

### 6.1 The core types

```
  IDENTITY LAYER            COMPUTE LAYER           DATA LAYER
  ---------------           --------------          -----------
  IDENTITY                  ASSET                   DATASTORE
   +- USER                   +- HOST                 +- DATABASE
   +- SERVICE_ACCOUNT        +- VM                   +- BUCKET
   +- WORKLOAD_IDENTITY      +- CONTAINER            +- FILE_SHARE
  GROUP                      +- FUNCTION             +- VECTOR_DATABASE
  ROLE                       +- CLUSTER             DATA_CLASS
  CREDENTIAL                APPLICATION
   +- API_KEY               REPOSITORY
   +- SSH_KEY               PIPELINE
   +- CERTIFICATE           ARTIFACT

  NETWORK LAYER             AI LAYER                POSTURE LAYER
  -------------             --------                -------------
  NETWORK                   AI_APPLICATION          VULNERABILITY
  SUBNET                    AI_MODEL                MISCONFIGURATION
  FIREWALL                  AI_AGENT                SECRET
  SECURITY_GROUP            MCP_SERVER              SECURITY_CONTROL
  LOAD_BALANCER             MCP_TOOL                COMPLIANCE_REQUIREMENT
  DNS_NAME                  PROMPT_EVENT
  FLOW_AGGREGATE            RAG_APPLICATION
```

### 6.2 Canonical keys — the hardest unglamorous problem

Two connectors see the same thing and call it different names. AWS calls it `arn:aws:iam::123456789012:user/priya`. Okta calls it `00u1a2b3c4d5`. AD calls it `CN=Priya S,OU=Eng,DC=corp,DC=local`. The EDR calls it `CORP\priyas`. The VPN log calls it `priya.s@corp.com`.

They are one person. If Overlook doesn't work that out, the graph is five disconnected fragments and every attack path stops at a source boundary.

**The Canonical Key is the answer.** Rules:

1. Every entity type has an ordered list of **identity attributes**, most authoritative first.
2. The canonical key is derived from the highest-confidence attribute available.
3. Keys are **normalized** before use: lowercase, trim, strip display formatting, resolve aliases.

```
IDENTITY (human):
  1. Verified corporate email        -> "email:priya.s@corp.com"
  2. IdP immutable object ID         -> "okta:00u1a2b3c4d5"
  3. AD objectGUID                   -> "adguid:8f14e45f-ceea-467a-9d29-2b1e4c4e1d3f"
  4. UPN                             -> "upn:priya.s@corp.local"
  5. SAM account name + domain       -> "sam:corp\priyas"

ASSET (host):
  1. Cloud instance ID               -> "aws:i-0abc123def456"
  2. Hardware UUID / SMBIOS          -> "hwuuid:4C4C4544-0032..."
  3. FQDN                            -> "fqdn:web-prod-04.corp.local"
  4. MAC address set                 -> "mac:00:1a:2b:3c:4d:5e"
  5. IP + time window                -> "ip:10.4.2.17@2026-08-12"   (weakest; time-bounded)

AI_AGENT:
  1. Registered agent ID in platform -> "bedrock-agent:AGT12345"
  2. Service account + app name      -> "svcapp:svc-devops-ai/devops-agent"
  3. Process signature + host        -> "proc:ollama@AST-77c2"
```

Note the descending confidence. An entity resolved by IP alone gets low confidence and a short validity window; one resolved by cloud instance ID is near-certain. **Confidence propagates into every downstream path score.** A path built on weak resolution must be visibly weaker in the UI, or analysts will lose trust the first time a merge is wrong.

### 6.3 The predicate vocabulary

Closed set. Adding to it requires a schema version bump and a written justification, because every predicate multiplies attack-path engine complexity.

| Predicate | Meaning | Traversable in paths? |
|---|---|---|
| `OWNS` | Administrative ownership | No (metadata) |
| `MEMBER_OF` | Group/role membership | Yes |
| `CAN_ASSUME` | Can become this identity | Yes — highest value |
| `CAN_READ` | Can read data from | Yes |
| `CAN_WRITE` | Can modify data in | Yes |
| `CAN_EXECUTE` | Can run code on | Yes — highest value |
| `CAN_MODIFY` | Can change configuration of | Yes |
| `CAN_DEPLOY` | Can push code/artifacts to | Yes |
| `CONNECTS_TO` | Observed network connection | Yes (weak) |
| `ROUTES_TO` | Network reachability exists | Yes |
| `EXPOSES` | Makes reachable from a zone | Yes |
| `RUNS_ON` | Workload-to-host placement | Yes (inherits host compromise) |
| `CONTAINS` | Containment/composition | Yes (downward) |
| `AUTHENTICATES_TO` | Credential relationship | Yes |
| `TRUSTS` | Federation / cross-account trust | Yes |
| `PROTECTS` | Control covering an asset | No (defensive; reduces edge weight) |
| `RETRIEVES_FROM` | RAG app to vector store | Yes |
| `INVOKES` | Agent to tool / model | Yes |
| `PROMPTED_BY` | Prompt event to identity | Yes |
| `HAS_VULNERABILITY` | Asset to CVE | No (property; enables edges) |
| `STORES` | Datastore to data class | No (property) |

### 6.4 Why `PROTECTS` doesn't traverse

A control does not create reachability; it reduces the probability of a path succeeding. Modelling it as a traversable edge produces nonsense paths ("attacker traverses the EDR to reach the server"). Instead, `PROTECTS` edges reduce the **weight** of adjacent traversable edges in path scoring. Chapter 21 covers the scoring maths.

---

## 7. Tokenization and the privacy boundary

### 7.1 The requirement

Three things must be true simultaneously, and they are in tension:

1. SaaS must never receive identifying values.
2. The **same entity** observed by **different Edge Nodes** must produce the **same token** — otherwise a customer with 5 Edge Nodes gets 5 disconnected graph fragments and cross-domain correlation, the entire product, fails.
3. An authorised analyst must be able to see the real name in the UI.

A random token satisfies (1) and fails (2) and (3). A reversible encryption with a vendor-held key satisfies (2) and (3) and fails (1) — and fails it in the way that matters, since the vendor can decrypt.

### 7.2 The design: deterministic, tenant-keyed HMAC

```
token = TYPE_PREFIX + "-" + truncate( HMAC-SHA256( tenant_key, canonical_key ), 16 bytes )

e.g.  canonical_key = "email:priya.s@corp.com"
      tenant_key    = <256-bit key, generated at tenant creation,
                       held by customer, escrowed in customer KMS/HSM,
                       NEVER transmitted to Overlook SaaS>

      token = "IDN-9f3a7c21e845b0d6"
```

Properties this gives us:

- **Deterministic** — every Edge Node with the tenant key produces the same token for the same entity. Fragments merge automatically. Requirement (2) satisfied.
- **Irreversible without the key** — SaaS cannot compute `canonical_key` from the token. Requirement (1) satisfied.
- **Not brute-forceable** — this is the subtle part. The keyspace of email addresses is small; without the tenant key, brute force is infeasible, but *if the key ever leaked*, an attacker could dictionary-attack every token. Hence: the key never leaves the customer environment, and each tenant's key is independent, so a compromise is contained to one customer.
- **Rotatable** — with pain. Rotation re-tokenizes the whole graph. Design a dual-token transition window (Edge emits both old and new token for one sync cycle, SaaS merges) rather than pretending rotation never happens.

### 7.3 Key management

```
  Tenant creation (in customer environment, during Edge enrollment)
        |
        v
  Edge generates 256-bit tenant_key using OS CSPRNG
        |
        +--> Wrapped with customer KMS/HSM key (envelope encryption)
        |         |
        |         v
        |    Stored in customer's KMS. Overlook has no access.
        |
        +--> Distributed to additional Edge Nodes via customer-controlled
             enrollment (operator pastes a one-time bootstrap token, or
             Edge Nodes fetch from shared KMS with their own IAM identity)
        |
        v
  Overlook SaaS receives:  nothing.
                           It sees only the tenant_id and the tokens.
```

The customer can, at any time, verify this claim by inspecting the Edge Node's outbound queue through the local UI. **Build that inspection UI early** — it converts the privacy claim from a promise into a demonstrable fact, which is worth more in a procurement review than any whitepaper.

### 7.4 De-tokenization: how the analyst sees real names

This is the gap in the original brief and the detail most likely to be discovered late and painfully. Nobody triages a graph of `IDN-9f3a7c21e845b0d6`.

**The design: de-tokenization happens in the analyst's browser, never on Overlook servers.**

```
   Analyst's browser (authenticated to Overlook SaaS)
        |
        | 1. Loads attack path view. SaaS returns the graph
        |    with tokens only:  IDN-9f3a... -> ROL-2b8e... -> DST-1c4b...
        |
        | 2. Browser extracts the list of tokens on screen (~40 tokens)
        |
        | 3. Browser calls the CUSTOMER'S Edge Node directly:
        |       POST https://overlook-edge.corp.local/api/v1/resolve
        |       Authorization: <SaaS-issued, short-lived, user-scoped JWT>
        |       Body: { tokens: ["IDN-9f3a...", "ROL-2b8e...", ...] }
        |
        | 4. Edge validates the JWT (signature, audience, expiry, user's
        |    RBAC scope), looks up its local token->value mapping table,
        |    and returns display names:
        |       { "IDN-9f3a...": { name: "Priya S", email: "priya.s@corp.com",
        |                          dept: "Engineering" }, ... }
        |
        | 5. Browser renders the graph with real names.
        v
   Overlook SaaS never sees step 3 or 4.
```

**Consequences you must accept:**

- The Edge Node is now a **production, low-latency, always-available service** with an SLA — not a batch collector. Chapter 10 sizes it accordingly.
- The analyst's browser must reach the Edge Node. Fine on corporate network/VPN; a problem for a SOC analyst at home. Mitigations: an Edge Node in a DMZ with mTLS client certs, or a customer-hosted reverse proxy, or a customer-operated relay. **Do not** solve this by relaying through Overlook SaaS — that would put plaintext names on our servers and destroy the entire premise.
- A **degraded mode** must exist: if the Edge is unreachable, the UI shows tokens with type icons and any non-identifying attributes (`IDENTITY · service account · Finance · admin privilege`). Surprisingly usable for triage; useless for action. Make the degraded state visually obvious.
- The mapping table at the Edge is the crown jewel of the deployment. Encrypted at rest, access-audited, and every resolve call logged with the requesting user, tokens requested, and timestamp — which, pleasantly, is also the customer's proof of who looked at what.

### 7.5 What is never tokenized, and what is never sent

Some values are sent in the clear because they carry no customer-identifying information and are essential for correlation across tenants:

**Sent in clear:** CVE IDs, cloud provider names, region names, public IP addresses of *external* services (`chatgpt.com`), well-known service identifiers, model names (`gpt-4o`), MCP tool names, misconfiguration rule IDs, port numbers, protocol names.

**Tokenized:** usernames, emails, hostnames, internal IPs, ARNs, resource names, group names, repository names, file paths, database names, bucket names, department names *(if the customer requests it)*.

**Never sent, in any form:** raw log lines, file contents, prompt text, response text, query text, credential values, certificate private keys, personal data of any kind, exact record counts, precise geolocation.

**Bucketed before sending:** record counts, data volumes, session durations, request rates, financial values. Bucketing is a privacy control that costs almost no analytical value — nobody makes a decision differently at 4,187,332 records versus "1M–10M".

---

## 8. Entity resolution

### 8.1 The problem restated

One person, five sources, five identifiers. One server, six sources, six identifiers. Resolution is the process of collapsing observations into entities. Get it right and the graph is a coherent map. Get it wrong in one direction (under-merge) and paths break silently. Get it wrong in the other (over-merge) and you tell a customer their intern is a domain admin.

**Under-merge is a missed finding. Over-merge is a false accusation.** Bias the algorithm toward under-merging, and make merges visible and reversible.

### 8.2 The algorithm

Three-stage: deterministic → probabilistic → graph-based.

**Stage 1 — Deterministic matching (high confidence, cheap).**
Direct match on a canonical key from the priority list. Two observations sharing `okta:00u1a2b3c4d5` are the same identity. Confidence 1.0. Handles ~70% of the work.

**Stage 2 — Probabilistic matching (medium confidence).**
Weighted-attribute scoring across candidates:

```
score = Σ (w_i × match_i)

  attribute            weight   match rule
  -------------------  ------   -----------------------------
  email local-part      0.30    exact after normalization
  display name          0.15    fuzzy (Jaro-Winkler > 0.92)
  employee ID           0.35    exact
  manager               0.10    resolved manager entity matches
  department            0.05    exact
  creation date         0.05    within 7 days

  merge if score >= 0.85
  quarantine for review if 0.65 <= score < 0.85
  do not merge below 0.65
```

The quarantine band matters. Those go to a **Resolution Review queue** in the local UI where an operator confirms or rejects. Every confirmation becomes a stored rule, so the same ambiguity is never asked twice. This is a real, ongoing operational task — build the queue as a first-class UI surface, not a debug page.

**Stage 3 — Graph-based reinforcement (evidence accumulation).**
If two candidate entities share many neighbours, that is evidence they are the same. `CORP\priyas` on host A and `priya.s@corp.com` in Okta both `MEMBER_OF` the same four groups and both authenticate to the same six assets → reinforcement raises the score.

Conversely, **negative evidence** blocks merges: two identities observed authenticating from geographically impossible locations within a short window, or two service accounts with explicitly conflicting owner attributes, are pinned apart.

### 8.3 Where resolution runs — and the multi-Edge trap

Resolution runs at the **Edge Node**, before tokenization. It cannot run in SaaS: SaaS only has tokens, and tokens of two unmerged identities are two different tokens. There is no way to discover from `IDN-9f3a…` and `IDN-4b2e…` that they are one person.

This creates a hard requirement:

> **All Edge Nodes in a tenant must resolve to the same canonical key for the same entity, or the graph fragments.**

Three mechanisms, all needed:

1. **Shared tenant key** — makes tokens identical *given* identical canonical keys.
2. **Canonical key priority rules are global** — shipped as tenant configuration, identical on every Edge Node, versioned. If Edge A prefers email and Edge B prefers SAM name for the same identity, they produce different tokens for the same person.
3. **A Resolution Directory** — a tenant-wide, customer-hosted mapping of `alternate_identifier -> canonical_key`, replicated between Edge Nodes. When Edge B sees `CORP\priyas` and has no local Okta connector, it consults the directory and learns the canonical key is `email:priya.s@corp.com`.

The Resolution Directory is a piece of infrastructure the original brief does not mention and which is **mandatory** for any multi-Edge deployment. Options for hosting it: elect a primary Edge Node; or a small dedicated customer-hosted service; or replicate via the customer's own datastore. Simplest workable v1: **designate one Edge Node as the resolution primary**, others query it, with a cached fallback if unreachable.

### 8.4 TRACE 1 — a person, from five sources to one node

Follow one identity all the way through.

```
T0   Okta connector polls /api/v1/users
     -> {id:"00u1a2b3", profile:{login:"priya.s@corp.com",
         firstName:"Priya", lastName:"S", department:"Engineering"}}

     Canonical key resolution:
       Priority 1 (verified corp email) -> "email:priya.s@corp.com"   MATCH
     Entity created: local_id=E-00417, canonical="email:priya.s@corp.com"
     Confidence 1.0.

T1   AD connector reads LDAP
     -> {objectGUID:"8f14e45f-...", sAMAccountName:"priyas",
         userPrincipalName:"priya.s@corp.local", mail:"priya.s@corp.com"}

     Priority 1 -> mail attribute -> "email:priya.s@corp.com"   MATCH (deterministic)
     Merged into E-00417.
     Aliases recorded: adguid:8f14e45f-..., sam:corp\priyas, upn:priya.s@corp.local
     Resolution Directory updated with all three aliases.

T2   AWS IAM connector reads users
     -> {UserName:"priya.s", Arn:"arn:aws:iam::123456789012:user/priya.s",
         Tags:[{Key:"email",Value:"priya.s@corp.com"}]}

     Tag gives email -> MATCH deterministic. Merged into E-00417.
     New edge observed:  E-00417 CAN_ASSUME arn:aws:iam::...:role/DevOpsAdmin

T3   EDR connector reports a logon
     -> {user:"CORP\priyas", host:"LT-4471", ts:"2026-08-12T09:14:22Z"}

     "CORP\priyas" is not a priority-1 key.
     Resolution Directory lookup: sam:corp\priyas -> email:priya.s@corp.com  MATCH
     Merged. New edge: E-00417 AUTHENTICATES_TO AST-LT-4471

T4   Overlook Agent on LT-4471 reports a process tree
     -> user "priyas", process Code.exe -> node.exe (Copilot extension)
        -> outbound TLS to api.githubcopilot.com

     Host context gives domain; sam:corp\priyas -> E-00417.
     New entity: AI_APPLICATION "GitHub Copilot"
     New edge: E-00417 USES AI_APPLICATION:github-copilot

T5   Fact Builder emits:

     ENTITY fact:
       token: IDN-9f3a7c21e845b0d6      (= HMAC(tenant_key,"email:priya.s@corp.com"))
       type: IDENTITY / human_user
       attributes: {department:"ENGINEERING", mfa:true, privileged:false,
                    source_count:5, resolution_confidence:1.0}

     RELATIONSHIP facts:
       IDN-9f3a... CAN_ASSUME    ROL-2b8e...   (conf 0.99)
       IDN-9f3a... AUTHENTICATES_TO AST-77c2...(conf 0.95)
       IDN-9f3a... USES          AIA-5d1f...   (conf 0.92)

T6   Privacy Gate strips:
       - "Priya S", "priya.s@corp.com", "CORP\priyas", "LT-4471",
         the ARN, the objectGUID — all replaced by tokens
       - the raw logon event — discarded, hash retained as evidence ref
       - department kept (customer policy allows; can be tokenized on request)

T7   Signed, queued, batched, compressed, sent over mTLS.

T8   SaaS graph now holds one node, five source attributions,
     three outbound edges, and zero knowledge of who Priya is.
```

Total bytes on the wire for this identity: roughly **900 bytes**. Total bytes read at the Edge to produce it: roughly **40 KB** of API responses and log lines.

---

## 9. The TrustGraph

### 9.1 Structure

The graph is **property-attributed and bitemporal**.

```
NODE
  token           IDN-9f3a7c21e845b0d6
  type            IDENTITY
  subtype         human_user
  tenant_id       TNT-7742
  properties      { department, privileged, mfa_enabled, ... }
  criticality     0..100          (crown-jewel weighting)
  first_seen      2026-06-02T09:14:22Z
  last_seen       2026-08-12T04:00:11Z
  removed_at      null
  sources         [okta, ad, aws_iam, edr, agent]
  confidence      1.0

EDGE
  from_token      IDN-9f3a7c21e845b0d6
  to_token        ROL-2b8e4f19a70c5d33
  predicate       CAN_ASSUME
  properties      { mechanism, conditions[], privilege_level, is_transitive }
  weight          0.12            (traversal cost; see 21.3)
  first_seen      2026-06-02T09:14:22Z
  last_seen       2026-08-12T04:00:11Z
  removed_at      null
  confidence      0.97
  evidence_ref    sha256:8a1f...c4d2
  edge_node_id    EDGE-ap-south-1-a
```

### 9.2 Bitemporality — and why it is not optional

The brief lists "Change Intelligence" as a component. That single phrase forces a major design decision, so state it explicitly:

> The TrustGraph stores **edge lifecycle**, not snapshots.

Every node and edge carries `first_seen`, `last_seen`, and `removed_at`. Nothing is ever hard-deleted; edges are tombstoned. This gives, for free:

- *"This path has existed for 41 days"* — path age, which is a better prioritisation signal than severity.
- *"This admin grant appeared 3 hours ago"* — new-privilege detection, one of the highest-signal alerts in security.
- *"Show me the graph as it was last Tuesday"* — investigation and audit.
- *"Did our fix actually break the path?"* — verification, which closes the loop on response.

The alternative — daily snapshots — costs vastly more storage and answers fewer questions. Take the lifecycle model.

The cost: **absence detection is hard**. If a connector fails for a day, edges are not "removed" — they are unobserved. Distinguish these:

```
edge.removed_at is set  ONLY when a source that WOULD have reported the
edge ran successfully and did NOT report it.

Requires per-source "coverage windows":
   source aws.iam ran at 04:00, succeeded, enumerated 412 roles.
   -> any IAM edge not in that enumeration is genuinely removed.

   source aws.iam ran at 04:00, FAILED after 90 roles.
   -> mark partial. Do NOT tombstone anything. Mark affected
      subgraph as STALE with the failure reason.
```

This is a common and damaging bug in graph products: a broken connector silently "fixes" thousands of findings. Build coverage windows from day one, and surface staleness in the UI at the point of use — a greyed-out badge on the path itself, not buried in a health page.

### 9.3 Scale

Per large tenant, planning targets:

| | Count | Notes |
|---|---|---|
| Nodes | 2–8 million | Dominated by cloud resources and identities |
| Edges | 20–120 million | IAM permission closure dominates |
| Edge churn | 1–5% / day | Ephemeral compute, short-lived containers |
| Crown jewels | 50–500 | Customer-designated |
| Attack paths computed | 10k–500k | Before dedup and ranking |
| Paths shown to a human | < 50 | After choke-point collapsing |

That last row is the product's real job. Computing 500,000 paths is an engineering problem. Presenting fewer than 50 that a human will act on is the *product* problem, and it is harder. Chapters 20–21 address it.

### 9.4 Storage choice

Three viable approaches, with an honest assessment:

| Approach | Pros | Cons | Verdict |
|---|---|---|---|
| Native graph DB (Neo4j, TigerGraph) | Natural queries, mature traversal | Licensing cost, multi-tenant isolation is awkward, hard to scale writes | Good for prototype, risky as the long-term core |
| Relational + recursive CTEs (Postgres) | Operationally boring, cheap, transactional, easy multi-tenancy | Traversal performance degrades past ~10M edges | Best v1 choice |
| Purpose-built in-memory graph over columnar storage | Fastest paths, full control | Most engineering effort | Where you end up at scale |

**Recommendation: start on Postgres** with an adjacency table plus materialized transitive-closure tables for the expensive predicates (`CAN_ASSUME`, `MEMBER_OF`). Design the graph access layer as an interface from day one so the engine can be swapped without touching the analytics above it. Do not spend v1 engineering budget on a bespoke graph engine; spend it on connectors and resolution quality, which are what actually determine whether the product works.

---

# PART III — THE COMPONENTS

---

## 10. Overlook Edge Analytics Node

### 10.1 The honest framing

The original brief lists 28 responsibilities for the Edge Node. Read as a single process, that is unbuildable. Read correctly, it is a **distributed system that ships as one appliance image and can be split across machines as load demands**.

Design it as roles from day one. Ship it as one image for small customers, where all roles run co-resident. Let large customers scale roles independently. If you build a monolith first, the first 50,000-host customer forces a rewrite.

### 10.2 The five roles

```
+-------------------------------------------------------------------+
|                     OVERLOOK EDGE NODE                            |
|                                                                   |
|  +-----------------+   +-----------------+   +-----------------+  |
|  |  R1 COLLECTOR   |   |  R2 PROCESSOR   |   |  R3 SCANNER     |  |
|  |                 |   |                 |   |                 |  |
|  | syslog listener |   | parse           |   | DSPM crawling   |  |
|  | netflow listener|   | normalize       |   | classification  |  |
|  | webhook rx      |-->| enrich          |   | CSPM evaluation |  |
|  | API pollers     |   | resolve         |<--| posture rules   |  |
|  | agent gateway   |   | fact building   |   |                 |  |
|  |                 |   |                 |   | IO-bound, bursty|  |
|  | IO-bound        |   | CPU-bound       |   | schedule-driven |  |
|  +-----------------+   +-----------------+   +-----------------+  |
|          |                     |                     |            |
|          +---------------------+---------------------+            |
|                                |                                  |
|                       +-----------------+                         |
|                       |  R4 STATE       |                         |
|                       |                 |                         |
|                       | local analytics |                         |
|                       | entity store    |                         |
|                       | token mapping   |                         |
|                       | evidence store  |                         |
|                       | credential vault|                         |
|                       | outbound queue  |                         |
|                       |                 |                         |
|                       | disk-bound      |                         |
|                       +-----------------+                         |
|                                |                                  |
|                       +-----------------+                         |
|                       |  R5 CONTROL     |                         |
|                       |                 |                         |
|                       | local UI + API  |                         |
|                       | resolve endpoint|                         |
|                       | SaaS sync       |                         |
|                       | response orch.  |                         |
|                       | health/telemetry|                         |
|                       |                 |                         |
|                       | latency-sensitive|                        |
|                       +-----------------+                         |
+-------------------------------------------------------------------+
```

**Why this split.** Each role has a different resource profile and a different failure consequence:

- R1 fails → data loss (unbuffered UDP syslog is gone forever). Needs redundancy and generous buffers.
- R2 fails → backlog grows; recoverable. Needs horizontal scale.
- R3 fails → posture data goes stale; recoverable and low urgency. Needs isolation so a runaway DSPM scan cannot starve R1/R2 — this is the single most common way these appliances fall over.
- R4 fails → everything fails. Needs the most operational care, backup, and integrity checking.
- R5 fails → analysts can't de-tokenize and response stops. Needs high availability, and it is the only role that must answer in milliseconds.

### 10.3 Sizing

Reference profiles. These are planning numbers to be validated by load testing, not measurements.

| Profile | Hosts | Events/day | vCPU | RAM | Disk | Layout |
|---|---|---|---|---|---|---|
| S | < 500 | < 50M | 8 | 32 GB | 500 GB | All roles, one VM |
| M | < 5,000 | < 500M | 16 | 64 GB | 2 TB | All roles, one VM |
| L | < 25,000 | < 3B | 32 | 128 GB | 8 TB | R3 split to own VM |
| XL | 25,000+ | 3B+ | cluster | — | — | R1/R2 scaled horizontally, R4 on dedicated storage |

Disk is dominated by the local analytics dataset and the evidence store — the two things that exist precisely so data does *not* leave. Retention is therefore a customer-tunable knob with a direct cost:

```
   evidence retention 30d   ->  ~1 TB at profile M
   evidence retention 90d   ->  ~3 TB at profile M
   evidence retention 365d  ->  ~12 TB at profile M
```

Default 90 days. Make the disk-cost consequence visible in the UI when the operator moves the slider.

### 10.4 The processing pipeline in detail

```
   [ SOURCE ]
       |
       v
   1. RECEIVE ................ protocol termination; per-source rate limiting;
       |                       immediate durable write to the ingest journal
       |                       (nothing is acknowledged before it is durable)
       v
   2. IDENTIFY ............... what is this? source fingerprinting:
       |                       vendor, product, format, version.
       |                       Cached per (source_ip, port, first 3 samples).
       v
   3. PARSE .................. grammar-driven extraction to a typed record.
       |                       Failure -> quarantine bucket + parser-health metric.
       |                       NEVER silently drop.
       v
   4. NORMALIZE .............. map vendor fields to the Overlook schema.
       |                       Timestamps to UTC; enums to canonical values;
       |                       IPs to normalized form; case folding.
       v
   5. ENRICH ................. attach known context: asset from IP, identity
       |                       from account, geo for external IPs, threat-intel
       |                       tags, business context from the CMDB connector.
       v
   6. RESOLVE ................ canonical key derivation + entity resolution
       |                       (Chapter 8). Emits stable local entity IDs.
       v
   7. CORRELATE (local) ...... rules over the local window: sequences,
       |                       thresholds, first-seen, rare-value detection.
       v
   8. FACT BUILD ............. produce/merge Security Facts. This is where
       |                       10,000 observations become one fact with
       |                       observation_count=10000.
       v
   9. PRIVACY GATE ........... tokenize, bucket, strip, schema-validate.
       |                       Fails closed: a fact that fails validation is
       |                       quarantined locally, never sent.
       v
  10. SIGN + QUEUE ........... Ed25519 sign; append to durable outbound queue.
       |
       v
  11. SYNC .................. batch, compress (zstd), mTLS to SaaS,
                             acknowledge, prune.

   Side branches:
       step 5 --> local analytics dataset (compressed, encrypted, retained)
       step 3 --> evidence store (raw, hashed, encrypted, TTL'd)
       step 6 --> token mapping table (canonical_key <-> token)
```

**Backpressure.** Each stage has a bounded queue. When a stage saturates, pressure propagates backwards to RECEIVE, which applies per-source rate limiting — dropping *the least valuable source first*, according to a configured priority. Never drop uniformly. Losing 10% of everything is worse than losing 100% of verbose debug syslog from a printer.

Priority classes, highest first: agent telemetry and AI Gateway facts → cloud audit logs → identity events → firewall/flow → application logs → everything else.

### 10.5 The credential vault

The Edge Node holds credentials for every connector — often 40+ sets of cloud keys, service accounts, and API tokens. **The Edge Node is therefore one of the most valuable targets in the customer's network.** Treat it accordingly:

- Credentials encrypted at rest with a key from the customer's KMS/HSM; never stored in configuration files or environment variables.
- The vault process is separate from the connector processes; connectors request a *scoped, time-bounded* credential handle rather than holding the secret.
- Every credential use is logged with connector, purpose, and timestamp.
- Preferred over static credentials, in order: workload identity (IRSA, managed identity, GCP workload identity) → short-lived OIDC federation → static keys with automatic rotation → static keys.
- Least-privilege connector roles, published as ready-to-apply IaC:

```
   OverlookAWSInventoryReader     (read-only, no data access, no decrypt)
   OverlookAzureSecurityReader
   OverlookGitHubSecurityReader
   OverlookFortiGateReader
   ...
   OverlookResponseRole           (SEPARATE, optional, off by default)
```

Splitting read from response is essential: it lets a customer deploy Overlook with zero write capability, which removes the single biggest procurement objection. Make read-only the default and response an explicit, separately-approved installation step.

### 10.6 High availability

```
   Active/Standby, shared nothing:

   EDGE-A (active)                    EDGE-B (standby)
     R1 receiving                       R1 receiving (both receive!)
     R2 processing                      R2 idle
     R4 state  ------ replicate ----->  R4 state (async, <60s lag)
     R5 serving VIP                     R5 ready

   Both nodes receive syslog/netflow (dual-target the sources) so a
   failover loses nothing in flight. Only one processes, elected via
   the state store's lease. Failover target: < 90 seconds.
```

For most customers a single Edge Node with good buffering is sufficient — the failure mode is delayed insight, not lost security. Offer HA for those who need it; do not make it mandatory complexity.

---

## 11. Connectors

### 11.1 The connector is the unit of product value

Blunt truth: **Overlook's practical value equals the number of connectors that work well.** The graph is empty without them. Plan for connectors to consume 40–50% of total engineering effort forever, and build the framework accordingly.

### 11.2 Framework contract

Every connector declares, in a manifest:

```yaml
connector:
  id: aws.iam
  version: 2.1.0
  vendor: Amazon Web Services
  category: [cloud, identity]

  auth:
    methods: [irsa, assume_role, access_key]
    least_privilege_policy: policies/aws-iam-reader.json

  collects:
    - entity: IDENTITY
      subtypes: [user, service_account]
      canonical_key_source: [tag.email, arn]
    - entity: ROLE
    - relationship: CAN_ASSUME
      from: [IDENTITY, ROLE]
      to: [ROLE]
      confidence: 0.99
    - relationship: MEMBER_OF

  schedule:
    mode: poll
    interval: 4h
    full_enumeration: true      # enables coverage windows / tombstoning

  cost:
    api_calls_per_run: "~3 per role + 2 per user"
    rate_limit_aware: true

  health:
    success_criteria: "enumerated >= 1 role and 0 auth errors"
```

The manifest is not documentation — it is **executable**. It drives:
- what the UI shows as coverage ("you have IAM coverage on 38 of 42 accounts")
- coverage windows for tombstoning (§9.2)
- the least-privilege policy the customer applies
- the health model
- the conformance test suite

### 11.3 Ingestion modes

| Mode | Sources | Notes |
|---|---|---|
| **API poll** | Cloud, IdP, SaaS, EDR, scanners | Dominant mode. Needs rate-limit awareness, pagination, incremental cursors, and backoff. |
| **Syslog push** | Firewalls, network gear, Unix hosts | UDP is lossy — recommend TCP/TLS. Buffer generously. |
| **Webhook** | GitHub, cloud event grids, SaaS | Needs a signed, authenticated receiver and replay protection. |
| **Flow** | NetFlow v9, IPFIX, sFlow, VPC flow logs | Enormous volume. Aggregate aggressively at receive (see 11.4). |
| **Agent** | Overlook Agent | mTLS, agent-initiated, bidirectional for response. |
| **File/share scan** | DSPM sources | Scheduled, throttled, read-only. |
| **Message bus** | Kafka, Event Hubs, Pub/Sub | For customers who already centralize. |

### 11.4 Flow data: aggregate at the edge of the Edge

Flow data will dwarf everything else — a mid-size network produces billions of flow records a day. Overlook does not need flows; it needs **reachability evidence**. So aggregate at receive time, before anything else touches it:

```
   Raw:  4.1 billion flow records/day
          |
          v  aggregate by (src_subnet, dst_subnet, dst_port, protocol)
             over 15-minute windows
          |
   Agg:  ~180,000 FLOW_AGGREGATE records/day
          |
          v  convert to graph edges
          |
   Facts: ~4,000 CONNECTS_TO edges/day (most are repeats -> last_seen bump)
```

A 1,000,000:1 reduction, with no loss of the signal that matters: *this subnet talks to that port on that subnet.* Resist every request to retain per-flow detail; that is an NDR feature and it is not this product.

### 11.5 Parser health as a first-class concern

Parsers rot. Vendors change log formats in a patch release and a connector silently starts producing garbage. Defences:

- **Never silently drop.** Unparsed records go to a quarantine bucket with a sample retained.
- **Parse-rate alerting.** If `parsed / received` for a source drops below its baseline, that is a P1 for the customer's Overlook operator, surfaced in the local UI.
- **Field-presence monitoring.** If `src_user` was populated in 94% of records last week and 3% today, the vendor changed the format.
- **Sample capture** for every parse failure class, so a support engineer can write a fix without asking the customer for data — which, given the privacy premise, they may be unable to send anyway.

That last point is a real operational constraint of the privacy architecture: **you cannot ask the customer to email you their logs.** Debugging tooling must run at the Edge, produce redacted diagnostics, and be usable by the customer's own operator. Budget for it.

---

## 12. The Overlook Agent

### 12.1 Scope discipline

The brief says the agent "is not intended to replace a full EDR" and then lists process trees, DNS, connections, quarantine, and process termination — which is an EDR feature set minus detection.

The disciplined position:

> **The agent collects only what APIs cannot provide, and executes only what nothing else can execute.**

Under that rule, most of the list falls away, because the customer already runs CrowdStrike or Defender and both expose process, network, and logon telemetry via API — with better coverage, already-approved kernel drivers, and no new agent to justify.

**Collect via existing EDR API (do not build):** process trees, command lines, network connections, DNS, logon events, host inventory, patch level.

**Collect via Overlook Agent (nothing else provides it):**
- MCP client and server configuration files (`~/.claude/`, `~/.cursor/`, `claude_desktop_config.json`, and equivalents) — nobody else parses these
- Locally-running model processes (ollama, llama.cpp, LM Studio, vLLM) and the models they have loaded
- IDE AI extension inventory and which endpoints they talk to
- Browser-to-AI-domain relationships with the owning user and process
- AI SDK and library presence in local environments
- Local credential-file presence for AI services (`.env` containing `OPENAI_API_KEY`, `~/.aws/credentials` used by an agent process)
- Local agent frameworks and their configured tools

That is a **thin, read-mostly, userland agent**. No kernel driver. No hooks. It can ship in months rather than years, and it will not be blamed for a blue screen.

### 12.2 Architecture

```
+-------------------------------------------------------+
|                  OVERLOOK AGENT                        |
|                                                        |
|   +------------------+     +---------------------+     |
|   |  COLLECTORS      |     |  RESPONSE EXECUTOR  |     |
|   |                  |     |                     |     |
|   | process snapshot |     | verify signature    |     |
|   | ai config scan   |     | check TTL + nonce   |     |
|   | model detection  |     | pre-flight check    |     |
|   | net connections  |     | execute             |     |
|   | listening ports  |     | verify result       |     |
|   | local accounts   |     | report              |     |
|   |                  |     | auto-rollback timer |     |
|   | interval-driven  |     |                     |     |
|   | low priority     |     | DISABLED BY DEFAULT |     |
|   +------------------+     +---------------------+     |
|            |                        ^                  |
|            v                        |                  |
|   +--------------------------------------------+       |
|   |  LOCAL BUFFER (bounded, encrypted)         |       |
|   |  24h or 200MB, whichever first             |       |
|   +--------------------------------------------+       |
|            |                                           |
|            v                                           |
|   +--------------------------------------------+       |
|   |  TRANSPORT: agent-initiated mTLS to Edge   |       |
|   |  outbound only, no listening port          |       |
|   +--------------------------------------------+       |
+-------------------------------------------------------+
```

**Agent-initiated, outbound-only.** The agent has no listening port. It polls the Edge Node for pending commands on its heartbeat. This removes an entire class of vulnerability (a remotely exploitable agent listener has ended companies) and makes firewall approval trivial.

### 12.3 Resource discipline

An agent that costs a developer 3% CPU gets uninstalled and takes the deal with it. Hard limits, self-enforced:

```
   CPU:      < 1% average, < 5% peak, self-throttling with a token bucket
   Memory:   < 150 MB RSS
   Disk IO:  < 5 MB/s during scans
   Network:  < 50 MB/day typical
   Scans:    full AI-config scan every 4h; process snapshot every 60s;
             all scans yield to system load above a threshold
```

Publish these as a contract and enforce them in the agent itself — a watchdog that throttles or suspends collection when the host is under load. Ship a resource-usage report in the local UI so the customer can verify.

### 12.4 Platform notes

| Platform | Considerations |
|---|---|
| Windows | Service, LocalSystem. AV/EDR exclusion documentation. Code-signing cert with EV. |
| macOS | Notarization, TCC prompts for file access (`~/Library`, Documents). Signed installer + MDM deployment profile so users are not prompted individually. |
| Linux | systemd unit, minimal deps, musl/static build to avoid glibc version hell across distros. |
| Containers | Do **not** deploy per-container. Use a DaemonSet reading the node, plus the Kubernetes API for workload identity mapping. |

macOS TCC is the underestimated one: scanning `~/.claude/` and `~/Library/Application Support/` triggers consent prompts unless deployed via MDM with a PPPC profile. Ship that profile with the product.

---

## 13. The Overlook AI Gateway

### 13.1 The hard truth about prompt visibility

To read a prompt typed into `chatgpt.com`, something must break TLS or sit inside the application. There are exactly four options, and each has real costs:

| Approach | How | Cost | Verdict |
|---|---|---|---|
| **TLS-intercepting proxy** | Enterprise CA on every device, MITM | Cert pinning breaks apps; privacy/legal review; user backlash; huge deployment friction | Do not lead with this |
| **Integrate an existing SWG/CASB** | Read logs/API from Zscaler, Netskope, Palo Alto, Defender for Cloud Apps | Customer already accepted the MITM cost. Free coverage. | **Do this first** |
| **Browser extension** | Managed extension reads the page | Precise, per-user, no TLS break; needs MDM deployment; browser-specific | **Do this second** |
| **API-layer gateway** | Overlook sits in front of model APIs as an LLM proxy | Natural fit; teams *want* a gateway for cost, routing, caching | **Do this third — and it is the strongest** |

### 13.2 The strategic insight

The brief treats the AI Gateway as one thing. It is really two products with different economics:

**(a) Employee AI usage visibility** — "who pastes what into ChatGPT". Low technical moat, high deployment friction, crowded market. **Get this via SWG/CASB integration and a browser extension. Do not build a proxy for it.**

**(b) Application and agent AI traffic** — an LLM gateway in front of Bedrock, Azure OpenAI, Vertex, and internal models, through which the customer's own applications and agents call models. Here, being inline is *welcome* — engineering teams deploy an LLM gateway anyway for cost tracking, rate limiting, failover, and caching. You arrive as infrastructure they wanted, and you get perfect visibility into the traffic that actually matters: **the agents with production credentials**.

(b) is where the differentiated product is. An employee pasting a customer list into ChatGPT is a DLP problem with a dozen vendors. An agent with an admin service account receiving an indirect prompt injection is an Overlook problem, and nobody else can see it, because nobody else has the graph to explain the consequence.

### 13.3 Inspection pipeline

```
   Request from application / agent / user
        |
        v
   [ AUTHENTICATE ]  which app, which identity, which agent
        |
        v
   [ PARSE ]  extract: model, messages, system prompt, tools declared,
        |     tool results included, attachments, retrieval context
        v
   [ INSPECT ] -- runs locally, in parallel, budget < 40ms p95
        |
        +-- Sensitive data:  PII / PCI / PHI  (regex + validators + NER)
        +-- Secrets:         API keys, tokens, private keys (entropy + patterns
        |                    + provider-specific formats + checksum validation)
        +-- Source code:     language detection + internal-repo fingerprint match
        +-- Prompt injection: direct (instruction-override patterns, role-play
        |                    framing, encoding tricks) and indirect (injection
        |                    markers inside retrieved documents or tool output)
        +-- Data volume:     unusual payload size, bulk paste detection
        +-- Tool declaration: which tools this call exposes to the model
        |
        v
   [ POLICY ]  allow | redact | warn | block  (per app / identity / data class)
        |
        v
   [ FORWARD ]  to the model
        |
        v
   [ INSPECT RESPONSE ]
        |
        +-- system prompt leakage
        +-- sensitive data in output
        +-- tool calls requested by the model  <-- highest value signal
        +-- links / exfiltration instructions
        |
        v
   [ EMIT FACTS ]  PROMPT_EVENT + relationships. Content never leaves.
        |
        v
   Response to caller
```

### 13.4 The latency budget is the product constraint

An inline component on a user-facing path has a hard budget. Exceed it and it gets removed.

```
   Total added latency budget:  < 50 ms p95, < 150 ms p99

   Realistic allocation:
     TLS termination + parse           5 ms
     Regex/pattern scanning            8 ms   (compiled, parallel)
     Secret detection + validation     6 ms
     ML classifier (small, local)     15 ms   (ONNX, CPU, quantized)
     Policy evaluation                 2 ms
     Fact emission (async, off-path)   0 ms
     ---------------------------------------
                                      36 ms
```

Design rules that follow:
- Fact emission is **always asynchronous**. Never block the user to write telemetry.
- Any inspection that cannot meet budget runs **out-of-band** on a sample, not inline.
- Large ML models do not go inline. If deep semantic analysis is needed, do it asynchronously on flagged traffic only.
- **Streaming responses must stay streaming.** Buffering an entire LLM response to inspect it destroys the UX and will get the gateway removed. Inspect incrementally on the stream, and accept that a block decision may come mid-stream.

### 13.5 Deployment modes

```
Mode A — Reverse proxy (application traffic)
   App --> https://ai-gw.corp.local/v1/chat/completions --> Overlook GW --> OpenAI
   Config change: one base URL. Lowest friction. Best coverage of agent traffic.

Mode B — Forward proxy / SWG integration
   User --> Corporate SWG --> [Overlook inspects via ICAP or log/API feed] --> Internet
   No new inline component. Coverage depends on the SWG.

Mode C — Browser extension
   Managed extension observes AI sites in-page, reports metadata to Edge.
   Per-user precision, no TLS break, MDM deployment required.

Mode D — SDK / middleware
   Drop-in wrapper for the customer's own AI applications.
   Deepest context (agent identity, tool calls, RAG retrieval) but requires
   the customer to change code. Best for internal agent platforms.
```

Ship A and D first. They cover the traffic that produces the differentiated findings.

---

## 14. Overlook SaaS

### 14.1 Service decomposition

```
                        +---------------------+
                        |   API GATEWAY       |
                        |   authn / authz     |
                        |   tenant routing    |
                        +---------------------+
                            |            |
        +-------------------+            +------------------+
        |                                                   |
+---------------+  +---------------+  +---------------+  +---------------+
| INGEST        |  | GRAPH         |  | ANALYTICS     |  | RESPONSE      |
|               |  |               |  |               |  |               |
| verify sig    |  | node/edge CRUD|  | attack path   |  | request       |
| verify schema |  | traversal API |  | risk scoring  |  | approval flow |
| dedupe        |  | temporal query|  | blast radius  |  | dispatch      |
| write to graph|  | subgraph      |  | change detect |  | audit         |
| emit changes  |  |               |  | choke points  |  |               |
+---------------+  +---------------+  +---------------+  +---------------+
        |                  |                  |                  |
        +------------------+------------------+------------------+
                                   |
                    +--------------------------------+
                    |   STORAGE                      |
                    |   graph store (per tenant)     |
                    |   change log                   |
                    |   findings / paths cache       |
                    |   config, RBAC, audit          |
                    +--------------------------------+
```

### 14.2 Multi-tenancy

Given the customer profile — regulated enterprises that chose Overlook *for* its privacy architecture — tenancy isolation must be stronger than the industry norm.

- **Data isolation:** separate schema per tenant, minimum. Separate database for large or regulated tenants. Never a shared table with a `tenant_id` column; one missing `WHERE` clause is a cross-customer breach, and this customer base will not forgive it.
- **Compute isolation:** attack-path computation is expensive and bursty. Run it in per-tenant workers with hard resource caps so one customer's 120M-edge graph cannot starve another's.
- **Key isolation:** every tenant's data encrypted with a tenant-specific key.
- **Region residency:** the SaaS control plane must be deployable per region (EU, India, US, Gulf) with no cross-region data flow. Given the target market, this is a launch requirement, not a later one.

### 14.3 What SaaS provides that the Edge cannot

A fair question: if everything is analysed at the Edge, why have SaaS at all? Four reasons, and they should be stated to customers plainly:

1. **Cross-Edge correlation.** The hybrid attack path crossing on-prem AD → GitHub Cloud → AWS → back to an on-prem Oracle DB spans three Edge Nodes. Only SaaS sees all of it.
2. **Cross-tenant intelligence.** "This MCP server version has a known tool-poisoning issue"; "this attack path shape is exploited in the wild"; "your peers remediate this class in 6 days, you take 40." Aggregate only, never per-customer.
3. **Content delivery.** Parsers, detection rules, AI application fingerprints, MCP server reputation, model inventory. This is a continuously-updated product, and it needs a distribution channel.
4. **Operational scale.** Attack-path computation over 120M edges is not something you want running on an appliance in a customer's data centre competing with ingestion.

---

## 15. Edge ↔ SaaS: trust, transport, sync

### 15.1 Enrollment

The bootstrap problem: how does a fresh Edge Node prove it belongs to a tenant, and how does it get a certificate?

```
 1. Customer admin, in the Overlook SaaS console, clicks "Add Edge Node".
    -> SaaS generates a one-time enrollment token (24h TTL, single-use,
       bound to tenant + a chosen node name).

 2. Admin runs the installer, pastes the token.

 3. Edge Node generates an Ed25519 keypair. The private key is
    generated in and never leaves the node (TPM/HSM-backed where available).

 4. Edge Node presents the enrollment token + a CSR to SaaS over TLS.

 5. SaaS validates the token, issues a client certificate bound to
    (tenant_id, edge_node_id), 90-day lifetime.

 6. Edge Node derives or receives the tenant tokenization key:
      - FIRST node in a tenant: generates it locally, wraps with customer KMS.
      - SUBSEQUENT nodes: obtain it from the customer's KMS using their own
        cloud identity, OR via an operator-mediated transfer.
        NEVER through Overlook SaaS.

 7. Ongoing: certificates auto-renew at 2/3 lifetime over the established
    mTLS channel. Revocation list checked on every connection.
```

Step 6 is the one to get right. It is the only step where a shortcut would let Overlook see the key, and taking that shortcut would quietly invalidate the entire privacy claim.

### 15.2 Transport

```
   Edge Node ------------ TLS 1.3, mutual auth ------------> Overlook SaaS
                          port 443, outbound only
                          Edge always initiates

   Batch format:
     header:  tenant_id, edge_node_id, batch_id, schema_version,
              fact_count, compression=zstd, sig_alg=ed25519
     body:    zstd(newline-delimited signed facts)
     footer:  batch signature, sha384 of body

   Batching policy:
     flush on 1000 facts, or 5 MB, or 60 seconds — whichever first
     (bounded latency AND bounded overhead)

   Acknowledgement:
     SaaS returns per-fact accept/reject with reasons.
     Rejected facts are quarantined at the Edge with the reason,
     surfaced in local UI. They are NOT retried blindly forever.
```

**Outbound-only, Edge-initiated.** SaaS never connects to the Edge Node. No inbound firewall rule is ever required. This matters enormously in procurement — "you do not need to open a port for us" removes a security-architecture review that can take months.

The consequence: SaaS cannot push. Commands and configuration wait in a queue that the Edge polls (§15.4).

### 15.3 Offline operation

```
   SaaS unreachable
        |
        v
   Outbound queue grows on local disk (durable, encrypted)
        |
   Default capacity: 7 days at expected fact rate, or 20% of disk
        |
        +-- at 60% -> warn in local UI
        +-- at 80% -> warn + email the Overlook operator
        +-- at 95% -> begin shedding lowest-priority fact classes
        |             (EVENT_SUMMARY first, RELATIONSHIP last)
        +-- at 100% -> shed all but FINDING and RELATIONSHIP facts
        v
   On reconnect: drain oldest-first, rate-limited so the backlog
   does not overwhelm ingest. Facts are idempotent (§5.5) so replay
   is safe. Duplicate detection is on semantic identity, not arrival.
```

During an outage the local UI keeps working: inventory, local findings, connector health, and evidence lookup all function. Only cross-Edge correlation, global paths, and response are unavailable. Say this clearly in the UI rather than showing a generic error.

### 15.4 The command channel

Since SaaS cannot connect inbound, control flows through a polled queue:

```
   Edge polls:  GET /v1/commands?after=<cursor>    every 30 seconds
                (long-poll, 25s hold, for near-real-time response)

   Command types:
     CONFIG_UPDATE    connector settings, policy, retention
     CONTENT_UPDATE   parsers, rules, AI fingerprints
     RESPONSE_REQUEST a response action for local validation (Ch. 30)
     DIAGNOSTIC       collect redacted diagnostics
     UPGRADE          initiate a staged upgrade

   Every command is signed by SaaS with a key the Edge pins at enrollment,
   carries a nonce and an expiry, and is validated locally before execution.
   Local policy can REFUSE any command class outright — an Overlook operator
   can configure "this Edge never accepts RESPONSE_REQUEST", and SaaS cannot
   override that.
```

That last sentence is a genuine trust feature. The customer's local policy beats the vendor's cloud, always. Make it visible in the UI as a set of toggles the customer controls.

---

## 16. The local management UI

Often treated as an afterthought. In this architecture it is a primary surface, because it is the only place several critical things can happen:

| Function | Why it must be local |
|---|---|
| Outbound inspection | Show the operator exactly what facts left, in the last hour/day. The privacy claim made concrete. |
| Token resolution admin | The mapping table lives here and nowhere else. |
| Evidence lookup | Raw evidence never leaves the Edge. |
| Connector health & credentials | Credentials never leave the Edge. |
| Resolution Review queue | Ambiguous merges need a human who knows the org (§8.2). |
| Privacy policy configuration | The customer decides what may be sent. Vendor cannot override. |
| Response local policy | Which response classes this Edge will ever accept. |
| Diagnostics | Support cannot receive customer data; the operator must self-diagnose. |

Design principle: **anything requiring plaintext customer data happens here.** Anything requiring cross-domain correlation happens in SaaS. That split is clean and easy to explain to an auditor.

---

## 17. Content and update pipeline

Parsers, rules, fingerprints, and model/MCP reputation are **continuously updated product**, and they need a delivery mechanism that works into air-gapped and change-controlled environments.

```
   Overlook content build
        |
        +-- parsers/          vendor log grammars
        +-- rules/            posture + correlation rules
        +-- ai-fingerprints/  AI app/domain/model identification
        +-- mcp-reputation/   known MCP servers and their risk
        +-- iam-semantics/    cloud permission -> capability mappings
        |
        v
   Signed content bundle (semver, Ed25519, reproducible build)
        |
        v
   CDN --> Edge Node pulls on schedule
        |
        +-- staged rollout: canary (1%) -> 10% -> 100% over 72h
        +-- customer can pin a version
        +-- customer can require manual approval (change-controlled envs)
        +-- automatic rollback if parse rate drops post-update
        |
        v
   Air-gapped: signed bundle downloadable as a file, applied via local UI
```

**`iam-semantics` deserves emphasis.** Knowing that `iam:PassRole` combined with `ec2:RunInstances` equals privilege escalation is *content*, not code. AWS ships new services monthly. That mapping is a living dataset, it is a genuine moat if maintained well, and it is the difference between an attack-path engine that finds real escalations and one that finds trivia. Staff it.

---

## 18. Deployment topologies

### 18.1 On-premises

```
                     CUSTOMER DATA CENTRE

  Active Directory ----------+
  VMware / Hyper-V ----------+
  GitLab / Jenkins ----------+
  Oracle / MSSQL ------------+
  File shares / NAS ---------+
  FortiGate / Palo Alto -----+---> OVERLOOK EDGE NODE
  NetFlow / IPFIX -----------+     (VM, 16 vCPU, 64 GB, 2 TB)
  CrowdStrike / Defender ----+           |
  Servers + Overlook Agent --+           | Security Facts
  Internal LLM / Ollama -----+           | mTLS 443 outbound
  MCP servers ---------------+           |
                                         v
                                  OVERLOOK SaaS

  Segmented zones (OT, PCI, DMZ):
      lightweight satellite collector -> forwards to Edge Node
      (collection only; no analytics, no credential vault, no state)
```

### 18.2 Cloud

```
                        CUSTOMER CLOUD ACCOUNT

   AWS / Azure / GCP APIs --------+
   IAM / Entra / Cloud IAM -------+
   CloudTrail / Activity Log -----+
   EKS / AKS / GKE ---------------+---> OVERLOOK EDGE NODE
   GitHub / CI-CD ----------------+     private subnet
   S3 / Blob / GCS ---------------+     NO public IP
   RDS / Cloud SQL ---------------+     egress via NAT
   VPC / VNet / Flow logs --------+           |
   VMs + Overlook Agent ----------+           | Security Facts
   Bedrock / Azure OpenAI --------+           | mTLS 443
   AI agents / MCP / RAG ---------+           v
                                       OVERLOOK SaaS

   Identity: IRSA / Managed Identity / Workload Identity
             -> no static credentials anywhere
```

The Edge Node always initiates. Overlook SaaS never holds customer cloud credentials and never calls customer cloud APIs. This is worth stating explicitly in the security whitepaper, because it eliminates the "vendor gets breached, all customers' clouds are exposed" scenario that has burned this industry repeatedly.

### 18.3 Deployment levels

| Level | Components | Adds | Typical time to value |
|---|---|---|---|
| **1** | Edge Node | Cloud/identity/network/app posture, asset + AI inventory, TrustGraph, attack paths, basic Shadow AI | 1 day to first graph, 1 week to full |
| **2** | + Agent | Host runtime context, local AI + MCP discovery, IDE assistant visibility, host response | +2 weeks (rollout) |
| **3** | + AI Gateway | Prompt security, tool-call analysis, RAG inspection, agent behaviour, AI DLP context | +4 weeks (integration) |

Level 1 must be independently valuable and independently sellable. If a customer needs the agent before they see value, the sales cycle doubles. Everything in Level 1 is API-only — that is the design constraint that makes a one-day POC possible.

---

# PART IV — THE INTELLIGENCE

---

## 19. Correlation

### 19.1 Two kinds, in two places

**Local correlation (Edge Node)** — operates on raw events within a time window, with full fidelity, before data is reduced.
- Sequence detection: failed auth × 20 → success → privilege change
- First-seen: this service account has never called this API before
- Rare-value: this user has never logged in from this ASN
- Threshold: this datastore returned 400× its normal row count

Output: a `FINDING` or `EVENT_SUMMARY` fact. The raw events stay local.

**Global correlation (SaaS)** — operates on the graph, across sources and Edge Nodes, without any raw events.
- Structural: an identity with `CAN_ASSUME` to an admin role and no MFA and dormant 90 days
- Cross-domain: a repository containing a secret that authenticates to a role that reaches a crown jewel
- Temporal: a new admin edge appeared 3 hours after a suspicious authentication finding
- Toxic combinations: publicly exposed + unpatched critical CVE + holds a credential + reaches a crown jewel

The division is forced by the privacy model, and it turns out to be the right division anyway: event-sequence correlation needs fidelity and locality; structural correlation needs breadth. Neither can do the other's job.

### 19.2 Toxic combinations

The single highest-value global correlation pattern. Individually-ignorable conditions that are critical together:

```
   Individually:
     "S3 bucket is public"                  -> 40,000 of these exist
     "bucket contains PII"                  -> 2,000 of these
     "bucket is in a production account"    -> 12,000 of these

   Together:
     public AND contains PII AND production AND
     no bucket policy restriction AND accessed from outside in last 30d
       -> 3 of these.

   Those 3 are the entire finding.
```

The maths matters: a platform showing 54,000 findings is worthless; the same platform showing 3 is indispensable. **Overlook's job is that reduction.** Every design decision should be evaluated against whether it improves or degrades it.

---

## 20. The Attack Path Engine

### 20.1 Definition

An attack path is a directed sequence of traversable edges from a **start condition** to a **crown jewel**, where every edge represents a capability an attacker would inherit.

```
  START CONDITIONS (where an attacker realistically begins)
    - internet-exposed asset with an exploitable vulnerability
    - a phishable human identity
    - a leaked/exposed credential
    - a third-party/supply-chain integration
    - an AI agent reachable by an untrusted input   <- Overlook-specific
    - an already-compromised asset (for blast radius)

  CROWN JEWELS (what matters)
    - customer-designated critical assets
    - datastores classified PII/PCI/PHI above a volume threshold
    - production identity providers and CI/CD systems
    - anything holding credentials that reach the above
```

### 20.2 The permission-closure problem

The single hardest technical problem in the system, and the one most often done badly.

A raw IAM policy says: `principal P may perform action A on resource R, if condition C`. Converting that into a graph edge `P CAN_READ R` requires evaluating a permission model that, in AWS alone, involves identity policies, resource policies, permission boundaries, SCPs, session policies, and condition keys — with a specific evaluation order and explicit-deny precedence.

**Where to compute it:** at the Edge Node, not in SaaS. Three reasons.

1. Policy documents are customer data. Shipping them to SaaS breaks the privacy model.
2. The closure is far smaller than the input. Thousands of policy documents collapse to a few hundred thousand resolved edges.
3. It needs resource-level detail (tags, names, paths) that would otherwise have to be tokenized and then reasoned over, which is impossible.

**What to ship upward:** the *resolved* edge, plus enough qualification to reason about it centrally:

```jsonc
{
  "subject": "IDN-9f3a...", "predicate": "CAN_WRITE", "object": "DST-1c4b...",
  "attributes": {
    "actions": ["s3:PutObject", "s3:DeleteObject"],
    "granted_via": ["identity_policy:ROL-2b8e...", "resource_policy"],
    "conditions": [
      {"key": "aws:SourceIp",  "op": "IpAddress", "satisfied_by_default": false},
      {"key": "aws:MultiFactorAuthPresent", "op": "Bool", "satisfied_by_default": false}
    ],
    "unconditional": false,
    "boundary_limited": true
  },
  "confidence": 0.93
}
```

Conditions must survive the trip. An edge that only exists from a specific IP range with MFA is a materially weaker edge than an unconditional one, and a path engine that ignores conditions will produce paths that do not exist — the fastest way to lose an analyst's trust permanently.

### 20.3 The algorithm

Naïve all-pairs traversal on 120M edges is intractable and unnecessary. The practical approach is **backwards from crown jewels**, which is both faster and better aligned with what users want:

```
  STEP 1 — Precompute closure for expensive predicates
    MEMBER_OF and CAN_ASSUME are transitive and dense.
    Materialize their transitive closure incrementally on change,
    not on query. This is 80% of the traversal cost.

  STEP 2 — Reverse BFS from crown jewels
    for each crown_jewel:
        frontier = {crown_jewel}
        for depth in 1..MAX_DEPTH (default 8):
            frontier = incoming_traversable_edges(frontier)
            prune: cumulative_weight > threshold  -> drop branch
            prune: confidence < 0.5               -> drop branch
            record predecessors
        emit paths where the terminal node is a START CONDITION

  STEP 3 — Score and rank        (Chapter 21)

  STEP 4 — Collapse
    Group paths sharing a choke point. Present the choke point,
    not the 400 paths through it.

  STEP 5 — Diff against previous run
    NEW / PERSISTING / RESOLVED. Only NEW generates a notification.
```

**Incrementality is mandatory.** A full recomputation over a large graph takes minutes to hours; customers change their environment continuously. On each graph change, recompute only the affected subgraph: a changed edge invalidates paths through it, which the reverse index locates directly.

```
   edge changed -> lookup affected_paths(edge) via reverse index
                -> recompute only those
                -> typical: 40 paths, ~200ms
                -> versus full recompute: 500k paths, ~25 minutes
```

### 20.4 Path explosion, and the honest answer

With permissive IAM, path counts explode combinatorially. A single over-permissive role can generate hundreds of thousands of paths. Controls, in order of importance:

1. **Depth limit.** 8 hops. Beyond that, paths stop being actionable narratives.
2. **Weight budget.** Cumulative traversal weight cap; a path of eight low-probability edges is not a real risk.
3. **Choke-point collapsing.** Present the shared edge once, not every path through it. Usually a 100:1 reduction.
4. **Equivalence classes.** 400 EC2 instances in one ASG with an identical role are one path with a count of 400, not 400 paths.
5. **Crown-jewel scoping.** Only compute to designated crown jewels. Everything else is blast radius, computed on demand.

The honest statement to make to customers: **path counts are not a metric.** "You have 47,000 attack paths" is noise theatre. "You have 6 choke points; removing 3 permissions eliminates 31,000 paths" is the product. Design the UI so the choke point is the primary object and paths are the drill-down.

### 20.5 TRACE 2 — a hybrid attack path, end to end

The path in the original brief, traced through every component. This is the reference example for how sources become a single narrative.

```
  THE PATH
    On-Prem AD --> Developer --> GitHub Cloud --> CI/CD --> AWS Role
      --> EC2 --> VPN --> On-Prem Oracle --> Customer PII

  WHO SAW WHAT

  Edge Node "EDGE-DC1" (on-prem):
    AD connector    -> IDENTITY   Priya  (canonical email:priya.s@corp.com)
                    -> MEMBER_OF  GRP "Developers"
    Oracle connector-> DATASTORE  prod-oracle-01, classified PII, 4.2M records
    FortiGate conn. -> ROUTES_TO  vpn-subnet 10.8.0.0/24 -> db-subnet 10.4.2.0/24
                                  port 1521 permitted
    NetFlow         -> CONNECTS_TO observed 10.8.0.x -> 10.4.2.17:1521

  Edge Node "EDGE-AWS" (cloud):
    GitHub connector-> REPOSITORY payments-api
                    -> IDENTITY  priya.s@corp.com CAN_DEPLOY payments-api
                    -> PIPELINE  gha-deploy-prod
                    -> PIPELINE CAN_ASSUME role/GHADeployRole  (OIDC trust)
    AWS IAM conn.   -> ROLE GHADeployRole CAN_ASSUME role/EC2AppRole
                                  (permission closure resolved locally)
    AWS EC2 conn.   -> ASSET i-0abc123 RUNS_ON role/EC2AppRole
                    -> ASSET i-0abc123 in subnet with route to VPN

  RESOLUTION
    Both Edges resolve "priya.s@corp.com" to canonical key
      email:priya.s@corp.com
    Both compute token IDN-9f3a7c21e845b0d6 (same tenant key)
    -> the two subgraphs JOIN in SaaS on that token.
    This join is the entire value of deterministic tokenization.
    With random tokens, this path would not exist.

  IN SaaS
    Reverse BFS from crown jewel DST-oracle (criticality 95):
      DST-oracle <- CONNECTS_TO  AST-ec2       (w 0.30, conf 0.88)
      AST-ec2    <- RUNS_ON      ROL-ec2app    (w 0.05, conf 0.99)
      ROL-ec2app <- CAN_ASSUME   ROL-ghadeploy (w 0.10, conf 0.97)
      ROL-ghadeploy <- CAN_ASSUME PIP-gha      (w 0.15, conf 0.95)
      PIP-gha    <- CAN_DEPLOY   REP-payments  (w 0.10, conf 0.99)
      REP-payments <- CAN_WRITE  IDN-9f3a      (w 0.10, conf 0.99)
      IDN-9f3a is a START CONDITION (phishable human, MFA present)

    Path score:
      reachability = product(1 - w) chain            -> 0.53
      crown jewel criticality                        -> 95
      data sensitivity (PII, 1M-10M bucket)          -> 90
      exposure of start (human, phishable, MFA on)   -> 55
      path age (exists 41 days)                      -> +8
      confidence (min across edges)                  -> 0.88
      ------------------------------------------------------
      RISK SCORE 87 / 100  CRITICAL

  IN THE UI (with de-tokenization from both Edge Nodes)
    Priya S -> payments-api -> GitHub Actions -> GHADeployRole
      -> EC2AppRole -> i-0abc123 -> VPN -> prod-oracle-01 -> 4.2M PII records

  CHOKE POINT ANALYSIS
    ROL-ghadeploy CAN_ASSUME ROL-ec2app appears in 1,240 paths.
    Recommended fix: scope GHADeployRole's sts:AssumeRole to a
    deployment-only role without EC2 instance-profile access.
    Impact: eliminates 1,240 paths, including 9 CRITICAL.
    Blast-radius preview of the change: 3 pipelines affected,
    2 require configuration updates.
```

Note what made this possible: **five connectors across two Edge Nodes, one canonical key, one deterministic token.** Remove any of those and the path is invisible. This is why Part II matters more than Part III.

---

## 21. Risk and prioritisation

### 21.1 The core problem

Prioritisation is where exposure products live or die. CVSS is not prioritisation — it describes a vulnerability in the abstract, with no knowledge of whether the asset is reachable or matters. Overlook's advantage is that it *knows the context*, so it must use it.

### 21.2 Scoring model

```
  PATH_RISK = f( REACHABILITY, IMPACT, EXPOSURE, CONFIDENCE, AGE )

  REACHABILITY  how likely can an attacker actually traverse this?
                product of per-edge traversal probabilities
                reduced by PROTECTS edges (EDR present, MFA required,
                network control, PAM in path)

  IMPACT        what is at the end?
                crown jewel criticality × data sensitivity ×
                blast radius from the terminal node

  EXPOSURE      how attackable is the start?
                internet-facing > phishable human > insider >
                requires prior access

  CONFIDENCE    min(edge confidence) across the path.
                A path is only as trustworthy as its weakest inference.
                Displayed, never hidden.

  AGE           older paths score higher — they have survived review,
                which means the organisation is blind to them
```

### 21.3 Edge weights

Each predicate has a base traversal probability, adjusted by attributes:

```
  predicate         base   adjustments
  ---------------   ----   ---------------------------------------------
  CAN_ASSUME        0.95   -0.30 if MFA condition; -0.20 if IP condition
                           -0.40 if requires external ID
  CAN_EXECUTE       0.90   -0.35 if EDR PROTECTS target
  MEMBER_OF         0.99   (essentially free for an attacker)
  CAN_WRITE         0.85   -0.25 if approval workflow PROTECTS
  CAN_DEPLOY        0.80   -0.40 if branch protection + required reviews
  CONNECTS_TO       0.60   observed traffic; weaker than configured reach
  ROUTES_TO         0.70   -0.30 if IPS/segmentation PROTECTS
  RUNS_ON           0.95   host compromise implies workload compromise
  INVOKES           0.85   agent-to-tool
  RETRIEVES_FROM    0.75   RAG retrieval
  PROMPTED_BY       0.50   requires social engineering or injection
```

These numbers are **hypotheses to be calibrated**, not truths. Build the calibration loop from day one: when a customer marks a path "not exploitable, here's why", that feedback adjusts weights for that tenant and, in aggregate and anonymised, informs the global defaults. A prioritisation engine that never learns from analyst disposition will be tuned out within a quarter.

### 21.4 What to show

```
  DO show:
    "3 choke points. Fixing them eliminates 31,000 paths."
    "This path has existed 41 days and reaches 4.2M PII records."
    "Confidence 0.88 — the network hop is inferred from flow data,
      not from a firewall rule."

  DO NOT show:
    "47,213 attack paths"          -> unactionable
    "Risk score 8.4"                -> meaningless without context
    "1,204 critical findings"       -> the customer already has 60,000
```

Every number in the UI should imply an action. If it does not, remove it.

---

## 22. Change intelligence

### 22.1 Why change beats state

A static posture report is read once and filed. **Change** is what a security team can act on:

- *A new admin grant appeared 3 hours ago* — highest-signal event in identity security
- *This path became reachable when a firewall rule changed at 14:22* — direct causation
- *This agent gained a new tool yesterday* — AI-specific and completely invisible to everything else
- *This MCP server appeared on 14 laptops this week* — shadow adoption in progress

The bitemporal graph (§9.2) gives all of this without extra machinery. The change feed is a byproduct of the graph writer.

### 22.2 The change feed

```
   Graph write -> diff against prior state -> ChangeEvent
       {
         type: EDGE_ADDED | EDGE_REMOVED | NODE_ADDED |
               PROPERTY_CHANGED | CONFIDENCE_CHANGED,
         subject, predicate, object,
         before, after,
         detected_at, source, edge_node_id,
         significance: computed
       }

   Significance is computed, not raw:
     - did this change create a new path to a crown jewel?   -> CRITICAL
     - did it increase an existing path's score by >20?       -> HIGH
     - did it grant privilege where none existed?             -> HIGH
     - did it remove a control (PROTECTS edge)?               -> HIGH
     - is it routine churn (autoscaling, ephemeral pods)?     -> SUPPRESS
```

That last line is essential. A Kubernetes cluster generates thousands of node/edge changes an hour from normal autoscaling. Without suppression by equivalence class, the change feed is unreadable within a day. **Suppress by pattern, not by rate:** learn that pods matching a workload template with an identical service account are one logical entity, and report the template's changes, not each pod's.

---

## 23. Blast radius

Blast radius is the attack path inverted: from a node, what is reachable?

```
   BLAST_RADIUS(node, depth) =
       forward BFS over traversable edges from node
       grouped by what is reached:
         - assets reachable
         - datastores readable / writable  (with sensitivity)
         - identities assumable
         - production systems modifiable
         - external egress available
```

Used in four places, and it is worth noting that this one computation serves all four — good sign the abstraction is right:

1. **Incident response** — "this host is compromised; what does the attacker now have?"
2. **AI agent assessment** — "if this agent is manipulated, what can it do?" (Chapter 27)
3. **Change preview** — "if we grant this role, what does it open up?"
4. **Response impact preview** — "if we quarantine this host, what breaks?" (Chapter 30)

Presentation matters more than computation:

```
  AGENT COMPROMISED: DevOps-Agent

  CAN REACH        14 systems       (3 production)
  CAN READ         3 databases      (1 contains PII, 4.2M records)
  CAN MODIFY       2 applications   (both customer-facing)
  CAN EXECUTE      cloud functions, EC2 instances via SSM
  CAN SEND         external email via Graph API
  CAN ASSUME       4 roles          (1 with Administrator)

  MOST SEVERE CONSEQUENCE:
    read + exfiltrate 4.2M customer records
    via  svc-devops-ai -> DevOpsAdmin -> prod-payments-db
                       -> ses:SendEmail (external)
```

---

# PART V — AI SECURITY

---

## 24. AI discovery

### 24.1 The five discovery surfaces

```
  1. NETWORK        DNS queries, TLS SNI, proxy logs, firewall logs
                    -> which AI services are being reached, by whom
                    Sources: Edge Node connectors (DNS, FW, proxy, SWG)

  2. HOST           processes, installed software, config files, extensions
                    -> local models, MCP servers, IDE assistants, SDKs
                    Sources: Overlook Agent

  3. CLOUD          Bedrock/Azure OpenAI/Vertex resources, model endpoints,
                    vector DBs, AI service accounts, model registries
                    Sources: cloud connectors

  4. CODE           AI SDK imports, API key patterns, prompt templates,
                    agent framework usage, MCP declarations in repos
                    Sources: repo connectors

  5. TRAFFIC        actual API calls with model, tokens, tools, identity
                    Sources: AI Gateway
```

Each surface sees something the others cannot. A complete AI inventory needs at least 1, 2, and 3; 4 and 5 turn inventory into security.

### 24.2 What an AI_APPLICATION node holds

```
  AI_APPLICATION  "GitHub Copilot"
    category            coding_assistant
    provider            GitHub / Microsoft
    hosting             external_saas
    approved            true          (customer-set)
    data_residency      US
    endpoints           api.githubcopilot.com, copilot-proxy.githubusercontent.com
    auth_method         oauth_corporate
    users               tokenized identity list
    first_seen          2026-03-14
    departments         [Engineering, Data]
    sensitive_data_seen true   (from Gateway or Agent inference)
    risk_score          42
```

Fingerprinting (which domains, processes, and code signatures mean which application) is **content**, shipped and updated via the content pipeline (Chapter 17). New AI tools appear weekly; the fingerprint set is a maintained asset and a genuine differentiator if it is comprehensive.

---

## 25. Shadow AI

### 25.1 Detection without a gateway

Most customers will not deploy the gateway on day one, so Shadow AI must work at Level 1–2:

```
  DNS query to  claude.ai            -> AI service contacted
  + source IP  10.4.2.88             -> asset
  + DHCP/agent  10.4.2.88 = LT-4471  -> host
  + logon data  LT-4471 = priyas     -> identity
  + agent proc  Chrome -> claude.ai  -> browser context
  + proxy bytes 2.4 MB uploaded      -> volume signal
  ------------------------------------------------------
  FACT: IDN-9f3a USES AIA-claude
        via AST-77c2, browser, 2.4 MB upload, unapproved
```

No content inspection required. The user-to-AI-service relationship, with volume, is enough to drive the inventory, the approval workflow, and the risk score. Content only becomes necessary to answer *what* was sent — which is Level 3.

### 25.2 The approval workflow

Shadow AI is only useful if it drives a decision:

```
   DISCOVERED  -> new AI application seen
       |
       +--> APPROVE        add to sanctioned list, monitor
       +--> APPROVE W/ CONDITIONS   e.g. no sensitive data, specific teams
       +--> RESTRICT       policy: warn users, block at SWG
       +--> BLOCK          push a block to the customer's proxy/SWG
       |
       v
   Track over time: usage after decision, policy violations,
   users still accessing a blocked service (a real signal —
   usually means a business need the policy ignored)
```

Report to the business, not just the SOC: *"Marketing uses 6 unapproved AI tools; 2 handle customer data."* That framing gets budget. "You have 214 AI findings" does not.

---

## 26. Prompt security

### 26.1 Three privacy modes

The brief's three modes are right. The important detail is that **mode is per-policy, not global** — a customer may want metadata-only for HR and full inspection for the engineering agent platform.

**Mode 1 — Metadata only (default).**
```
  PROMPT_EVENT
    identity      IDN-9f3a...
    application   AIA-chatgpt
    model         gpt-4o
    length_bucket 500-1000
    has_pii       true
    has_secrets   false
    has_code      true
    injection     none_detected
    risk          HIGH
    timestamp     2026-08-12T09:14Z
```
No content anywhere, not even at the Edge.

**Mode 2 — Local inspection (recommended).**
Content is inspected at Edge/Gateway, classified, and **discarded**. Only classification results leave. A hash and a short-TTL encrypted excerpt may be retained locally as evidence, accessible only through the local UI with RBAC and audit.

**Mode 3 — Full capture (off by default).**
Content retained at the Edge, never sent to SaaS. Requires explicit configuration, named approvers, retention limits, encryption, and per-access audit. Some regulated customers need this for their own investigations. It should feel deliberately heavy to enable.

### 26.2 Detection pipeline

```
  SENSITIVE DATA
    regex + validators (Luhn for cards, checksums for national IDs)
    + NER for names/addresses (small local model)
    + customer-defined patterns (employee ID formats, project codenames)

  SECRETS
    provider-specific patterns (sk-..., ghp_..., AKIA...)
    + entropy analysis for unknown formats
    + live validation OPTIONAL and off by default
      (validating a key by calling the provider is itself an exfiltration
       and will be viewed that way — make it opt-in with a clear warning)

  SOURCE CODE
    language detection + internal-repository fingerprint matching
    (hash n-grams of internal repos at the Edge; match without
     ever sending code anywhere)

  DIRECT PROMPT INJECTION
    instruction-override patterns ("ignore previous instructions")
    role-play framing, encoding tricks (base64, unicode confusables,
    zero-width characters), system-prompt extraction attempts

  INDIRECT PROMPT INJECTION      <- the important one
    injection markers found inside RETRIEVED content or TOOL OUTPUT
    rather than user input. This is the attack that actually
    compromises agents, and it is invisible without gateway or
    SDK-level visibility into what the model was fed.
```

### 26.3 Why indirect injection is the one that matters

Direct injection is a user trying to jailbreak a chatbot — mostly a content-policy problem. Indirect injection is an *attacker* placing instructions in a document, a web page, a Jira ticket, or an email that an agent will later read, causing the agent to act with its own privileges.

That is a privilege-escalation attack that uses no vulnerability, and it is exactly what the TrustGraph is built to reason about: the consequence is not "the model said something bad" but "the agent's service account can reach production."

```
   Attacker -> comment in a public issue containing hidden instructions
        |
        v
   Support agent (AI) reads the issue via its GitHub tool
        |
        v
   Instructions cause the agent to call its database tool
        |
        v
   svc-support-ai -> RDS read -> customer records
        |
        v
   Agent's email tool sends them out

   No CVE. No misconfiguration in the traditional sense.
   Only a graph can show this, and only if AI_AGENT, MCP_TOOL, and
   the identity/data layers are in the SAME graph.
```

---

## 27. AI agents and MCP

### 27.1 The agent as a privileged identity

The correct mental model, and it is worth repeating because it drives everything:

> **An AI agent is a non-human identity with a natural-language attack surface.**

Everything already known about service-account security applies — least privilege, credential rotation, monitoring, blast-radius analysis. What is new is that the *instruction channel* is untrusted and unverifiable.

An AI_AGENT node:

```
  AI_AGENT  "DevOps-Agent"
    model              AIM-claude-sonnet
    platform           internal / bedrock / langchain / custom
    owner              IDN-team-platform
    runs_as            IDN-svc-devops-ai        <- the critical link
    tools              [aws-cli, github, jira, slack, filesystem]
    mcp_servers        [MCP-aws, MCP-github]
    autonomy           high | approval_required | read_only
    triggered_by       [IDN-* engineering group]
    data_access        via runs_as identity's permissions
    external_egress    true
    deployed_since     2026-05-02
```

`runs_as` is the edge that makes everything work. It connects the AI layer to the identity layer, and through it to cloud, data, and network. Without it, agent security is a standalone dashboard; with it, agent security is exposure management.

### 27.2 MCP: the new supply chain

MCP servers are, from a security perspective, **plugins with credentials, installed by end users, from unvetted sources**. The historical analogue is browser extensions, and that went badly for a decade.

What Overlook discovers:

```
  From the Agent (local configs):
    ~/.claude/claude_desktop_config.json
    ~/.cursor/mcp.json
    VS Code settings, Windsurf, Zed, and equivalents
    -> server name, command, args, env (credential PRESENCE, never values)

  From repos:
    mcp.json / .mcp/ declarations committed to source

  From the network:
    connections to known remote MCP endpoints

  From the Gateway:
    actual tool invocations, with arguments classified
```

MCP-specific risks worth modelling explicitly:

| Risk | Description | Detection |
|---|---|---|
| Over-scoped filesystem | `mcp-filesystem` rooted at `/` or a home directory | Config parse |
| Credential exposure | Long-lived tokens in MCP env config | Config parse (presence + type) |
| Unvetted server | Server from an unknown npm/PyPI package | Reputation content feed |
| Tool poisoning | Malicious instructions embedded in a tool's description | Gateway inspection of tool schemas |
| Confused deputy | Agent invokes a tool on behalf of an unauthorised user | Gateway + identity correlation |
| Shadow MCP | Server on a laptop nobody approved | Agent discovery |

The graph makes the consequence concrete:

```
   AI_AGENT  "Assistant"
       INVOKES -> MCP_SERVER "mcp-filesystem"
                     CONTAINS -> /Users/priya/work/
                                    CONTAINS -> finance/payroll-2026.xlsx
                                                   STORES -> DATA_CLASS PII

   FINDING: agent has unrestricted read of a directory containing
            PII, via an unapproved MCP server, on 14 hosts.
```

### 27.3 Tool-call modelling

```
   PROMPT --> AI_AGENT --> MCP_TOOL --> IDENTITY --> ACTION --> ASSET
```

Each hop is a graph edge, so a tool call becomes a traversable path — which means the attack-path engine handles AI attacks with no special-casing. That is the payoff of putting AI in the same graph rather than a separate module.

---

## 28. RAG security

RAG systems create a data-access path that bypasses conventional access control: a document a user cannot open may still be retrievable through a RAG application that indexed it with a service identity.

```
   THE CORE PROBLEM

   Direct access:
      Priya --X--> /finance/salaries.xlsx      (ACL denies)

   Via RAG:
      Priya --> HR-Assistant --> vector index --> chunk of salaries.xlsx
                                                   -> answer contains salary data

   The ACL was enforced on the file. It was NOT enforced on the
   embedding of the file.
```

What Overlook models:

```
  RAG_APPLICATION "HR-Assistant"
      RETRIEVES_FROM -> VECTOR_DATABASE "hr-index"
                            CONTAINS -> source documents
                                          STORES -> DATA_CLASS [PII, HR_CONFIDENTIAL]
      runs_as        -> IDENTITY svc-hr-assistant
      accessible_by  -> GROUP "All Employees"        <- the finding
      permission_model -> none | document_acl | post_filter
```

The finding writes itself: *the retrieval identity has broader access than the users who can query it, and the retrieval layer does not enforce per-user ACLs.*

Other RAG conditions to detect:
- Sensitive documents indexed without corresponding query-side restriction
- Vector DB publicly reachable or lacking authentication
- Cross-tenant retrieval possible in a multi-tenant index
- Documents from an untrusted source indexed into a trusted index (**data poisoning / indirect injection vector**)
- Excessive retrieval volume by one identity (bulk extraction through the RAG interface)

---

## 29. TRACE 3 — the AI Privilege Gap

The product's flagship finding, traced from raw collection to the screen.

```
  COLLECTION

  T0  Repo connector reads github.com/corp/devops-agent
        - detects framework imports (langchain, anthropic SDK)
        - finds agent definition with 6 declared tools
        - finds deployment manifest -> service account svc-devops-ai
      FACTS:  AI_AGENT AGT-devops exists
              AGT-devops RUNS_AS IDN-svc-devops-ai
              AGT-devops INVOKES MCP-aws, MCP-github, TOOL-slack, ...

  T1  AWS IAM connector, permission closure computed AT THE EDGE
        svc-devops-ai -> role/DevOpsAdmin -> AdministratorAccess
      FACTS:  IDN-svc-devops-ai CAN_ASSUME ROL-devopsadmin (conf 0.99)
              ROL-devopsadmin CAN_READ  DST-prod-payments (unconditional)
              ROL-devopsadmin CAN_WRITE DST-prod-payments (unconditional)
              ROL-devopsadmin CAN_EXECUTE AST-* (via SSM)

  T2  AWS IAM connector, for the human
        priya.s -> policy: s3:GetObject on one bucket. Nothing else.
      FACTS:  IDN-9f3a CAN_READ DST-artifacts (conf 0.99)

  T3  Agent (on Priya's laptop) + Slack connector
        Priya's laptop runs the agent CLI; Slack shows her invoking
        the devops-agent bot 40 times in 30 days
      FACTS:  IDN-9f3a CAN_INVOKE AGT-devops (conf 0.94, observed)

  T4  DSPM scanner classifies the target
      FACTS:  DST-prod-payments STORES DATA_CLASS [PII, PCI]
              record_count_bucket "1M-10M"
              criticality 95 (customer-designated crown jewel)

  CORRELATION IN SaaS

  Rule AI-PRIVILEGE-GAP fires when:
      identity I  CAN_INVOKE  agent A
      A           RUNS_AS     identity S
      effective_privilege(S) >> effective_privilege(I)
      AND S reaches a crown jewel that I cannot reach directly

  Evaluation:
      effective_privilege(IDN-9f3a)          = READ, 1 bucket, 0 crown jewels
      effective_privilege(IDN-svc-devops-ai) = ADMIN, 340 resources,
                                               3 crown jewels
      gap = ADMIN - READ = CRITICAL
      crown jewels newly reachable = 3

  THE FINDING (after de-tokenization in the analyst's browser)

  ================================================================
   AI PRIVILEGE GAP                                    CRITICAL
  ================================================================
   Priya S (Developer, Engineering)
     direct AWS privilege:            READ  (1 bucket)
     effective privilege via agent:   ADMIN (340 resources)

   Path:
     Priya S
       -> invokes -> DevOps-Agent            (40 invocations / 30 days)
       -> runs as -> svc-devops-ai
       -> assumes -> DevOpsAdmin (AdministratorAccess)
       -> reads   -> prod-payments-db  [PII + PCI, 1M-10M records]

   Also newly reachable: prod-identity-store, prod-backup-vault

   Exists since: 2026-05-02  (102 days)
   Confidence:   0.94  (invocation observed via Slack, not enforced)

   RECOMMENDED FIX
     1. Replace AdministratorAccess on svc-devops-ai with a scoped
        policy. Estimated: removes 3 crown jewels from reach,
        eliminates 1,240 paths.
     2. Require human approval for agent actions touching production.
     3. Restrict who can invoke DevOps-Agent to the platform team (4 users).

   IMPACT PREVIEW OF FIX 1
     - 3 pipelines use this role; 2 need policy updates
     - 14 automated jobs affected
     - no user-facing service impact
  ================================================================
```

Nothing in that finding requires a vulnerability, a CVE, a malicious actor, or a detection. It is a **structural** exposure — the kind that exists for months, that no scanner reports, and that only a cross-domain graph can surface. That is the product.

---

# PART VI — ACTION

---

## 30. Response

### 30.1 The trust chain

The design principle in the original brief is correct and should be defended against every future convenience argument:

> **Overlook SaaS never controls an agent directly. Every command passes through the customer's Edge Node, which may refuse it.**

Post-2024, the industry has learned what happens when a vendor can push arbitrary instructions to every endpoint simultaneously. The customer-side veto is not bureaucracy; it is the feature that makes response deployable in a regulated environment.

```
   Analyst clicks "Quarantine host"        [ Overlook SaaS ]
        |
        v
   RBAC check: does this user hold the response role for this scope?
        |
        v
   Impact preview computed from blast radius
        |
        v
   Approval: four-eyes if policy requires
        |
        v
   Command constructed:
        { action, target_token, ttl, nonce, issued_at, expires_at,
          issued_by, approved_by, justification, rollback_policy }
        |
        v
   Signed by SaaS response key (Ed25519)
        |
        v
   Placed in the command queue
        |
   ============ Edge polls and retrieves ============
        |
        v
   [ EDGE NODE — the customer's control point ]
        |
        +-- verify signature against the pinned SaaS key
        +-- verify nonce is unused (replay protection)
        +-- verify not expired
        +-- check LOCAL policy:
        |     is this action class enabled on this Edge?
        |     is the target in an allowed scope?
        |     is it inside a maintenance window?
        |     is the target on the protected-asset list?
        |       (domain controllers, ICS, life-safety systems —
        |        NEVER quarantined automatically, no exceptions)
        +-- resolve target token -> real host  (only the Edge can do this)
        +-- LOCAL approval if configured (customer may require its own
        |     second approval independent of SaaS)
        |
        v
   Re-signed with the EDGE key, scoped to one agent, short TTL
        |
        v
   [ AGENT ]  (polls; no inbound port)
        +-- verify Edge signature against pinned Edge key
        +-- verify TTL and nonce
        +-- pre-flight: is the target still what we think it is?
        |     (process still running with the same hash? host still
        |      matches the identifiers in the command?)
        +-- EXECUTE
        +-- verify result
        +-- report outcome
        +-- start rollback timer
```

Note the **double signature** and the **token resolution at the Edge**. SaaS issues a command against a token; it does not and cannot know which physical machine that is. Only the Edge resolves it. This means a compromise of Overlook SaaS cannot target a specific customer machine — an attacker would be issuing commands against opaque identifiers, subject to local policy, with local approval, inside maintenance windows, excluding protected assets.

### 30.2 The four actions

Start with exactly these. Each is narrow, reversible, and locally verifiable.

**Quarantine host.** Isolate network access except to the Edge Node.
```
   Implementation: local firewall rules (Windows Filtering Platform,
   nftables, pf) — allow only Edge Node IP:port, deny all else.
   TTL MANDATORY. Default 4 hours, max 24.
   Auto-release on TTL expiry EVEN IF the Edge is unreachable —
   the agent's own timer releases it. A quarantine that survives a
   management-plane outage is an outage of the customer's business.
   Protected-asset list is absolute.
```

**Terminate process.** Kill a specific process.
```
   Command targets PID + process hash + start time, not just PID
   (PIDs are reused; killing the wrong process is how you take
   down a database).
   Pre-flight verifies all three still match. Mismatch -> refuse.
   Not reversible — so it requires higher approval than quarantine.
```

**Block connection.** Targeted local firewall rule.
```
   Target: IP, IP+port, or subnet. TTL mandatory.
   Lowest-risk action; good candidate for the first automation.
```

**Lock account / terminate session.** Disable a local account or kill a session.
```
   Prefer doing this at the IdP via connector (broader effect,
   fully reversible, audited centrally) over doing it locally.
   Local action only for local accounts that the IdP does not manage.
```

### 30.3 Universal requirements

Every response action, without exception:

| Requirement | Rationale |
|---|---|
| Impact preview before execution | Blast radius of the *response*. Quarantining a domain controller is worse than the incident. |
| TTL and auto-rollback | Failure mode must be "control expires", never "control persists forever". |
| Nonce | Replay protection. A captured command must not be re-executable. |
| Dual signature | SaaS authorises; Edge authorises; agent verifies both. |
| Local policy veto | Customer's Edge can refuse any class, and SaaS cannot override. |
| Protected-asset list | Absolute deny for DCs, ICS/OT, medical, life-safety. |
| Maintenance-window awareness | No automated action during a change freeze. |
| Full audit | Who, what, when, why, approved by whom, outcome, rollback. Exportable. |
| Verification | Did it actually work? Report the observed state, not the intent. |
| Dry-run | Every action must support a no-op mode that reports what would happen. |

### 30.4 Automation posture

Do not ship full automation early. Graduate:

```
  Stage 1   RECOMMEND ONLY     Overlook suggests; human executes elsewhere
  Stage 2   ONE-CLICK          Human clicks; Overlook executes
  Stage 3   APPROVAL WORKFLOW  Automation proposes; human approves
  Stage 4   AUTOMATED, SCOPED  Automatic within narrow, customer-defined
                               boundaries (e.g. "block connections to
                               known-malicious IPs on non-production hosts")
```

Most customers will never enable Stage 4, and that is fine. The value is in knowing *what* to do, which is a graph problem, not in doing it, which is a commodity.

---

## 31. Break Attack Path

### 31.1 The differentiating response

Traditional response is *reactive* — something bad happened, contain it. Overlook's distinctive response is **preventive**: the graph knows which single change eliminates the most exposure.

```
   CHOKE POINT IDENTIFIED

   Edge: ROL-ghadeploy CAN_ASSUME ROL-ec2app

   Appears in:   1,240 attack paths
   Of which:     9 reach crown jewels
                 3 rated CRITICAL

   PROPOSED CHANGE
     Remove sts:AssumeRole for role/EC2AppRole from GHADeployRole's
     identity policy.

   PREDICTED EFFECT (simulated on the graph before any change)
     paths eliminated       1,240
     crown jewels protected 3
     exposure score change  -18 points
     residual paths to same crown jewels: 2 (via a different route)

   OPERATIONAL IMPACT (from blast radius of the change itself)
     3 pipelines currently use this assumption:
        gha-deploy-prod       -> will break, needs new role
        gha-deploy-staging    -> will break, needs new role
        gha-smoke-test        -> unaffected (uses a different path)
     14 automated jobs run through these pipelines
     estimated remediation effort: 2-4 hours

   OUTPUT OPTIONS
     [ Copy IAM policy diff ]  [ Open Jira ticket ]  [ Create PR ]
     [ Apply via connector ]                 <- optional, off by default
```

### 31.2 Graph simulation before change

The ability to answer *"what happens if I make this change?"* before making it is a direct consequence of holding the graph. Implementation: apply the proposed edge change to an in-memory overlay, recompute affected paths, diff.

This is genuinely valuable and largely unavailable elsewhere, because it requires exactly what Overlook has — a complete, cross-domain, permission-resolved graph. It should be a headline capability, not a hidden feature.

---

# PART VII — OPERATIONS

---

## 32. TRACE 4 — an analyst investigating

The de-tokenization flow (§7.4) in practice, since it is the least conventional part of the system.

```
  09:14  Analyst opens Overlook SaaS. Dashboard shows:
           "3 NEW critical paths in the last 24h"

  09:14  Browser -> SaaS: GET /v1/paths?severity=critical&status=new
         SaaS returns paths as TOKENS:
           IDN-9f3a -> AGT-4c8d -> IDN-svc7e2 -> ROL-2b8e -> DST-1c4b

  09:14  Browser collects the 47 unique tokens on screen.
         Browser -> Edge Node (DIRECT, not via SaaS):
           POST https://overlook-edge.corp.local/api/v1/resolve
           Authorization: Bearer <SaaS-issued JWT>
             { sub: "analyst@corp.com", scope: ["resolve:identity",
               "resolve:asset","resolve:datastore"], exp: +5min,
               aud: "EDGE-ap-south-1-a" }
           Body: { tokens: [ 47 tokens ] }

  09:14  Edge validates JWT signature (SaaS key pinned at enrollment),
         audience, expiry, and the analyst's LOCAL RBAC.
         Analyst has resolve:identity but NOT resolve:prompt_content.
         Edge returns names for 47 tokens; would refuse prompt content.
         Edge writes an audit record: who, which tokens, when.

  09:14  Browser renders:
           Priya S -> DevOps-Agent -> svc-devops-ai
             -> DevOpsAdmin -> prod-payments-db

  09:16  Analyst clicks the edge "svc-devops-ai CAN_ASSUME DevOpsAdmin"
         to see why Overlook believes it.
         Browser -> SaaS: gets evidence_ref sha256:8a1f...c4d2
         Browser -> Edge: GET /api/v1/evidence/sha256:8a1f...c4d2
         Edge returns the actual IAM policy document.
         SaaS never saw it. Audit record written.

  09:18  Analyst marks the path "confirmed, remediating".
         Browser -> SaaS: status update (no customer data involved).

  09:20  Analyst clicks "Break this path".
         Impact preview computed in SaaS on tokens.
         De-tokenized in browser for display.
         Analyst exports the IAM policy diff and opens a ticket.

  IF THE EDGE IS UNREACHABLE (analyst at home, no VPN):
    Browser shows degraded mode:
       [IDENTITY · human · Engineering · not privileged]
         -> [AI_AGENT · high autonomy]
         -> [IDENTITY · service account · ADMIN]
         -> [DATASTORE · PII+PCI · 1M-10M records · crown jewel]
    Banner: "Names unavailable — not connected to your Overlook Edge."
    Triage is possible. Action is not. This is by design.
```

The degraded view is more useful than it first appears — the *shape* of the finding is intact, which is often enough to decide urgency. Build it deliberately rather than treating it as an error state.

---

## 33. Securing Overlook itself

A platform holding a complete map of a customer's attack surface is the single most valuable target in their environment. If it is compromised, the attacker gets a prioritised list of how to reach everything.

This must be addressed proactively, in writing, before the first customer's security team asks:

**Edge Node**
- Minimal, immutable base image; read-only root filesystem where possible
- No inbound ports except the local UI and agent gateway, both mTLS
- Credential vault separate from connector processes, KMS/HSM-backed
- Automatic security patching with staged rollout
- Local audit log, tamper-evident (hash-chained), exportable to the customer's SIEM
- Full-disk encryption; the token mapping table encrypted separately

**Agent**
- No listening port, outbound-only
- Signed binaries, EV code-signing
- Command validation requiring two independent signatures
- Response capability disabled by default and separately installed
- Published resource limits, self-enforced

**SaaS**
- Per-tenant encryption keys and data isolation
- No customer credentials held, ever
- No inbound access to customer environments, ever
- Response signing keys in an HSM with separated duties
- Regional deployment for data residency
- Complete audit trail, customer-exportable

**The strongest statement available, and it should lead the whitepaper:**

> Even a total compromise of Overlook SaaS yields: tokenized identifiers, relationship types, and risk scores. No names, no credentials, no data, no network access to any customer, and no ability to command any specific machine.

Very few security vendors can make that claim. It is worth the architectural discipline required to keep it true — and it is worth having an independent third party verify it.

---

## 34. Product surfaces

### 34.1 CISO dashboard

```
  OVERLOOK

  Exposure Score          72  (-8 this month)
  Critical Attack Paths   6   (2 new)
  Crown Jewels at Risk    3 of 47
  Active Response         1

  Coverage:  Cloud 94% | Apps 71% | Data 88% | Identity 97%
             Network 62% | Runtime 45% | AI 78%

  Top 3 actions this month:
    1. Scope GHADeployRole  -> eliminates 1,240 paths
    2. Remove admin from svc-devops-ai -> protects 3 crown jewels
    3. Restrict HR-Assistant retrieval -> closes 1 data exposure

  Trend: exposure down 11% since May
```

Coverage percentages are important and unusual. They tell the customer *how much Overlook cannot see*, which builds trust and drives expansion naturally: "Runtime 45%" is an honest gap statement and also a sales motion.

### 34.2 Analyst workspace

Priorities, in order: the path view (the graph, rendered as a narrative, not a hairball); evidence drill-down; change feed; investigation timeline; and the response console.

A note on graph visualisation: **do not render the full graph.** A 2-million-node force-directed diagram is a demo, not a tool. Render paths as linear narratives with branch indicators, and offer neighbourhood exploration two hops at a time.

### 34.3 AI security console

```
  AI SECURITY

  AI Assets 47   Shadow AI 12   Agents 8   Risky Prompts 34   Critical AI Paths 3

  Navigation:  Inventory | Shadow AI | Prompts | Data | Agents | MCP | RAG | Paths
```

One page with a drawer for detail beats seven separate pages. Every item links back into the main TrustGraph — AI security is a lens on the graph, not a separate product.

---

## 35. Scale and cost model

```
  Per large tenant, steady state:

  Edge Node:  2.4 TB/day ingested, 12 MB/day egress to SaaS
  SaaS:       ~8M nodes, ~120M edges
              ~40k graph writes/day after dedup
              path recomputation: incremental, ~200ms per change event
              full recompute: nightly, ~25 min in a per-tenant worker

  Cost drivers, in order:
    1. Attack-path computation (CPU, bursty)   -> per-tenant workers, capped
    2. Graph storage (edges dominate)          -> tombstone compaction after 90d
    3. Change-feed processing                  -> suppression by equivalence class
    4. Ingest verification (signature checks)  -> batch verification

  What is NOT a cost driver, unusually:
    Data volume from customers. 12 MB/day/tenant means bandwidth and
    ingest storage are effectively free. This is a direct economic
    consequence of the privacy architecture, and it is a real margin
    advantage over log-based competitors whose COGS scale with
    customer data volume.
```

Worth stating plainly to investors: the privacy architecture is not only a compliance story, it is a **gross-margin story**. Competitors pay to store customer data; Overlook does not.

---

## 36. Open decisions

Things this document does not settle. Each needs an explicit decision before build.

| # | Decision | Options | Lean |
|---|---|---|---|
| 1 | Graph storage | Postgres / Neo4j / custom | Postgres for v1, interface-isolated |
| 2 | Edge Node packaging | OVA + AMI / container / installer | OVA + AMI + container; Windows installer later |
| 3 | Resolution Directory hosting | Primary Edge / dedicated service / customer DB | Primary Edge for v1 |
| 4 | Own agent vs EDR-only for v1 | Build thin agent / integrate only | Integrate only for v1; thin agent for AI/MCP discovery in v2 |
| 5 | AI Gateway first mode | Reverse proxy / SWG integration / extension / SDK | Reverse proxy + SDK first |
| 6 | Prompt default mode | Metadata / local inspection | Metadata-only default; inspection opt-in |
| 7 | Token rotation strategy | Never / scheduled / on-demand dual-token | On-demand dual-token window |
| 8 | Multi-region | Single region v1 / multi from start | Multi from start — the target market requires it |
| 9 | Response in v1 | Ship / defer | Defer execution; ship recommend-only |
| 10 | Pricing unit | Per asset / per identity / per Edge / platform | Per identity + asset band; avoid data-volume pricing (it contradicts the architecture) |
| 11 | Confidence in the UI | Show always / show on drill-down | Show always — it is a trust feature, not a caveat |
| 12 | Cross-tenant intelligence | Opt-in / opt-out / none | Opt-in, aggregate only, published methodology |

---

## 37. What breaks first

An honest pre-mortem. Each of these has killed a comparable product.

1. **Entity resolution quality.** If the graph merges wrongly, every finding is suspect and trust never recovers. *Mitigation:* bias toward under-merge, ship the Resolution Review queue in v1, show confidence everywhere, make merges reversible with one click.

2. **Connector rot.** Vendors change formats; connectors silently degrade; the graph quietly becomes fiction. *Mitigation:* parse-rate and field-presence monitoring, coverage windows, staged content rollout with automatic rollback.

3. **Path explosion and noise.** 47,000 paths is the same failure as 60,000 vulnerabilities. *Mitigation:* choke points as the primary object, equivalence classes, crown-jewel scoping, and a hard rule that no view shows more than ~50 items.

4. **The Edge Node becoming an operational burden.** If customers must babysit an appliance, they will not renew. *Mitigation:* aggressive self-monitoring, automatic recovery, honest health reporting, and a support model that works without receiving customer data.

5. **De-tokenization friction.** If analysts cannot see names easily, they will not use the product. *Mitigation:* build the browser-to-Edge resolve path in v1, not v3; make degraded mode genuinely usable; solve the remote-analyst case explicitly (DMZ Edge or customer-hosted relay).

6. **Scope explosion.** Eight domains, 25 components, one team. *Mitigation:* Chapter 1.4 and the five findings in 38.4. Re-read them whenever a roadmap meeting adds a domain.

7. **The AI Gateway's deployment friction** being underestimated, consuming quarters, and delivering a commodity DLP feature. *Mitigation:* SWG integration and SDK first; treat the inline proxy as a later, optional mode.

---

## 38. Build model

Overlook is built as **one system, in one program of work**. There are no delivery phases, no wedge release, and no capability held back for a later stage. The architecture is designed so that parallel tracks can proceed simultaneously against stable contracts, and the sequencing that does exist is dependency ordering measured in weeks of lead time, not roadmap stages measured in quarters.

### 38.1 What makes parallel construction possible

Three contracts decouple the work. Once they are fixed, every track can proceed without waiting on the others:

```
   THE SECURITY FACT SCHEMA (Ch. 5)
      decouples the Collection Plane from the Intelligence Plane.
      Connector authors and graph engineers never block each other.

   THE ENTITY MODEL AND PREDICATE VOCABULARY (Ch. 6)
      decouples every connector from every other connector, and
      decouples all of them from the attack path engine.

   THE COMMAND AND CONTENT ENVELOPES (Ch. 15, 17)
      decouple the Edge Node from SaaS release cycles, and decouple
      content (parsers, primitives, fingerprints) from code entirely.
```

**These three must exist before anything else is built.** They are the only true prerequisite in the system, and they are a matter of weeks, not quarters.

### 38.2 The parallel tracks

```
  PLATFORM        Edge Node runtime — the five roles, transport,
                  enrollment, credential vault, local UI, sync,
                  offline buffering, health

  CONNECTORS      the framework and the fleet
                  (see 03-connectors.md — five sub-tracks, 118 connectors)

  GRAPH           entity resolution, Resolution Directory, tokenization,
                  the bitemporal graph, coverage windows, change feed

  IAM             permission closure per cloud, escalation primitive
                  engine, directory depth, CIEM
                  (see 02-iam-deep-dive.md)

  ANALYTICS       attack path engine, risk scoring, choke points,
                  blast radius, graph simulation

  AI              AI entity discovery, agent and MCP modelling,
                  the AI Gateway, prompt inspection

  ENDPOINT        the Overlook Agent — collection and response executor

  RESPONSE        the signed command chain, approvals, impact preview,
                  Break Attack Path

  CONTENT         iam-semantics, escalation primitives, parsers,
                  AI fingerprints, MCP reputation, classification patterns
                  — ships independently of all code

  EXPERIENCE      SaaS console, local UI, de-tokenization path,
                  onboarding, investigation workspace
```

### 38.3 The lead-time dependencies

Not sequencing — lead times. Each is a matter of a few weeks of head start, not a gate that stops other work.

```
   Contracts (38.1)         must lead everything                ~4 weeks
   Connector framework      must lead the connector fleet       ~6 weeks
   Entity resolution        must lead the path engine           ~4 weeks
   Permission closure       must lead escalation primitives     ~4 weeks
   Security Fact + graph    must lead the SaaS console          ~4 weeks
   Agent transport          must lead response execution        ~6 weeks

   Everything else runs concurrently.
```

### 38.4 The rule that keeps it honest

> Every track must produce something that contributes to a finding no other tool in the customer's environment can produce. Work that only makes the platform more capable, without changing what the customer can learn, is infrastructure — necessary, but never the measure of progress.

The five findings the whole system exists to produce:

```
  1. AI PRIVILEGE GAP     user privilege < agent privilege, reaching a crown jewel
  2. ESCALATION TO ADMIN  a synthesized path that no policy declares
  3. HYBRID PATH          on-prem AD -> Entra -> cloud -> production data
  4. CHOKE POINT          one policy line, thousands of paths
  5. RIGHTSIZE            340 permissions granted, 12 used, policy attached
```

If a piece of work does not eventually serve one of those five, it should be challenged.

---

## Appendix A — Reading the original brief against this document

| Original brief | This document | Change |
|---|---|---|
| Edge Node: 28 responsibilities | 5 roles with distinct resource profiles | Decomposed |
| "Tokenized identity IDs" | Deterministic HMAC with customer-held key | Specified; makes multi-Edge possible |
| De-tokenization | Browser-to-Edge direct resolve | **Added** — was missing entirely |
| Entity resolution at Edge | + Resolution Directory across Edges | **Added** — required for multi-Edge |
| Attack paths in SaaS | Permission closure at Edge, resolved edges + conditions to SaaS | Clarified |
| Agent: broad EDR-like collection | Thin agent, AI/MCP-specific; EDR via API | Narrowed |
| AI Gateway as a deployment tier | Four modes; SWG/SDK before inline proxy | Re-sequenced |
| Change Intelligence (one line) | Bitemporal graph with coverage windows | Specified |
| 25 components | One product (the graph) with capability areas | Reframed |
| No build model | One program, parallel tracks on fixed contracts | **Added** |
| No content pipeline | Signed content bundles with staged rollout | **Added** |
| No multi-tenancy or scale model | Specified with planning numbers | **Added** |

---

## Appendix B — Glossary quick reference

`Canonical Key` — source-independent identifier for an entity, priority-ordered.
`Choke Point` — an edge appearing in many paths; removing it breaks all of them.
`Coverage Window` — proof that a source ran completely, enabling safe tombstoning.
`Crown Jewel` — customer-designated critical asset; the target of path computation.
`Edge Node` — the customer-side appliance.
`Evidence Reference` — hash pointing at raw evidence retained at the Edge.
`Permission Closure` — resolved effective permissions after policies, boundaries, and conditions.
`Privacy Gate` — the component deciding what may leave the customer environment.
`Resolution Directory` — tenant-wide alias-to-canonical-key mapping shared across Edge Nodes.
`Security Fact` — the only unit of data crossing from Edge to SaaS.
`Token` — deterministic tenant-keyed pseudonym.
`TrustGraph` — the unified bitemporal graph in SaaS.

---

*End of document.*

# Overlook — The Contracts

**Version:** 1.0.0-draft
**Date:** 2026-08-14
**Status:** Normative. This document is the authority.
**Supersedes:** all schema fragments in docs 01–12 and the engine/connector series. Where they disagree with this document, this document wins. Part XI lists every reconciliation.

---

# PART I — HOW TO USE THIS DOCUMENT

## 1. What this is

The eight contracts that every other component keys on. They are the **seams** between parallel tracks: once these are fixed, connector work, engine work, storage work and UI work proceed independently without touching each other.

```
  1  Entity model                  Part II
  2  Predicate vocabulary          Part III
  3  Canonical key rules           Part IV
  4  Capability vocabulary         Part V
  5  Observation schema            Part VI
  6  Security Fact schema          Part VII
  7  Connector manifest schema     Part VIII
  8  Privacy policy schema         Part IX
```

## 2. The three that are expensive to reverse

Changing these re-partitions data that already exists. They deserve disproportionate scrutiny before anything depends on them.

```
  ✕ SIGNIFICANT ATTRIBUTES per predicate       (§III.4)
      defines fact identity. Changing it re-partitions every fact
      ever built.

  ✕ CANONICAL KEY PRIORITY RULES               (Part IV)
      must be byte-identical on every Edge Node in a deployment.
      Divergence fragments the graph SILENTLY — no error, no alert,
      just attack paths that quietly do not exist.

  ✕ TOKENIZATION INPUT                          (§IX.3)
      tokens are HMAC(deployment_key, canonical_key). Change how a
      canonical key is derived and every token for that entity type
      changes, orphaning the entire history.
```

## 3. Versioning

```
  SCHEMA VERSION      overlook.contracts.v1
  ENUM ADDITIONS      minor bump. Backwards compatible.
  ENUM REMOVALS       major bump. Requires a migration plan.
  FIELD ADDITIONS     minor bump. Consumers MUST preserve unknown
                      fields rather than rejecting them.
  FIELD REMOVAL       major bump.
  SEMANTIC CHANGE     major bump, even if the shape is unchanged.
                      Redefining what CAN_READ means is a breaking
                      change to every consumer.

  Every Edge Node asserts its contract version hash on sync.
  A mismatch between nodes in one deployment is an ALARM, not a
  warning — it is the precondition for silent graph fragmentation.
```

## 4. The closed-enum rule

Entity types, subtypes, predicates and capabilities are **closed enumerations**. A connector cannot invent `can_read_maybe`. Adding a value requires:

1. a written justification of what it expresses that no existing value does
2. a decision on traversability and significance
3. a schema version bump
4. conformance fixtures

This is deliberate friction. Every predicate multiplies path-engine complexity.

---

# PART II — ENTITY MODEL

## 5. Node types

Twenty-four types. Subtypes are also closed.

### 5.1 Identity layer

```
IDENTITY
  human_user            workforce person, IdP-managed, phishable
  service_account       long-lived, key-based, non-human
  workload_identity     IRSA / Managed Identity / GCP WIF / SPIFFE
  application_sp        OAuth app or service principal with its own perms
  machine_account       AD computer object — a principal, often forgotten
  cicd_identity         GitHub OIDC, GitLab JWT, Jenkins credential
  agent_identity        an AI agent's runtime identity          ← new class
  break_glass           emergency access account
  guest_user            B2B / external / partner
  database_user         database-local principal, no IdP presence
  device_admin          network or storage device administrative account

GROUP
  (no subtypes — provider distinguished by properties)

ROLE
  assumable_role        AWS role, GCP SA impersonation target
  directory_role        Entra directory role, admin role
  permission_set        AWS Identity Center permission set
  database_role         database-internal role
  k8s_role              Role or ClusterRole

CREDENTIAL
  api_key · access_key · ssh_key · certificate · token · password
```

### 5.2 Compute layer

```
ASSET
  host                  physical or virtual machine
  vm                    hypervisor-managed VM
  container             running container instance
  function              serverless function
  cluster               K8s / compute cluster
  node                  K8s node
  endpoint              user endpoint (laptop, workstation)
  network_device        firewall, switch, router, load balancer host
  storage_appliance     NAS / SAN

APPLICATION
  (deployed application or service, distinct from the asset it runs on)

WORKLOAD
  (K8s Deployment/DaemonSet/StatefulSet/Job/CronJob — the template,
   not the pod. Pods are ASSET/container.)
```

### 5.3 Code layer

```
REPOSITORY
PIPELINE                CI/CD pipeline or workflow definition
ARTIFACT                container image, package, build output
```

### 5.4 Data layer

```
DATASTORE
  database              relational or NoSQL instance
  bucket                object storage container
  file_share            SMB/NFS share
  table                 table, collection or index within a datastore
  vector_store          vector database index or collection
  drive                 SharePoint site, Drive, Box folder tree
  warehouse             Snowflake / BigQuery / Redshift / Synapse

DATA_CLASS
  (PII · PCI · PHI · SECRET_MATERIAL · SOURCE_CODE · CONFIDENTIAL ·
   customer-defined. Attached via STORES, not a subtype.)
```

### 5.5 Network layer

```
NETWORK
  vpc · vnet · subnet · vlan · zone · segment · on_prem_range

FIREWALL                policy enforcement point
SECURITY_GROUP          cloud-native rule group (SG, NSG, firewall rule)
LOAD_BALANCER
DNS_NAME
FLOW_AGGREGATE          aggregated observed traffic, not a real thing —
                        a first-class node so CONNECTS_TO has an anchor
```

### 5.6 AI layer

```
AI_APPLICATION          ChatGPT, Copilot, an internal LLM app
AI_MODEL                a specific model, hosted somewhere
AI_AGENT                an agent with tools and a runtime identity
MCP_SERVER
MCP_TOOL
RAG_APPLICATION
PROMPT_EVENT            an aggregated prompt observation, never content
```

⚠ **`VECTOR_DATABASE` is not a type.** It is `DATASTORE` / `vector_store`. See §XI.1.

### 5.7 Posture layer

```
VULNERABILITY           a CVE instance on an asset
MISCONFIGURATION        a rule-detected configuration state
SECRET                  a credential instance found somewhere
                        (distinct from CREDENTIAL: SECRET is the
                         discovered artifact, CREDENTIAL is the
                         authentication mechanism it represents)
SECURITY_CONTROL        an EDR sensor, a WAF, a CA policy, a NetworkPolicy
COMPLIANCE_REQUIREMENT
```

### 5.8 Organisation layer

```
ACCOUNT                 cloud account / subscription / project / tenant
ORG_UNIT                OU, folder, management group, AD OU
BUSINESS_SERVICE        a business service from the CMDB — the
                        customer's own criticality designation
```

## 6. Required node attributes

Every node, regardless of type:

```jsonc
{
  "canonical_key": "email:priya.s@meridian.com",   // plaintext, appliance only
  "type": "IDENTITY",
  "subtype": "human_user",
  "properties": { },                                // type-specific
  "criticality": 0,                                 // 0-100, default 0
  "first_seen": "2026-06-02T09:14:22Z",
  "last_seen":  "2026-08-14T04:00:11Z",
  "removed_at": null,
  "confidence": 1.0,                                // resolution confidence
  "sources": ["ad", "entra", "aws", "crowdstrike"],
  "resolution_method": "deterministic"              // deterministic |
                                                    // probabilistic |
                                                    // reinforced | manual
}
```

---

# PART III — PREDICATE VOCABULARY

## 7. The twenty-two predicates

| Predicate | Meaning | Traversable | Base weight |
|---|---|---|---|
| **Identity and privilege** | | | |
| `MEMBER_OF` | membership in a group or role | ✓ | 0.99 |
| `CAN_ASSUME` | can become this identity | ✓ | 0.95 |
| `AUTHENTICATES_TO` | uses this asset or service with credentials | ✓ | 0.70 |
| `TRUSTS` | federation or cross-domain trust | ✓ | 0.85 |
| **Capability** | | | |
| `CAN_READ` | can read data from | ✓ | 0.90 |
| `CAN_WRITE` | can modify data in | ✓ | 0.85 |
| `CAN_EXECUTE` | can run code on | ✓ | 0.90 |
| `CAN_MODIFY` | can change the configuration of | ✓ | 0.85 |
| `CAN_DEPLOY` | can push code or artifacts to | ✓ | 0.80 |
| `CAN_ACCESS` | can use this application or service | ✓ | 0.75 |
| **Structural** | | | |
| `RUNS_ON` | workload placed on an asset | ✓ | 0.95 |
| `RUNS_AS` | workload executes as this identity | ✓ | 0.95 |
| `CONTAINS` | containment or composition | ✓ (downward) | 0.90 |
| `INVOKES` | identity→agent, or agent→tool | ✓ | 0.85 |
| `RETRIEVES_FROM` | RAG application to its retrieval source | ✓ | 0.75 |
| **Network** | | | |
| `ROUTES_TO` | configured reachability exists | ✓ | 0.70 |
| `CONNECTS_TO` | observed traffic occurred | ✓ | 0.60 |
| `EXPOSES` | makes reachable from a zone | ✓ | 0.80 |
| **AI** | | | |
| `PROMPTED_BY` | a prompt event attributed to an identity | ✓ | 0.50 |
| **Non-traversable** | | | |
| `OWNS` | administrative or business ownership | ✕ | — |
| `PROTECTS` | a control covering an entity | ✕ | — |
| `HAS_VULNERABILITY` | asset to CVE | ✕ | — |
| `STORES` | datastore to data class | ✕ | — |

## 8. Why four are non-traversable

```
  OWNS                 ownership is accountability, not access. An
                       owner who cannot log in has no path.

  PROTECTS             a control does not create reachability. It
                       REDUCES THE WEIGHT of adjacent traversable
                       edges. Modelling it as traversable produces
                       nonsense: "the attacker traverses the EDR."

  HAS_VULNERABILITY    a CVE is a property that makes an edge more
                       likely to be traversed, not a path in itself.

  STORES               data classification is a property of the
                       datastore, expressed as an edge only because
                       DATA_CLASS is shared across datastores.
```

## 9. Significant attributes — the fact-identity table

**This is the most consequential table in the document.** An attribute is *significant* if changing it means this is a **different assertion**, not an updated one.

```
  merge_key = hash(fact_type, subject.canonical_key, predicate,
                   object.canonical_key,
                   canonical(significant_attributes))
```

| Predicate | Significant attributes |
|---|---|
| `MEMBER_OF` | `mechanism` |
| `CAN_ASSUME` | `mechanism`, `conditions`, `privilege_level`, `granted_via` |
| `AUTHENTICATES_TO` | `mechanism` |
| `TRUSTS` | `mechanism`, `conditions`, `direction` |
| `CAN_READ` | `mechanism`, `conditions`, `resource_pattern` |
| `CAN_WRITE` | `mechanism`, `conditions`, `resource_pattern` |
| `CAN_EXECUTE` | `mechanism`, `conditions` |
| `CAN_MODIFY` | `mechanism`, `conditions`, `resource_pattern` |
| `CAN_DEPLOY` | `mechanism`, `conditions`, `target_environment` |
| `CAN_ACCESS` | `mechanism` |
| `RUNS_ON` | — |
| `RUNS_AS` | — |
| `CONTAINS` | `containment_type` |
| `INVOKES` | `invocation_type` |
| `RETRIEVES_FROM` | `permission_model` |
| `ROUTES_TO` | `port`, `protocol`, `action`, `conditions` |
| `CONNECTS_TO` | `port`, `protocol` |
| `EXPOSES` | `port`, `protocol`, `zone` |
| `PROMPTED_BY` | — |
| `OWNS` | `ownership_type` |
| `PROTECTS` | `control_type` |
| `HAS_VULNERABILITY` | `cve_id` |
| `STORES` | `data_class` |

### 9.1 Never significant, for any predicate

```
  last_seen · first_seen · observation_count · confidence
  evidence_ref · sources · collection metadata · edge_node_id
```

### 9.2 The two failure modes

```
  TOO MANY SIGNIFICANT   fact explosion. Every observation creates a
                         "new" fact, the 200,000:1 reduction never
                         happens, and the outbound queue fills.

  TOO FEW SIGNIFICANT    distinct realities collapse. A conditional
                         edge and an unconditional edge become
                         indistinguishable. The graph lies.
```

The worked example: `mechanism` is significant on `CAN_ASSUME` because a direct role assumption and an OIDC federation to the same role are **genuinely two different edges**. Collapse them and an over-broad OIDC trust hides inside a fact that already existed (`engines/10-fact-builder.md §8.3`).

## 10. Universal edge attributes

```jsonc
{
  "from": "email:priya.s@meridian.com",
  "predicate": "CAN_ASSUME",
  "to": "arn:aws:iam::123456789012:role/GHADeployRole",
  "attributes": {
    "mechanism": "oidc_federation",
    "conditions": [ { "key": "...", "op": "...",
                      "satisfiability": "CONDITIONAL" } ],
    "privilege_level": "ELEVATED",
    "granted_via": ["identity_policy:pol-4f2a"],
    "capped_by": ["boundary:DevBoundary"],
    "synthesized": true,
    "primitive_id": "aws.oidc.subject_condition_too_broad",
    "primitive_version": 2
  },
  "weight": 0.10,
  "confidence": 0.95,
  "first_seen": "...", "last_seen": "...", "removed_at": null,
  "evidence_ref": "sha256:8a1f...c4d2",
  "edge_node_id": "EDGE-DC1"
}
```

### 10.1 Condition satisfiability — closed enum

```
  ALWAYS          no condition, or always true in this environment
  CONDITIONAL     satisfiable from a position an attacker may hold
  HARD            requires something unlikely to be obtained
  UNSATISFIABLE   references a nonexistent principal or tag
                  → THE EDGE IS NOT CREATED
```

### 10.2 Privilege level — closed enum

```
  NONE · READ · WRITE · PRIVILEGED · ADMIN
```

---

# PART IV — CANONICAL KEYS

## 11. The rule

Every entity type has an **ordered** list of identity attributes, most authoritative first. The canonical key derives from the highest-confidence attribute available, in the form `scheme:value`.

```
  ⚠ THIS LIST IS SHIPPED CONFIGURATION, VERSIONED, AND IDENTICAL ON
    EVERY EDGE NODE IN A DEPLOYMENT.

    If DC1 prefers email and CLD prefers SAM name, they produce
    different canonical keys for the same person, therefore different
    tokens, therefore two disconnected subgraphs — with no error
    anywhere.
```

## 12. Priority lists

```
IDENTITY / human_user
  1  email:<verified corporate email>            confidence 1.00
  2  idp:<provider>:<immutable object id>        confidence 1.00
  3  adguid:<objectGUID>                         confidence 0.98
  4  upn:<userPrincipalName>                     confidence 0.95
  5  sam:<domain>\<sAMAccountName>               confidence 0.90

IDENTITY / service_account · workload_identity
  1  arn:<full ARN>  |  gcpsa:<email>  |  entrasp:<objectId>
  2  k8ssa:<cluster>/<namespace>/<name>
  3  svcname:<provider>:<account>:<name>

IDENTITY / agent_identity
  1  agentplat:<platform>:<agent id>             registered agents
  2  svcapp:<runs_as canonical key>/<app name>
  3  proc:<framework>@<asset canonical key>      self-hosted

IDENTITY / machine_account
  1  adguid:<objectGUID>
  2  sam:<domain>\<name>$

IDENTITY / database_user
  1  dbuser:<datastore canonical key>:<username>

ASSET / host · vm · endpoint
  1  cloudid:<provider>:<instance id>            confidence 1.00
  2  hwuuid:<SMBIOS UUID>                        confidence 0.98
  3  fqdn:<fully qualified domain name>          confidence 0.92
  4  mac:<sorted mac address set>                confidence 0.85
  5  ip:<address>@<date>                         confidence 0.60
                                                 ← TIME-BOUNDED, weakest

ASSET / container
  1  k8s:<cluster>/<namespace>/<pod>/<container>

ASSET / network_device
  1  serial:<vendor>:<serial number>
  2  mgmtip:<address>
  3  fqdn:<name>

ROLE
  1  arn:<ARN>  |  azrole:<definition id>@<scope>  |  gcprole:<name>
  2  k8srole:<cluster>/<namespace|cluster-scoped>/<name>
  3  dbrole:<datastore canonical key>:<name>

GROUP
  1  adguid:<objectGUID>  |  entragroup:<objectId>  |  oktagroup:<id>
  2  groupname:<provider>:<scope>:<name>

DATASTORE
  1  cloudid:<provider>:<resource id or ARN>
  2  conn:<engine>://<host>:<port>/<database>
  3  share:<host>/<share path>
  4  url:<normalised site or drive URL>

REPOSITORY
  1  repo:<provider>:<org>/<name>
PIPELINE
  1  pipeline:<provider>:<repo or project>/<name>

NETWORK
  1  cidr:<provider>:<account>:<cidr>
  2  cidr:onprem:<cidr>

AI_APPLICATION
  1  aiapp:<vendor>:<application id>
  2  aiapp:domain:<primary domain>
AI_MODEL
  1  aimodel:<provider>:<model id>[:<version>]
MCP_SERVER
  1  mcp:<server name>@<asset canonical key>     unregistered (agent)
  2  mcpreg:<registry>:<server id>               registered
MCP_TOOL
  1  mcptool:<mcp server canonical key>:<tool name>

SECRET
  1  secretfp:<fingerprint hash>                 correlates across locations
CREDENTIAL
  1  cred:<provider>:<credential id>

ACCOUNT
  1  account:<provider>:<account id>
BUSINESS_SERVICE
  1  bizsvc:<cmdb source>:<sys id>
```

## 13. Normalization — applied before key derivation

```
  ALL KEYS         lowercase the scheme; trim; collapse internal
                   whitespace; NFC-normalise unicode
  email            lowercase entirely
  fqdn / dns       lowercase; strip trailing dot
  sam / upn        lowercase both parts; canonical domain form
  DN               lowercase; normalise attribute spacing
  ARN              preserve case in resource segment ONLY where the
                   provider is case-sensitive; lowercase service and
                   region
  GUID / UUID      lowercase; strip braces
  MAC              lowercase; colon-separated; sorted if a set
  CIDR             canonical network address; explicit prefix length
  paths            provider-native separator; resolve `.` and `..`;
                   preserve case on case-sensitive filesystems
```

⚠ **The lowercase rule on email is load-bearing.** AD returns `Priya.S@Meridian.com`; Entra returns `priya.s@meridian.com`. Without normalization these are two people, and every cross-domain path breaks (`engines/04-normalizer-and-enrichment.md §8`).

## 14. Resolution confidence bands

```
  1.00        deterministic match on a priority-1 or 2 key
  0.90-0.99   deterministic match on a lower-priority key
  0.85-0.89   probabilistic, above the merge threshold
  0.65-0.84   QUARANTINED for human review — never auto-merged
  < 0.65      not merged
```

---

# PART V — CAPABILITY VOCABULARY

## 15. The twelve capabilities

Provider action strings map to capabilities through action groups. Analysts reason about capabilities; the engine retains specific actions only where needed.

```
  data.read          data.write         data.delete
  compute.execute    compute.deploy     compute.modify
  identity.assume    identity.modify    credential.read
  config.modify      network.modify     policy.modify
```

## 16. The mapping shape

```yaml
action_group:
  id: data.read
  capability: CAN_READ
  actions:
    aws:   [s3:GetObject, s3:ListBucket, dynamodb:GetItem,
            dynamodb:Query, rds-data:ExecuteStatement, kms:Decrypt,
            secretsmanager:GetSecretValue, ...]
    azure: [Microsoft.Storage/.../blobs/read, ...]
    gcp:   [storage.objects.get, bigquery.tables.getData, ...]
  retain_specific_action: false
```

```
  RETAIN SPECIFIC ACTION only where it is needed downstream:
    escalation primitives   the pattern is the specific action
    rightsizing output      the customer needs the exact permission
    evidence                defending a finding

  Everything else compresses to the group. 18,000 AWS actions
  become ~40 groups become 12 capabilities.       ≈ 400:1
```

## 17. Two mappings that are not obvious

```
  kms:Decrypt → CAN_READ
    Encryption is only a control if the principal lacks decrypt.
    A graph that models bucket access but ignores KMS reports
    false negatives on encrypted data and false positives on
    "protected by encryption."

  secretsmanager:GetSecretValue → CAN_READ *and* an IDENTITY edge
    Reading a credential means becoming whatever it authenticates
    as. The capability alone understates it.
```

## 18. Escalation ingredients

Some actions map to **no capability**. They are ingredients that only matter in combination, and must be retained at action granularity:

```
  iam:PassRole · lambda:CreateFunction · ec2:RunInstances ·
  iam:CreatePolicyVersion · iam:AttachUserPolicy ·
  Microsoft.Authorization/roleAssignments/write ·
  iam.serviceAccounts.actAs · <all K8s escalation verbs>
```

---

# PART VI — OBSERVATION SCHEMA

## 19. The contract between connectors and engines

An **Observation** is one sighting, from one source, at one moment, about resolved entities. It never leaves the appliance and it carries **plaintext canonical keys** — resolution, derivation and fact-building all require them.

```jsonc
{
  "schema": "overlook.observation.v1",
  "observed_at": "2026-08-14T04:00:11Z",

  "source": {
    "connector": "aws",
    "connector_version": "2.1.0",
    "collector": "iam.roles",
    "instance": "account:123456789012",
    "run_id": "run_01JC8Q2K7M4N5P6R7S8T9V0W1X",
    "edge_node_id": "EDGE-CLD"
  },

  "observation_type": "RELATIONSHIP",   // ENTITY | RELATIONSHIP |
                                        // PROPERTY | FINDING | EVENT

  "subject": {
    "canonical_key": "email:priya.s@meridian.com",
    "type": "IDENTITY",
    "subtype": "human_user",
    "resolution_confidence": 1.0,
    "resolution_method": "deterministic"
  },

  "predicate": "CAN_ASSUME",

  "object": {
    "canonical_key": "arn:aws:iam::123456789012:role/GHADeployRole",
    "type": "ROLE",
    "subtype": "assumable_role",
    "resolution_confidence": 1.0,
    "resolution_method": "deterministic"
  },

  "attributes": { },

  "evidence_ref": "sha256:8a1f...c4d2",
  "source_confidence": 0.99
}
```

## 20. Rules

```
  1  Observations are IMMUTABLE. Never updated, only superseded.
  2  Every observation carries resolution confidence AND method.
     Downstream confidence can never exceed resolution confidence.
  3  An observation with an unresolved subject or object is
     QUARANTINED, not emitted with a placeholder.
  4  Derived observations (from E7/E8/E9) use the same schema, with
     source.collector naming the engine and its content version.
  5  Evidence must exist BEFORE any observation references it.
```

---

# PART VII — SECURITY FACT SCHEMA

## 21. The five types

```
  ENTITY          this thing exists, with these properties
  RELATIONSHIP    this entity relates to that entity, this way
  PROPERTY        this attribute holds for this entity
  FINDING         this condition is worth attention
  EVENT_SUMMARY   this behaviour occurred N times in this window
```

## 22. The wire format

```jsonc
{
  "schema": "overlook.fact.v1",
  "fact_id": "01JC8Q2K7M4N5P6R7S8T9V0W1X",   // ULID, transport dedup only
  "fact_type": "RELATIONSHIP",
  "edge_node_id": "EDGE-CLD",
  // NO tenant_id — one appliance serves one customer (09 §2.1).
  // The mTLS client certificate establishes which deployment
  // a batch came from; deployment_id lives in the batch header.

  "subject":   { "type": "IDENTITY", "token": "IDN-9f3a7c21e845b0d6" },
  "predicate": "CAN_ASSUME",
  "object":    { "type": "ROLE",     "token": "ROL-2b8e4f19a70c5d33" },

  "attributes": {
    "mechanism": "oidc_federation",
    "conditions": ["subject_condition_broad"],
    "privilege_level": "ELEVATED",
    "synthesized": true,
    "primitive_id": "aws.oidc.subject_condition_too_broad",
    "primitive_version": 2
  },

  "confidence": 0.95,
  "sources": [
    { "id": "aws.iam", "confidence": 0.99,
      "last_seen": "2026-08-14T04:00:11Z", "agrees": true }
  ],
  "disagreement": false,

  "first_seen": "2026-05-22T09:14:22Z",
  "last_seen":  "2026-08-14T04:00:11Z",
  "observation_count": 8455,
  "valid_until": "2026-08-21T04:00:11Z",
  "removed_at": null,

  "evidence": {
    "ref": "sha256:8a1f...c4d2",
    "kind": "iam_trust_policy",
    "retention_expires": "2026-11-12T00:00:00Z"
  },

  "content_hash": "sha256:e91b...77af"
}
```

## 23. Type-specific requirements

```
  ENTITY          subject required · predicate and object MUST be absent
                  attributes carry the node properties
  RELATIONSHIP    subject, predicate, object all required
  PROPERTY        subject required · predicate MUST be "HAS_PROPERTY"
                  · object absent · attributes carry key/value
  FINDING         subject required · attributes MUST carry rule_id,
                  rule_version, severity, evidence_refs[]
  EVENT_SUMMARY   subject required · attributes MUST carry window_start,
                  window_end, event_type, count_bucket
                  ⚠ NEVER individual events
```

## 24. Semantic identity and idempotency

```
  fact_id            per-emission ULID. Transport dedup ONLY.
                     Never used for merging.

  semantic identity  merge_key from §9.
                     The console upserts on this, taking
                     max(last_seen), min(first_seen), and the
                     highest-confidence source.

  Replay MUST be idempotent. After a partition the appliance
  resends; without upsert-on-semantic-identity, every outage
  produces duplicate edges and every path count becomes untrustworthy.
```

## 25. Emission policy

```
  NEW fact                       → emit immediately
  significant attribute changed  → emit immediately
  confidence changed > 0.05      → emit
  disagreement flag flipped      → emit
  fact retracted                 → emit immediately
  ONLY last_seen / count changed → DO NOT emit.
                                   Heartbeat, default 24h.

  ⚠ heartbeat MUST be shorter than valid_until, or facts age out on
    the console and reappear, producing permanent churn.
```

## 26. Retraction

```
  A fact may be retracted ONLY when a COVERAGE WINDOW proves a source
  that WOULD have reported it ran to completion and did not.

  No complete window → retract nothing → mark the subgraph STALE.

  Stated in E12, E13 and here. Violating it in any one of the three
  produces the same outcome: findings silently resolve and the
  exposure score improves while nothing changed.
```

---

# PART VIII — CONNECTOR MANIFEST SCHEMA

## 27. The manifest

Declarative and **executable** — it drives scheduling, rate governance, health, coverage, the least-privilege policy shown to the customer, and the Controller UI.

```yaml
schema: overlook.manifest.v1

connector:
  id: aws
  version: 2.1.0
  vendor: Amazon Web Services
  domain: cloud_infrastructure
  applies_to_pct: 78

  api_surface: hybrid              # configuration | log_stream | hybrid
  functions: [collect]             # respond is a SEPARATE manifest

  execution:
    placement: any                 # any | pinned:<edge_node_id>
    requires_reachability: [aws_api_endpoints]

  auth:
    methods:
      - id: irsa
        preferred: true
      - id: assume_role
      - id: access_key
        deprecated: true
    least_privilege_policy: policies/aws-reader.json

  rate_limits:
    domains: [tenant, provider, account, service, operation_class]
    default_ceiling_pct: 30        # of the customer's published quota
    strategy: token_bucket_with_backoff

  collectors:
    - id: iam.roles
      api_surface: configuration
      load_bearing: true
      produces:
        entities:
          - { type: ROLE, subtype: assumable_role,
              canonical_key_source: [arn] }
        relationships:
          - { predicate: CAN_ASSUME, from: IDENTITY, to: ROLE,
              confidence: 0.97 }
      schedule: { interval: 4h, delta_cursor: null }
      full_enumeration: true
      emits_coverage_window: true
      coverage_scope: [account, region]
      cost: "~3 calls per role"
      requires_scope: ["iam:ListRoles", "iam:GetRole",
                       "iam:GetRolePolicy"]
      degrades_gracefully: false
      ocsf_class: null             # entity/relationship — no OCSF class
      depends_on: []

    - id: cloudtrail
      api_surface: log_stream
      emits_coverage_window: false # a stream can NEVER prove completeness
      ocsf_class: 3002
      schedule: { interval: 15m, delta_cursor: event_time }

  health:
    success_criteria: "roles_enumerated >= 1 AND auth_errors == 0"
    silent_threshold_pct: 5        # of this collector's own baseline
    staleness_threshold: 18h
    parse_rate_baseline: null      # JSON source, not applicable
```

## 28. Manifest rules

```
  1  ONE PARSER PER SOURCE TYPE, strictly 1:1. Borrowed from
     Chronicle. Makes parser health measurable and ownership
     unambiguous.
  2  emits_coverage_window is FALSE by default. A collector must
     prove it enumerates completely to claim it.
  3  A collector with functions:[respond] cannot appear in a
     collect manifest. Response is always a separate manifest with
     separate credentials.
  4  depends_on is a HARD gate. The dependency gate defers only
     the named collector, not the whole connector.
  5  requires_scope drives both the least-privilege policy and
     graceful degradation when a scope is absent.
```

---

# PART IX — PRIVACY POLICY SCHEMA

## 29. Allow-list, never deny-list

```
  DENY-LIST    "remove these sensitive fields"
               FAILS OPEN. A connector author adds a field, it ships,
               nobody notices.

  ALLOW-LIST   "only these fields may leave"
               FAILS CLOSED. A new field is dropped until someone
               deliberately permits it.

  This is the single most important implementation detail of the
  privacy claim. Everything else is mechanism; this is posture.
```

## 30. The policy

```yaml
schema: overlook.privacy.v1
version: 7
edited_by: shiva
edited_at: 2026-08-02T11:04:00Z

tokenization:
  algorithm: HMAC-SHA256
  key_ref: kms://customer/overlook-deployment-key
  token_length_bytes: 16
  classes:
    tokenized: [username, email, hostname, internal_ip, arn,
                resource_name, group_name, repository_name,
                file_path, database_name, bucket_name]
    clear:     [cve_id, cloud_provider, region, external_domain,
                model_name, mcp_tool_name, rule_id, port, protocol]
    optional:  [department]        # customer chooses

bucketing:
  record_count:  [0, 1K, 10K, 100K, 1M, 10M, 100M, "100M+"]
  data_volume:   order_of_magnitude
  duration:      15m
  request_rate:  order_of_magnitude

allow_list:
  RELATIONSHIP: [subject, predicate, object, mechanism, conditions,
                 privilege_level, synthesized, primitive_id,
                 primitive_version, confidence, first_seen, last_seen,
                 observation_count, evidence_ref, sources,
                 disagreement, removed_at]
  ENTITY:       [subject, type, subtype, criticality, confidence,
                 first_seen, last_seen, sources, removed_at,
                 properties.allowed]
  PROPERTY:     [subject, property_key, property_value_bucketed,
                 confidence, first_seen, last_seen, evidence_ref]
  FINDING:      [subject, rule_id, rule_version, severity, confidence,
                 first_seen, last_seen, evidence_refs]
  EVENT_SUMMARY:[subject, event_type, window_start, window_end,
                 count_bucket, anomaly_score]

prompt_handling: metadata_only     # metadata_only | local_inspection |
                                   # full_capture

on_validation_failure: quarantine  # NEVER "send anyway"
on_key_unavailable: halt           # NEVER "send plaintext"
```

## 31. Non-negotiable behaviours

```
  ✕ no bypass path exists in code for "urgent" facts
  ✕ key unavailable → outbound HALTS. Facts accumulate locally.
    No degraded mode, no fallback, no "just this once."
  ✓ every transmitted fact is reproducible in the outbound
    inspection feed, with a before/after diff
  ✓ the policy is exportable as a human-readable document —
    a procurement and audit artifact generated from live
    configuration
```

---

# PART X — CONFORMANCE

## 32. What "done" means for this contract

Testable, not aspirational. This is the gate that unblocks parallel work.

```
  [ ] every entity type and subtype has at least one fixture
  [ ] every predicate has a fixture with all its significant
      attributes populated
  [ ] a fixture round-trips: observation → fact → wire → parse,
      byte-identical
  [ ] a validator REJECTS: unknown predicate, unknown entity type,
      missing required field, a field absent from the allow-list
  [ ] identical canonical key input produces an identical token on
      two independently configured Edge Nodes
  [ ] the same entity observed via two different priority-list
      attributes resolves to ONE canonical key
  [ ] a fact with only last_seen changed does NOT emit
  [ ] a fact with a changed significant attribute DOES emit as new
  [ ] 10,000 identical observations collapse to ONE fact with
      observation_count = 10000
  [ ] a manifest with emits_coverage_window on a log_stream
      collector FAILS validation
  [ ] a PROPERTY fact carrying a plaintext hostname is QUARANTINED
  [ ] contract version hash mismatch between two Edge Nodes ALARMS
```

## 33. Fixture set structure

```
  fixtures/
    entities/       one per type × subtype
    predicates/     one per predicate, significant attributes populated
    facts/          one per fact type, valid and invalid pairs
    manifests/      valid, and each rule-violation case
    privacy/        allow-list pass and fail cases
    canonical_keys/ per entity type, every priority level, plus the
                    normalization edge cases (case, unicode, braces,
                    trailing dots)
    tokens/         known key + known input → known token
```

---

# PART XI — RECONCILIATIONS

Where this document changes what earlier documents said. Each is a deliberate correction.

```
  1  VECTOR_DATABASE is no longer a node type.
     It is DATASTORE / vector_store.
     WHY: it inherits classification, exposure and access semantics
     from DATASTORE for free. A separate type duplicated all of it.
     AFFECTS: 01 §26, 02, 07-ai-platforms, engines/09

  2  ASSIGNED_TO is removed as a predicate.
     It is AUTHENTICATES_TO with mechanism: enrollment.
     WHY: fewer predicates, and mechanism already distinguishes
     enrollment (authoritative) from observed_logon (weak).
     AFFECTS: connectors/02 Scalefusion, connectors/10 agent

  3  CAN_INVOKE is removed.
     Both directions use INVOKES — identity INVOKES agent,
     agent INVOKES tool. Direction disambiguates.
     AFFECTS: 02 §29, engines/07

  4  RUNS_AS is added as a distinct predicate.
     RUNS_ON is workload→asset placement. RUNS_AS is
     workload→identity execution context. They were conflated.
     AFFECTS: 01 §26, connectors/01, connectors/06

  5  tenant_id is absent from all schemas.
     Already applied in 01, 04, 09. Restated here as normative.

  6  WORKLOAD is added as a node type, distinct from ASSET/container.
     WHY: a Deployment is a template; a pod is an instance. Merging
     them made equivalence-class suppression impossible.
     AFFECTS: connectors/01 Kubernetes

  7  ACCOUNT, ORG_UNIT and BUSINESS_SERVICE added as node types.
     WHY: scope hierarchy was implicit in properties, which made
     inheritance in Azure and GCP closure impossible to express.
     AFFECTS: 02 §4, connectors/01, connectors/08

  8  SECRET and CREDENTIAL are distinct types.
     SECRET is the discovered artifact (a token in a file).
     CREDENTIAL is the authentication mechanism it represents.
     WHY: one credential may be discovered as many secrets in many
     locations, and the fingerprint correlation depends on the split.
     AFFECTS: 02 §18, connectors/03
```

---

## Open questions

```
  Q1  applies_to_pct in the manifest — measured how? It drives
      coverage reporting and catalog ordering, and right now it is
      an estimate with no method behind it.

  Q2  DATA_CLASS as a node with STORES edges, versus a property
      array on DATASTORE. Node form allows cross-datastore
      classification queries; property form is simpler and cheaper.
      Currently specified as a node — worth challenging.

  Q3  Should confidence be a single scalar, or split into
      resolution_confidence and assertion_confidence throughout?
      Currently collapsed via min(). Splitting is more honest and
      more expensive.

  Q4  valid_until is per-fact. Should the staleness horizon instead
      be per-predicate, since a CAN_ASSUME edge ages differently
      from a CONNECTS_TO?
```

---

*This document is normative. Changes require a version bump and updated fixtures.*

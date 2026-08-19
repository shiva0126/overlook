# Overlook — IAM, Identity and Entitlement Deep Dive

**Version:** 0.1 (design draft)
**Date:** 2026-08-12
**Companion to:** `01-system-design.md`
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

## Why this document exists

The system design document treats IAM as a subproblem of the attack path engine — about one and a half pages out of forty. That is a serious misallocation.

In a real cloud exposure graph:

```
   Edge type distribution, typical enterprise tenant (8M nodes, 120M edges)

   IAM-derived edges          ~88M    73%
     CAN_ASSUME                31M
     CAN_READ / CAN_WRITE      42M
     MEMBER_OF                 11M
     CAN_EXECUTE                4M
   Network reachability       ~18M    15%
   Code / pipeline             ~7M     6%
   Data / classification       ~4M     3%
   AI / agent / MCP            ~2M     2%
   Runtime / process           ~1M     1%
```

**Roughly three quarters of the TrustGraph is IAM.** The attack path engine, the risk model, the blast radius calculation, and every flagship finding — including the AI Privilege Gap — are functions of the IAM subgraph. If the IAM layer is shallow, the product is shallow, regardless of how good everything else is.

This document specifies that layer.

| Part | Chapters | Covers |
|---|---|---|
| A — Foundations | 1–6 | The IAM data model, the three privilege states, per-cloud evaluation semantics, closure at scale, build-vs-borrow |
| B — Escalation | 7–12 | Escalation primitives as synthesized edges: AWS, Azure, GCP, Kubernetes, federation |
| C — Directories | 13–15 | Active Directory graph depth, Entra ID, standing vs eligible privilege |
| D — Dynamics | 16–19 | ABAC/conditional edges, non-human identity lifecycle, secrets as identity, CIEM |
| E — Output | 20–24 | Findings catalog, remediation, confidence, connector implications, what the layer must deliver |

---

# PART A — FOUNDATIONS

---

## 1. The IAM data model

### 1.1 Four things, not one

Most IAM tooling fails because it models permissions as a single flat concept. There are four distinct object classes, and conflating them is the root of most incorrect graphs.

```
  PRINCIPAL          the thing that acts
                     user, group, role, service account, workload identity,
                     managed identity, service principal, computer account

  GRANT              a statement that a principal may do something
                     identity policy, resource policy, role assignment,
                     ACE on a directory object, RoleBinding

  CONSTRAINT         a statement that caps or blocks what grants allow
                     SCP, permission boundary, session policy, deny assignment,
                     deny policy, org policy, Conditional Access

  CAPABILITY         the resulting ability, expressed in graph terms
                     "CAN_ASSUME role X", "CAN_READ datastore Y"
```

The pipeline is always: **Grants − Constraints → Capabilities → Graph edges.** A connector collects grants and constraints. The Edge Collector computes capabilities. Only capabilities become edges.

### 1.2 The principal taxonomy

```
  HUMAN
    workforce user           (IdP-managed, MFA-capable, phishable)
    external / guest user    (B2B, partner, contractor)
    break-glass account      (should be tiny in number, heavily monitored)

  NON-HUMAN (NHI)
    service account          (long-lived, key-based)
    workload identity        (IRSA, Managed Identity, GCP WIF, SPIFFE)
    application / SP         (OAuth app with its own permissions)
    machine account          (AD computer objects — often forgotten)
    CI/CD identity           (GitHub OIDC, GitLab JWT, Jenkins credential)
    agent identity           (an AI agent's runtime identity)  <- new class

  ROLE / ASSUMABLE
    assumable role           (AWS role, GCP SA impersonation target)
    directory role           (Entra role, AD group with rights)
    eligible role            (PIM — activatable but not active)
```

The **agent identity** row is the one nobody else models yet. An AI agent's `runs_as` identity behaves like a service account with one crucial difference: its instruction channel is untrusted natural language. In the graph it is an NHI; in the risk model it carries an additional exposure factor because it can be *induced* to act (Chapter 20.4).

### 1.3 The principal node

```jsonc
{
  "token": "IDN-9f3a7c21e845b0d6",
  "type": "IDENTITY",
  "subtype": "service_account",
  "properties": {
    "provider": "aws",
    "is_human": false,
    "created_at": "2024-11-02",
    "last_used": "2026-08-11T22:14:00Z",
    "dormant_days": 0,
    "mfa_enabled": null,             // n/a for NHI
    "credential_count": 2,
    "oldest_credential_age_days": 641,
    "owner": "IDN-team-platform",    // often null — that IS a finding
    "granted_permission_count": 340,
    "effective_permission_count": 210,
    "used_permission_count_90d": 12,
    "privilege_tier": "ADMIN",       // NONE|READ|WRITE|PRIVILEGED|ADMIN
    "reaches_crown_jewels": 3,
    "is_agent_identity": true,
    "escalation_paths_out": 4
  }
}
```

Three of those fields — `granted`, `effective`, `used` — are the subject of the next chapter and do not exist anywhere in the current system design.

---

## 2. Granted vs Effective vs Used

### 2.1 The three states

```
  GRANTED     What the attached policies say, read literally.
              Source: policy documents, role assignments.
              Use: compliance, "what did someone intend?"
              Trap: wildly overstates reality. Ignores SCPs and boundaries.

  EFFECTIVE   What the principal can ACTUALLY do right now, after
              evaluating every layer in the correct precedence order,
              with conditions resolved where resolvable.
              Source: computed. This is the expensive one.
              Use: THE GRAPH. Every edge must be effective, not granted.

  USED        What the principal has actually done in a window.
              Source: CloudTrail, Azure Activity Log, GCP Cloud Audit Logs,
                      Kubernetes audit, AD event logs.
              Use: rightsizing, dormancy, anomaly baselines.
```

### 2.2 Why the distinction is product-critical

```
   Role: DevOpsAdmin

   GRANTED    AdministratorAccess         -> "*" on "*"       (2 lines of JSON)
   EFFECTIVE  after SCP DenyOutsideRegion,
              after permission boundary DevBoundary,
              after 3 explicit denies       -> 8,400 actions across 210 services
   USED       in the last 90 days          -> 12 actions, 3 services, 41 resources

   Three completely different answers. The graph must use EFFECTIVE.
   The remediation must use USED.
   Only the compliance report cares about GRANTED.
```

An exposure product that builds edges from *granted* permissions produces a graph where almost every principal reaches almost everything — technically defensible, operationally useless. An exposure product that never computes *used* can identify the problem but never propose a safe fix, because it cannot tell the customer which of the 8,400 actions are load-bearing.

**Overlook must compute all three.** Effective drives the graph. Used drives remediation. Granted drives compliance evidence.

### 2.3 The rightsizing output

This is the most immediately actionable artifact IAM analysis produces, and it costs little once the three states exist:

```
  RIGHTSIZE  svc-devops-ai

  Effective permissions   8,400 actions / 210 services
  Used (90 days)             12 actions /   3 services
  Unused but reachable     8,388 actions
  Of the unused:
      - 41 are escalation primitives   <- these matter most
      - 6 reach crown jewels
      - 1,204 are destructive (Delete*, Terminate*, Put*Policy)

  PROPOSED POLICY  (generated, ready to apply)
     s3:GetObject, s3:PutObject on arn:aws:s3:::deploy-artifacts/*
     ecs:UpdateService on arn:aws:ecs:*:*:service/prod-api
     ... 10 more

  PREDICTED GRAPH EFFECT
     escalation paths out:  4 -> 0
     crown jewels reachable: 3 -> 0
     attack paths eliminated: 1,240
     AI Privilege Gap finding: RESOLVED

  RISK OF APPLYING
     3 actions used 61-90 days ago are not in the proposed policy
     (quarterly jobs?) — flagged for human review before apply
```

That last block is what separates a usable recommendation from a dangerous one. A 90-day window misses quarterly processes. Always surface the edge cases rather than silently dropping them.

---

## 3. Policy evaluation: AWS

### 3.1 The precedence chain

Getting this order wrong produces a graph that is confidently, systematically wrong. AWS evaluates in this order, and any **explicit deny anywhere terminates immediately**:

```
   1. Explicit DENY in ANY policy type          -> DENY. Stop.
   2. Service Control Policy (SCP)              -> if not allowed, DENY
   3. Resource-based policy                     -> if allowed, may ALLOW outright
                                                   (cross-account: BOTH sides needed,
                                                    EXCEPT for a few services where
                                                    the resource policy alone suffices)
   4. Permission boundary                       -> if not allowed, DENY
   5. Session policy (assumed-role sessions)    -> if not allowed, DENY
   6. Identity-based policy                     -> if allowed, ALLOW
   7. Otherwise                                 -> implicit DENY
```

Subtleties that must be encoded:

- **Cross-account requires both sides**, except where a resource policy alone grants access (S3, KMS, Lambda, SNS, SQS, Secrets Manager and others behave differently — this is a per-service lookup table, not a rule).
- **Permission boundaries do not grant.** A boundary allowing `*` grants nothing; it only fails to restrict.
- **SCPs do not grant either**, and they do not apply to the management account, and they do not apply to service-linked roles.
- **Resource Control Policies (RCPs)** are the newer org-level constraint on resource policies. Same "cap, don't grant" semantics.
- **`NotAction` / `NotResource`** invert set semantics and are a common source of policy-analysis bugs. Handle explicitly; do not treat as sugar.

### 3.2 Representation in the Edge Collector

Store evaluation as a **capability set**, not as parsed policy:

```jsonc
{
  "principal": "IDN-svc-devops-ai",
  "capabilities": [
    {
      "action_pattern": "s3:Get*",
      "resource_pattern": "arn:aws:s3:::prod-payments-*/*",
      "effect": "ALLOW",
      "granted_via": ["identity_policy:pol-4f2a", "resource_policy:bkt-9c1e"],
      "capped_by": ["boundary:DevBoundary"],
      "conditions": [
        {"key":"aws:SourceVpce","op":"StringEquals","values":["vpce-0abc"],
         "satisfiability":"CONDITIONAL"}
      ],
      "unconditional": false
    }
  ]
}
```

`satisfiability` is the field that makes conditions usable downstream:

| Value | Meaning | Effect on edge weight |
|---|---|---|
| `ALWAYS` | No condition, or a condition always true in this environment | full weight |
| `CONDITIONAL` | Satisfiable by an attacker who already has some position (e.g. `SourceVpce` if they're inside the VPC) | reduced weight |
| `HARD` | Requires something an attacker is unlikely to obtain (`aws:MultiFactorAuthPresent`, hardware-bound) | heavily reduced |
| `UNSATISFIABLE` | Cannot be met (references a nonexistent principal/tag) | edge not created |

Resolving satisfiability requires environmental context — do we know the attacker's likely position? — so it is computed at the Edge Collector where that context exists, and shipped as a classification rather than as a raw condition block.

### 3.3 Action-to-capability mapping

A permission string is not a capability. `s3:GetObject` is `CAN_READ`. `iam:PassRole` is not `CAN_*` anything on its own — it is an **escalation ingredient** (Part B).

The mapping table is content, versioned and updated with the cloud providers:

```
  s3:GetObject, s3:ListBucket           -> CAN_READ        (data)
  s3:PutObject, s3:DeleteObject         -> CAN_WRITE       (data)
  sts:AssumeRole                        -> CAN_ASSUME      (identity)
  ssm:SendCommand, ssm:StartSession     -> CAN_EXECUTE     (compute)
  ec2:RunInstances                      -> CAN_DEPLOY      (compute)
  kms:Decrypt                           -> CAN_READ        (via encryption)
  secretsmanager:GetSecretValue         -> CAN_READ        (credential!)  <-
  iam:PassRole                          -> ESCALATION_INGREDIENT
  lambda:UpdateFunctionCode             -> CAN_EXECUTE as the function's role
```

Two of those deserve emphasis:

- **`kms:Decrypt` is a data-access edge.** Encryption is only a control if the principal lacks the decrypt permission. A graph that models S3 access but ignores KMS will report false negatives on encrypted data and false positives on "protected by encryption."
- **`secretsmanager:GetSecretValue` is an identity edge.** Reading a secret that contains a credential means becoming whatever that credential authenticates as. See Chapter 18.

---

## 4. Policy evaluation: Azure and GCP

### 4.1 Azure — two planes, one graph

Azure's defining complication is that **Azure RBAC and Entra ID roles are separate systems** and both must be in the graph, plus the bridge between them.

```
   ENTRA ID (directory plane)          AZURE RBAC (resource plane)
   ------------------------------      ---------------------------
   Global Administrator                Owner
   Privileged Role Administrator       Contributor
   Application Administrator           User Access Administrator
   Cloud Application Administrator     Reader
   Groups Administrator                custom roles
   User Administrator
   Directory role assignments          Role assignments at:
   Graph API app permissions             management group
   OAuth2 consent grants                 subscription
                                         resource group
                                         resource

   THE BRIDGE (the part that is always missed):
     Global Administrator can elevate itself to
     "User Access Administrator at the root management group scope"
     -> one directory role becomes Owner of EVERY subscription.

   This must be an edge. It is a single API toggle
   (elevateAccess) and it is how directory compromise becomes
   total cloud compromise.
```

Azure evaluation order:

```
   1. Deny assignments        -> DENY, wins over everything (incl. Owner)
   2. Azure Policy (deny)     -> blocks resource operations
   3. Role assignments        -> union of all applicable, inherited down
                                 the management group -> subscription ->
                                 resource group -> resource hierarchy
   4. ABAC conditions on role assignments
   5. Otherwise               -> DENY
```

Unlike AWS, Azure role assignments are **additive with inheritance** — a Reader at management-group scope is a Reader on every resource beneath. Inheritance must be materialized carefully or the closure explodes; see Chapter 5.

### 4.2 GCP — hierarchy and impersonation

```
   Organization -> Folder -> Project -> Resource
     IAM policies at each level INHERIT DOWNWARD and are ADDITIVE.
     A grant at the org level applies to every resource under it.

   Evaluation:
     1. Deny policies (IAM Deny)     -> DENY, evaluated first
     2. Union of allow policies across the hierarchy
     3. Conditions (CEL expressions) on bindings
     4. Otherwise -> DENY

   The GCP-specific hazard: SERVICE ACCOUNT IMPERSONATION
     iam.serviceAccounts.getAccessToken   -> become the SA, right now
     iam.serviceAccounts.actAs            -> attach the SA to a resource
     iam.serviceAccounts.signJwt          -> mint tokens as the SA
     iam.serviceAccountKeys.create        -> permanent credential for the SA

   Each of these is a CAN_ASSUME edge. GCP's impersonation model
   makes SA-to-SA chains common and deep — 4-5 hop chains are normal
   in a mature GCP estate. Chain depth must be bounded (Ch. 5.4).
```

### 4.3 The comparative table

| | AWS | Azure | GCP |
|---|---|---|---|
| Grant model | Policy documents | Role assignments | Policy bindings |
| Inheritance | None (explicit only) | Downward, additive | Downward, additive |
| Deny | Explicit deny wins | Deny assignments win | IAM Deny wins |
| Org constraint | SCP, RCP | Azure Policy, mgmt-group roles | Org policies, IAM Deny |
| Boundary concept | Permission boundary | — | — |
| Conditions | Condition keys | ABAC conditions | CEL expressions |
| Identity bridge | Federation / OIDC | Entra ↔ RBAC (elevateAccess) | Workload Identity Federation |
| Impersonation | sts:AssumeRole | Managed Identity | SA impersonation (4 ways) |
| Hardest part | Policy precedence + cross-account | Two planes + the bridge | Hierarchy + impersonation chains |

---

## 5. Closure at scale

### 5.1 The combinatorial problem

Naïve materialization is impossible:

```
   400 accounts
   × 12,000 principals per account (incl. roles)
   × 18,000 distinct actions across AWS services
   × resources (unbounded, often wildcard patterns)

   = billions to trillions of (principal, action, resource) triples
```

You cannot store this. You cannot compute it eagerly. Four techniques, applied together, bring it into range.

### 5.2 Technique 1 — Action grouping

Analysts do not care about 18,000 actions. They care about ~12 capabilities. Collapse at ingestion:

```
   18,000 actions -> 40 action groups -> 12 graph capabilities

   e.g. ACTION_GROUP "data.read" = {
          s3:GetObject, s3:ListBucket, dynamodb:GetItem, dynamodb:Query,
          rds-data:ExecuteStatement, kms:Decrypt, secretsmanager:GetSecretValue, ...
        }  -> capability CAN_READ
```

Retain the specific actions **only** for (a) escalation primitives, (b) the rightsizing output, and (c) evidence. Everything else compresses to a group. This alone is a 400:1 reduction.

### 5.3 Technique 2 — Resource pattern preservation

Do not expand `arn:aws:s3:::prod-*` into 4,000 bucket edges. Keep the pattern, and expand lazily only when a specific resource is queried or when the resource is a crown jewel.

```
   Stored:   IDN-x CAN_READ pattern("arn:aws:s3:::prod-*")
   Expanded: only for the 47 buckets marked crown jewel or PII-classified
```

This inverts the usual approach and is the single biggest saving, because wildcard grants are exactly the ones that would explode.

### 5.4 Technique 3 — Assume-chain closure with bounds

`CAN_ASSUME` is transitive and is the densest predicate. Materialize its closure incrementally, with limits:

```
   max_chain_depth = 6
   prune if cumulative_confidence < 0.4
   collapse cycles (A->B->A is A<->B, depth 1)
   equivalence-class roles with identical policy hashes
       (400 identical per-account roles -> 1 class with count 400)
```

Policy-hash equivalence classes are underrated: large orgs deploy the same role via Terraform to hundreds of accounts. Treating them as one class with a multiplicity is both faster and produces a far better UI.

### 5.5 Technique 4 — Lazy expansion at query time

```
   PRECOMPUTED (always current, incrementally maintained):
     - CAN_ASSUME transitive closure
     - MEMBER_OF transitive closure
     - escalation edges (Part B)  <- always precompute; they're rare and critical
     - capabilities to crown jewels only

   COMPUTED ON DEMAND:
     - full capability set for one principal ("what can Priya do?")
     - blast radius from an arbitrary node
     - reachability to non-crown-jewel resources
```

### 5.6 Incremental recomputation

```
   IAM change detected (policy modified, assignment added)
        |
        v
   Determine affected principal set
        - direct: the principal named
        - transitive: anyone who CAN_ASSUME them  (reverse closure lookup)
        - group members if a group policy changed
        |
        v
   Recompute capabilities for that set only
        typical: 1-200 principals, 100-800ms
        worst case (org-level SCP change): full recompute, queued
        |
        v
   Diff capabilities -> emit changed edges only
        |
        v
   Path engine invalidates paths through changed edges
```

**Realistic budget:** a 400-account AWS estate should complete a full closure in under 30 minutes on the Edge Collector Edge L, and an incremental update in under one second. If the design cannot hit those, the product cannot be near-real-time, and near-real-time is what makes the change feed valuable.

---

## 6. Build vs borrow

### 6.1 The trap

Reimplementing AWS's policy evaluation engine from scratch is a multi-year project with a long tail of subtle bugs, each of which produces a wrong graph. Several companies have sunk enormous effort here.

### 6.2 What the providers give you free

| Capability | AWS | Azure | GCP |
|---|---|---|---|
| Effective permission check | IAM Policy Simulator (`SimulatePrincipalPolicy`) | `checkAccess` API | `testIamPermissions` |
| External access analysis | Access Analyzer (external + unused) | Defender for Cloud | Policy Analyzer |
| Unused permissions | Access Analyzer unused-access findings | Entra Permissions Management | Recommender (role rightsizing) |
| Policy validation | Access Analyzer policy validation | Azure Policy compliance | Policy Troubleshooter |
| Last-used data | IAM `GetServiceLastAccessedDetails` | Entra sign-in logs | Recommender / audit logs |

### 6.3 The recommended hybrid

```
   STATIC ANALYSIS (build)          -- fast, complete, offline
     parse all policies
     compute candidate capabilities
     apply precedence rules
     -> produces the graph, with confidence 0.85-0.95

   PROVIDER VERIFICATION (borrow)   -- authoritative, rate-limited, costly
     for HIGH-VALUE edges only:
       - any edge reaching a crown jewel
       - any escalation primitive
       - any cross-account edge
     call SimulatePrincipalPolicy / checkAccess / testIamPermissions
     -> confirms or refutes, raises confidence to 0.99 or removes the edge

   FREE GROUND TRUTH (borrow)       -- no cost, high value
     ingest Access Analyzer findings directly as facts
     ingest unused-access findings for rightsizing
     ingest last-accessed data for the USED state
```

Verifying only high-value edges is the key design decision. Simulating every edge is impossible (rate limits, cost); simulating none leaves you unable to defend a finding when a customer's cloud team says "that's wrong." Verify the ones you will show to a human.

**Confidence should be visible and sourced:**

```
   confidence 0.99   verified via provider simulation API
   confidence 0.95   static analysis, unconditional, single-account
   confidence 0.85   static analysis with resolvable conditions
   confidence 0.70   static analysis, cross-account, resource policy inferred
   confidence 0.55   inferred from observed activity in audit logs only
```

---

# PART B — ESCALATION PRIMITIVES

---

## 7. The concept

### 7.1 Why this is the most important chapter

No policy document says *"this principal can become an administrator."* It says `iam:CreatePolicyVersion`, and a human who knows AWS understands that this is game over.

**Escalation primitives are the encoded form of that knowledge.** They are patterns that, when matched against a principal's effective capabilities, **synthesize a graph edge that exists in no policy**.

```
   Without escalation primitives:
     Priya -> CAN_READ -> one S3 bucket.  Path length 1. Boring.

   With escalation primitives:
     Priya -> CAN_ASSUME(synthesized) -> DevOpsAdmin -> CAN_READ -> prod DB
     because Priya holds iam:PassRole + lambda:CreateFunction.
     Path length 3. Critical.
```

Every serious attack path in cloud runs through at least one of these. A path engine without them finds only what someone deliberately granted — which is, definitionally, the permissions nobody is worried about.

### 7.2 The primitive schema

Primitives are **content**, not code — shipped and updated through the content pipeline (system design §17), independently versioned, and testable.

```yaml
primitive:
  id: aws.privesc.passrole_lambda
  version: 3
  provider: aws
  severity: critical
  mitre: [T1548, T1078.004]

  # what the principal must be able to do
  requires:
    all_of:
      - action: "iam:PassRole"
        resource: "$TARGET_ROLE"
      - action: "lambda:CreateFunction"
      - any_of:
          - action: "lambda:InvokeFunction"
          - action: "lambda:CreateEventSourceMapping"

  # what must be true of the environment
  preconditions:
    - "$TARGET_ROLE trust policy permits lambda.amazonaws.com"
    - "no SCP denies lambda:CreateFunction in this account"

  # the edge to synthesize
  produces:
    predicate: CAN_ASSUME
    from: "$PRINCIPAL"
    to: "$TARGET_ROLE"
    weight: 0.85
    confidence: 0.92
    synthesized: true
    rationale: >
      Principal can create a Lambda function, pass $TARGET_ROLE to it,
      and invoke it — executing arbitrary code with that role's credentials.

  remediation:
    - "Scope iam:PassRole with a Condition on iam:PassedToService"
    - "Restrict the Resource of iam:PassRole to specific role ARNs"
    - "Apply a permission boundary limiting passable roles"

  verification:
    method: provider_simulation
    actions: ["iam:PassRole", "lambda:CreateFunction"]
```

### 7.3 Edges must be labelled as synthesized

```jsonc
{
  "predicate": "CAN_ASSUME",
  "attributes": {
    "synthesized": true,
    "primitive_id": "aws.privesc.passrole_lambda",
    "primitive_version": 3,
    "rationale": "iam:PassRole + lambda:CreateFunction + lambda:InvokeFunction"
  },
  "confidence": 0.92
}
```

Non-negotiable for three reasons: an analyst will not believe an edge they cannot explain; a cloud engineer will demand to know why Overlook says a read-only role is admin; and when a primitive turns out to be wrong, every edge it produced must be findable and retractable in one operation.

### 7.4 Preconditions matter more than permissions

`iam:PassRole` + `lambda:CreateFunction` only escalates if the target role's **trust policy actually permits Lambda to assume it**. Skipping precondition checks is how these engines generate false positives at scale — and one confident false "you have an admin escalation path" costs more trust than ten missed findings.

Every primitive must state its preconditions, and the engine must evaluate them against real collected state.

---

## 8. AWS escalation catalog

Organised by mechanism. This is the v1 target set; the catalog is living content.

### 8.1 Direct IAM manipulation — become admin in one call

```
  iam:CreatePolicyVersion + iam:SetDefaultPolicyVersion
      -> rewrite an attached policy to "*"                    CRITICAL
  iam:AttachUserPolicy / AttachGroupPolicy / AttachRolePolicy
      -> attach AdministratorAccess                           CRITICAL
  iam:PutUserPolicy / PutGroupPolicy / PutRolePolicy
      -> inline policy granting "*"                           CRITICAL
  iam:AddUserToGroup
      -> join an admin group                                  CRITICAL
  iam:UpdateAssumeRolePolicy (+ sts:AssumeRole)
      -> add self to a privileged role's trust policy         CRITICAL
  iam:CreateAccessKey (on another principal)
      -> permanent credential for that principal              CRITICAL
  iam:CreateLoginProfile / iam:UpdateLoginProfile
      -> set console password for another user                CRITICAL
  iam:PassRole + iam:CreateServiceLinkedRole variants
```

### 8.2 PassRole + compute — the classic family

All follow the shape *"pass a role to something that runs code."*

```
  iam:PassRole + ec2:RunInstances (+ instance profile)
  iam:PassRole + lambda:CreateFunction + lambda:InvokeFunction
  iam:PassRole + lambda:CreateFunction + lambda:CreateEventSourceMapping
  iam:PassRole + cloudformation:CreateStack
  iam:PassRole + glue:CreateDevEndpoint
  iam:PassRole + codebuild:CreateProject + codebuild:StartBuild
  iam:PassRole + sagemaker:CreateNotebookInstance + CreatePresignedNotebookUrl
  iam:PassRole + datapipeline:CreatePipeline + PutPipelineDefinition
  iam:PassRole + ecs:RegisterTaskDefinition + ecs:RunTask
  iam:PassRole + apigateway / states:CreateStateMachine
```

**The generalisation worth encoding:** `iam:PassRole` scoped to `Resource: "*"` combined with **any** compute-creation permission is an escalation to **every role in the account whose trust policy allows that service**. Model it that way rather than as N separate primitives — it is both more accurate and produces far fewer, better findings.

### 8.3 Modify existing compute

No PassRole needed; hijack something that already holds a role.

```
  lambda:UpdateFunctionCode              -> run as the function's role
  lambda:UpdateFunctionConfiguration     -> env manipulation / layers
  ssm:SendCommand / ssm:StartSession     -> run as the instance's role
  ec2:ModifyInstanceAttribute (userdata) -> code exec on next boot
  ecs:UpdateService with new task def    -> run as the task role
  eks:UpdateNodegroupConfig / K8s access -> Chapter 11
  batch, glue:UpdateJob, codebuild:UpdateProject
  cloudformation:UpdateStack             -> alter infra as the stack role
```

### 8.4 Credential and data access as escalation

```
  secretsmanager:GetSecretValue      -> whatever the secret authenticates
  ssm:GetParameter (SecureString)    -> same
  kms:Decrypt (+ ciphertext access)  -> decrypt stored credentials
  ec2:GetPasswordData (+ key)        -> Windows admin password
  rds:DownloadDBLogFilePortion       -> credentials in logs
  s3:GetObject on a state bucket     -> Terraform state contains secrets  <-
  ec2:DescribeInstanceAttribute      -> userdata often contains secrets
  ecr / codeartifact                 -> registry credentials
```

**Terraform state buckets deserve their own rule.** They routinely contain database passwords, API keys, and private keys in plaintext, and read access to them is granted casually.

### 8.5 Trust and organisational

```
  organizations:LeaveOrganization      -> escape SCP constraints entirely
  organizations:AttachPolicy / DetachPolicy
  sts:AssumeRole with a wildcard/overly-broad trust policy
  Trust policy with "Principal": "*"   -> ANY AWS account can assume  CRITICAL
  Trust policy with an account-only principal and no ExternalId
                                        -> confused deputy
  iam:CreateOpenIDConnectProvider / UpdateOpenIDConnectProviderThumbprint
  iam:CreateSAMLProvider / UpdateSAMLProvider  -> mint federated identities
  account:CloseAccount, iam:DeleteAccountPasswordPolicy
```

The wildcard-trust-policy case is worth calling out separately: it is a **start condition** for the path engine, not merely an edge — it means an attacker with *any* AWS account is one API call from being inside.

---

## 9. Azure escalation catalog

### 9.1 The directory-to-resource bridge

The highest-severity Azure pattern, and the one most often missing from competing products:

```
  Global Administrator
      -> toggles "Access management for Azure resources" (elevateAccess API)
      -> becomes User Access Administrator at the ROOT management group
      -> grants self Owner on EVERY subscription in the tenant

  Synthesized edges:
      IDN-globaladmin CAN_ASSUME ROL-owner@<every subscription>
      weight 0.95, confidence 0.98, primitive azure.privesc.elevate_access
```

Every Entra role that can *reach* Global Administrator therefore reaches every subscription. That transitive fact must be materialized, or Azure attack paths are systematically understated.

### 9.2 Entra ID directory escalation

```
  RoleManagement.ReadWrite.Directory (Graph app permission)
      -> assign self Global Administrator                     CRITICAL
  AppRoleAssignment.ReadWrite.All
      -> grant self any Graph permission                      CRITICAL
  Application.ReadWrite.All  /  Application Administrator
      -> add credentials to ANY service principal
      -> become that SP, inheriting its permissions           CRITICAL
  Cloud Application Administrator                             (same, minus on-prem)
  Privileged Authentication Administrator
      -> reset a Global Admin's credentials                   CRITICAL
  User Administrator / Helpdesk Administrator
      -> reset passwords for non-admin users
  Groups Administrator + a group with role assignment
      -> add self to a privileged group
  Dynamic group with an attacker-controllable attribute
      -> set your own department -> auto-join a privileged group  <- subtle
  Directory Synchronization Accounts (AAD Connect)
      -> on-prem to cloud pivot, extremely high value
  Partner / GDAP delegated admin relationships
      -> external tenant holds privilege in yours
```

The **dynamic group membership rule** case is a genuinely underappreciated one: if a user can edit an attribute referenced by a dynamic group's membership rule, and that group holds a role assignment, the user can grant themselves that role by editing their own profile. Detecting it requires parsing dynamic membership rules and cross-referencing user-writable attributes.

### 9.3 Azure resource-plane escalation

```
  Microsoft.Authorization/roleAssignments/write   (Owner, UAA)
      -> grant self anything                                  CRITICAL
  Microsoft.Compute/virtualMachines/runCommand/action
      -> execute as the VM's managed identity                 HIGH
  Microsoft.Compute/virtualMachines/extensions/write
      -> custom script extension = code exec                  HIGH
  Microsoft.Web/sites/config/list, /publishxml
      -> App Service credentials and managed identity
  Automation Account: RunAs / Hybrid Worker / Runbook write
  Logic Apps with a managed identity + workflow write
  Microsoft.KeyVault/vaults/accessPolicies/write
      -> grant self secret read -> credentials                CRITICAL
  Microsoft.ContainerService/managedClusters/listClusterAdminCredential
      -> AKS cluster-admin                                    CRITICAL
  Microsoft.Resources/deployments/write (+ template)
  Managed Identity assignment to a resource you control
```

---

## 10. GCP escalation catalog

```
  IMPERSONATION FAMILY  (the GCP signature)
    iam.serviceAccounts.getAccessToken      -> become the SA now     CRITICAL
    iam.serviceAccounts.getOpenIdToken      -> identity tokens as SA
    iam.serviceAccounts.signBlob            -> sign as the SA
    iam.serviceAccounts.signJwt             -> mint SA credentials
    iam.serviceAccountKeys.create           -> permanent SA key      CRITICAL
    iam.serviceAccounts.implicitDelegation  -> chain through another SA
    iam.serviceAccounts.actAs + <create compute>  -> run as the SA

  POLICY MANIPULATION
    resourcemanager.projects.setIamPolicy   -> grant self Owner      CRITICAL
    resourcemanager.folders.setIamPolicy
    resourcemanager.organizations.setIamPolicy                       CRITICAL
    iam.roles.update                        -> add perms to a custom role
                                               you already hold      CRITICAL

  COMPUTE / DEPLOY (all need actAs on the target SA)
    compute.instances.create                -> VM with the SA attached
    compute.instances.setMetadata           -> startup script code exec
    cloudfunctions.functions.create/update
    run.services.create/update
    cloudbuild.builds.create                -> runs as the Cloud Build SA,
                                               which is often Editor  <-
    deploymentmanager.deployments.create    -> runs as the DM SA
    composer.environments.create
    dataproc.clusters.create, dataflow, notebooks

  DATA -> CREDENTIAL
    secretmanager.versions.access
    storage.objects.get on a Terraform state bucket
    cloudkms.cryptoKeyVersions.useToDecrypt

  ORG POLICY
    orgpolicy.policy.set    -> disable constraints (e.g. SA key creation)
```

**`cloudbuild.builds.create` is the standout.** The default Cloud Build service account historically carries broad project Editor rights, so the ability to trigger a build is frequently equivalent to project takeover. It is granted routinely to CI users who have no idea.

---

## 11. Kubernetes RBAC

### 11.1 Why it belongs in the same graph

Kubernetes is where cloud IAM and workload identity meet, and it is the most common bridge between "compromised container" and "compromised cloud account."

```
   POD  --(mounts)--> SERVICE ACCOUNT TOKEN
        --(IRSA / Workload Identity / Managed Identity)-->
        CLOUD ROLE --> CLOUD RESOURCES

   And in the other direction:
   CLOUD IDENTITY --(eks:AccessEntry / aks admin creds / gke getCredentials)-->
        K8s CLUSTER-ADMIN --> every SA in the cluster --> every cloud role
        those SAs map to
```

That second direction is the one that surprises people: a cloud IAM permission (`container.clusters.getCredentials`, `listClusterAdminCredential`, EKS access entries) grants cluster-admin, which grants every workload identity in the cluster, which grants their cloud roles. It is a privilege *laundering* path across two IAM systems.

### 11.2 The escalation catalog

```
  create pods (or deployments/jobs/cronjobs/daemonsets/statefulsets)
      -> mount ANY service account token in the namespace     CRITICAL
      -> with hostPath/privileged/hostPID -> node compromise  CRITICAL
         -> node's kubelet credentials + node's cloud role
  get/list secrets
      -> service account tokens, and everything else          CRITICAL
  create pods/exec, pods/attach, pods/portforward
      -> code execution in an existing pod
  impersonate (users/groups/serviceaccounts)
      -> act as cluster-admin directly                        CRITICAL
  bind, escalate (on roles/clusterroles)
      -> grant yourself more than you have (by design!)       CRITICAL
  create/update rolebindings, clusterrolebindings
  patch/update nodes             -> scheduling manipulation
  certificatesigningrequests + approve
      -> mint a new client certificate as any user/group      CRITICAL
  create/update mutatingwebhookconfigurations
      -> intercept and modify every API object                CRITICAL
  update deployments/daemonsets in kube-system
  ephemeralcontainers                 -> code exec in running pods
```

`bind` and `escalate` are worth highlighting because they are *intentional* Kubernetes features that permit privilege escalation, and they are frequently granted to platform teams without anyone registering what they mean.

### 11.3 Modelling

```
  Nodes:   K8S_CLUSTER, K8S_NAMESPACE, K8S_SERVICE_ACCOUNT,
           K8S_ROLE, K8S_WORKLOAD, K8S_NODE
  Edges:   IDENTITY CAN_ASSUME K8S_SERVICE_ACCOUNT   (via IRSA/WI/MI annotation)
           K8S_SERVICE_ACCOUNT CAN_ASSUME ROLE       (the cloud bridge)
           K8S_WORKLOAD RUNS_AS K8S_SERVICE_ACCOUNT
           K8S_WORKLOAD RUNS_ON K8S_NODE
           K8S_NODE CAN_ASSUME ROLE                  (node instance role)
```

The IRSA / Workload Identity annotation is the critical join, and it lives in an annotation on the Kubernetes service account referencing a cloud role ARN. Both connectors — the cloud one and the Kubernetes one — must be present for the bridge to exist, which has a direct implication for connector packaging (Chapter 23).

---

## 12. Federation and cross-boundary trust

Federation edges are how attack paths cross clouds, and they are the least-instrumented part of most environments.

```
  IDP -> CLOUD
    Entra ID  --SAML/OIDC-->  AWS IAM roles
        role trust policy names the SAML provider + a condition on
        SAML:aud / SAML:sub. A loose condition = anyone in the tenant
        can assume the role.
    Okta      --SAML/OIDC-->  AWS / GCP / Azure
    Google Workspace --> GCP  (domain-wide delegation!)

  CI/CD -> CLOUD  (the modern default, and frequently misconfigured)
    GitHub Actions --OIDC--> AWS role
        trust condition on token.actions.githubusercontent.com:sub
        MISCONFIGURATION: sub condition of "repo:org/*" or missing
        entirely -> ANY repo in the org, or ANY GitHub repo anywhere,
        can assume the role.                                   CRITICAL
    GitLab CI, CircleCI, Buildkite --OIDC--> cloud
    Terraform Cloud --OIDC--> cloud

  CLOUD -> CLOUD
    GCP Workload Identity Federation <-- AWS / Azure / OIDC
    Azure federated identity credentials on app registrations
    AWS roles trusted by another account (with/without ExternalId)

  ON-PREM -> CLOUD
    AD FS, AAD Connect / Cloud Sync, Password Hash Sync,
    Pass-through Authentication agents, Seamless SSO (AZUREADSSOACC$)
```

### 12.1 The GitHub OIDC condition check

Worth stating as its own rule because it is now extremely common and the failure is silent:

```
  AWS role trust policy:
    "Condition": {
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:acme/*"     <-- BAD
      }
    }

  Should be:
        "token.actions.githubusercontent.com:sub":
            "repo:acme/payments-api:ref:refs/heads/main"             <-- GOOD

  With the loose form, any workflow in any acme repo — including one a
  contractor can open a PR against — can assume this role.

  Missing the sub condition entirely -> ANY GitHub repository ON EARTH
  can assume the role.                                          CRITICAL
```

Overlook should detect this from the trust policy alone, without needing the GitHub connector — and detect it *better* when the GitHub connector is present, because it can then enumerate which repositories and which contributors actually inherit the role.

---

# PART C — DIRECTORIES

---

## 13. Active Directory

### 13.1 Why AD cannot be a footnote

Overlook's headline claim is the **hybrid** attack path. Hybrid paths begin on-premises, because that is where the workforce authenticates and where the oldest, least-reviewed privilege lives. The system design gave AD one connector line. It needs BloodHound-class depth, because that is the bar the customer's red team already uses, and any gap will be found in the first POC.

If Overlook's AD model is shallower than the free tool the customer already runs, the "unified graph" claim collapses.

### 13.2 The AD node types

```
  AD_DOMAIN            AD_OU               AD_GPO
  AD_USER              AD_GROUP            AD_COMPUTER
  AD_SERVICE_ACCOUNT   AD_GMSA             AD_TRUST
  AD_CONTAINER         AD_CERT_TEMPLATE    AD_CA
```

`AD_COMPUTER` matters more than people expect: computer accounts are principals, they can be delegated to, and RBCD abuse works by writing to a computer object.

### 13.3 ACL-derived edges

Directory object ACLs are a permission system as expressive as cloud IAM, and almost nobody audits them. Each ACE becomes an edge:

```
  GenericAll             -> full control                  CAN_MODIFY (total)
  GenericWrite           -> write most attributes         CAN_MODIFY
  WriteDacl              -> rewrite the ACL -> GenericAll CAN_MODIFY (total)
  WriteOwner             -> take ownership -> WriteDacl   CAN_MODIFY (total)
  Owns                   -> implicit WriteDacl
  AllExtendedRights      -> includes password reset       CAN_MODIFY
  ForceChangePassword    -> reset the password            CAN_ASSUME
  AddMember / Self       -> join a group                  MEMBER_OF (potential)
  WriteProperty on
      servicePrincipalName        -> targeted Kerberoast  CAN_ASSUME
      msDS-KeyCredentialLink      -> shadow credentials   CAN_ASSUME  CRITICAL
      msDS-AllowedToActOnBehalfOf -> RBCD                 CAN_ASSUME  CRITICAL
      member                      -> add to group
      gPCFileSysPath              -> GPO content control  CAN_EXECUTE
  GetChanges + GetChangesAll      -> DCSync = every hash  CAN_ASSUME(domain) CRITICAL
  ReadLAPSPassword (ms-Mcs-AdmPwd or msLAPS-Password)
                                  -> local admin on that host  CAN_EXECUTE
```

**WriteDacl and WriteOwner are transitive to full control** and must be expanded as such. An analyst seeing "WriteOwner on Domain Admins" may not realise it is equivalent to membership; the graph must make it so.

### 13.4 Kerberos delegation

The three delegation types, in ascending order of how badly they are understood:

```
  UNCONSTRAINED DELEGATION  (TrustedForDelegation)
     Any user authenticating to this host leaves a TGT in its memory.
     Compromise the host -> impersonate anyone who touched it,
     including, with a coercion trick, a Domain Controller.
     Edge:  AD_COMPUTER CAN_ASSUME AD_USER (any who authenticate)
            + CAN_ASSUME AD_DOMAIN if coercion is feasible    CRITICAL

  CONSTRAINED DELEGATION  (msDS-AllowedToDelegateTo)
     Impersonate any user to the listed services.
     With protocol transition (TRUSTED_TO_AUTH_FOR_DELEGATION),
     no prior authentication is needed at all.
     Edge:  AD_COMPUTER/SA CAN_ASSUME <target service's host>  HIGH

  RESOURCE-BASED CONSTRAINED DELEGATION  (msDS-AllowedToActOnBehalfOfOtherIdentity)
     Configured on the TARGET, so anyone who can WRITE that attribute
     on a computer object can take it over. Combined with the default
     MachineAccountQuota of 10 (any user may join 10 computers),
     this is a one-hop takeover available to ordinary users.
     Edge:  <anyone with WriteProperty on the target> CAN_ASSUME target  CRITICAL
```

RBCD plus a non-zero `MachineAccountQuota` is one of the highest-value findings available in an AD environment, and it is invisible to every posture tool that only reads group memberships.

### 13.5 Certificate services (AD CS)

An entire escalation surface that lives in a separate service and is routinely unmonitored. Model the well-known template misconfiguration classes:

```
  Template allows requester-supplied subject (SAN) + client auth EKU
      + enrollable by ordinary users
      -> request a certificate as Domain Admin                CRITICAL
  Template with "Any Purpose" or "Certificate Request Agent" EKU
  Enrollment agent misconfiguration -> request on behalf of others
  Vulnerable CA ACLs (ManageCA / ManageCertificates)
      -> approve your own request / re-enable templates       CRITICAL
  NTLM relay to the CA web enrollment endpoint
  Weak certificate mapping / no strong mapping enforcement

  Edge: AD_USER CAN_ASSUME AD_USER(any) via AD_CERT_TEMPLATE
```

### 13.6 Other AD structures

```
  GPO abuse
     write access to a GPO linked to an OU containing servers
     -> code execution on every computer in that OU           CRITICAL
     Edge: AD_USER CAN_EXECUTE AD_COMPUTER (for each linked object)

  Nested groups
     effective membership must be transitively resolved.
     Real estates reach 8-12 levels of nesting, and the deep ones
     are exactly where forgotten privilege accumulates.

  AdminSDHolder
     write access -> persistent rights on all protected groups

  Kerberoasting
     a privileged user account with an SPN and a weak password
     -> offline crack -> CAN_ASSUME
     Overlook can identify the CANDIDATES (privileged + SPN +
     password age) without ever attempting a crack.

  AS-REP roasting
     DONT_REQ_PREAUTH on a privileged account

  Trusts
     intra-forest, cross-forest, external, with/without SID filtering.
     SID history abuse crosses trust boundaries.
     Edge: AD_DOMAIN TRUSTS AD_DOMAIN (with filtering attributes)

  Stale / shadow admins
     principals with privileged ACLs but no group membership —
     invisible to "who is in Domain Admins?" and therefore to
     most tools and most auditors.                            <- high value
```

### 13.7 Collection method

An unavoidable design point: comprehensive AD collection means LDAP enumeration at a scale that resembles reconnaissance, and it will trip the customer's own detections. Handle it deliberately:

- Read-only LDAP/LDAPS with a dedicated, least-privilege account
- Paged, throttled queries with configurable rate limits
- A published collection profile the customer can hand to their SOC to allowlist
- Schedule-aware — full sweep nightly, delta by `uSNChanged` in between
- Optional: consume the customer's existing BloodHound/SharpHound output rather than collecting again, for customers who already run it

That last option is worth building. It is cheap, it respects an existing investment, and it removes the objection "we don't want another tool doing LDAP sweeps."

---

## 14. Entra ID

### 14.1 What makes Entra different

Entra is not "AD in the cloud." It is a different model with different escalation paths, and the interesting privilege lives in **applications**, not users.

```
  APPLICATION-CENTRIC PRIVILEGE
    App registration  ->  Service principal  ->  API permissions
                                              ->  app roles
                                              ->  delegated permissions
                                              ->  credentials (secrets/certs)
                                              ->  federated identity credentials

  An app with RoleManagement.ReadWrite.Directory is Global Admin.
  Nobody reviews app permissions with the rigour they review admin groups.
```

### 14.2 The Entra escalation surface

```
  GRAPH APPLICATION PERMISSIONS (app-only, no user context)
    RoleManagement.ReadWrite.Directory   -> assign self GA        CRITICAL
    AppRoleAssignment.ReadWrite.All      -> grant self anything   CRITICAL
    Application.ReadWrite.All            -> add creds to any SP   CRITICAL
    Directory.ReadWrite.All              -> broad directory write
    Group.ReadWrite.All                  -> join privileged groups
    User.ReadWrite.All                   -> password reset (non-admin)
    PrivilegedAccess.ReadWrite.AzureAD   -> PIM manipulation
    Policy.ReadWrite.ConditionalAccess   -> disable CA policies   CRITICAL

  DELEGATED PERMISSIONS + CONSENT
    user consent enabled for unverified apps -> illicit consent grants
    admin consent granted broadly            -> standing app privilege
    OAuth apps with offline_access + mail    -> persistent mailbox access

  OWNERSHIP (the forgotten one)
    App owner  -> can add credentials       -> become the SP    CRITICAL
    SP owner   -> same
    Group owner-> can add members           -> inherit its roles
    Ownership is NOT a role assignment and is missed by role-based reviews.

  DIRECTORY ROLES
    Global Administrator, Privileged Role Administrator,
    Privileged Authentication Administrator, Application Administrator,
    Cloud Application Administrator, Groups Administrator,
    User Administrator, Authentication Administrator,
    Hybrid Identity Administrator (-> AAD Connect -> on-prem)
    Partner Tier2 Support (legacy, extremely powerful)

  IDENTITY INFRASTRUCTURE
    AAD Connect sync account            -> on-prem <-> cloud pivot  CRITICAL
    Seamless SSO computer account (AZUREADSSOACC$) -> silver tickets
    Federated domain / token-signing certificate
        -> forge SAML tokens for ANY user (Golden SAML)             CRITICAL
    Federated identity credentials on app registrations
        -> external OIDC issuer can obtain tokens as the app
```

### 14.3 Conditional Access as PROTECTS edges

Conditional Access is Entra's real access-control layer, and it must be modelled — otherwise the graph treats an MFA-protected admin role identically to an unprotected one.

```
  CA policy: "require MFA for admin roles"
      -> PROTECTS edge on the relevant CAN_ASSUME edges
      -> reduces edge weight

  But model the GAPS, which is where findings come from:
      - excluded users / break-glass accounts (often forgotten,
        often not monitored, often with static passwords)
      - excluded applications
      - legacy authentication still permitted -> bypasses CA entirely
      - report-only policies mistaken for enforced ones
      - device-compliance conditions with no MDM actually enrolled
      - "trusted location" exclusions covering a wide corporate range

  FINDING: "Global Administrator role is reachable by 4 identities;
            2 of them are excluded from the MFA CA policy."
```

---

## 15. Standing vs eligible privilege

### 15.1 PIM changes the meaning of an edge

Privileged Identity Management (Entra), and equivalents (AWS IAM Identity Center permission sets with approval, GCP privileged access requests), introduce a state the current graph model cannot express: **privilege that is not active but can be activated.**

```
  STANDING     the principal holds it right now
               -> full-weight edge

  ELIGIBLE     the principal can activate it, possibly with
               MFA / approval / justification / time bound
               -> real edge at reduced weight, NOT a non-edge

  ACTIVATED    currently active, time-bounded
               -> full-weight edge with an expiry
```

Treating eligible as "no edge" is the common mistake and it understates risk badly: an attacker who compromises an eligible user simply activates. Treating it as equal to standing overstates risk and buries the genuinely worse standing grants.

### 15.2 Modelling

```jsonc
{
  "predicate": "CAN_ASSUME",
  "to": "ROL-global-admin",
  "attributes": {
    "assignment_type": "ELIGIBLE",
    "activation_requires": ["mfa", "approval", "justification"],
    "approvers": ["IDN-a1", "IDN-b2"],
    "max_duration_hours": 8,
    "activation_count_90d": 0,        // never used -> remove it
    "last_activated": null
  },
  "weight": 0.55                       // vs 0.95 for standing
}
```

`activation_count_90d = 0` is a strong rightsizing signal: eligibility nobody ever activates is pure attack surface with no operational value.

### 15.3 The findings this enables

```
  "12 identities are ELIGIBLE for Global Administrator.
   3 have never activated it in 90 days.
   1 can activate without approval."

  "svc-automation holds STANDING Owner on 40 subscriptions
   where eligible-only would suffice — it activates on a
   predictable weekly schedule."

  "Break-glass account BG-01 has standing Global Administrator,
   is excluded from Conditional Access, and has not rotated
   its password in 640 days."
```

---

# PART D — DYNAMICS

---

## 16. ABAC and conditional edges

### 16.1 The problem ABAC creates

The graph model in the system design assumes edges are discovered by enumeration and change when someone changes a policy. ABAC breaks that assumption: the edge exists **because of an attribute value**, and attributes change constantly, outside any policy review.

```
   Policy:
     Allow ec2:* where  aws:PrincipalTag/team == aws:ResourceTag/team

   Consequence:
     Every principal tagged team=payments can control every resource
     tagged team=payments.

     Change one tag on one principal  ->  thousands of edges appear
     instantly, with no policy change, no approval, and no audit
     event that any posture tool watches.
```

### 16.2 Modelling: predicates, not materialized edges

```jsonc
{
  "type": "CONDITIONAL_CAPABILITY",
  "principal_selector": { "tag": "team", "op": "EQUALS_RESOURCE_TAG" },
  "resource_selector":  { "type": "ec2:instance", "tag": "team" },
  "capability": "CAN_EXECUTE",
  "current_match_count": 4120,
  "expands_to_sample": ["AST-1a2b", "AST-3c4d", "..."],
  "volatility": "HIGH"
}
```

Then:

- **Materialize** the expansion only for crown jewels and their immediate neighbourhood.
- **Keep the predicate** for everything else, expanding on demand at query time.
- **Watch the attribute, not the policy.** A tag change on a principal is a privilege change and should be a first-class event in the change feed.

### 16.3 The ABAC-specific findings

```
  "Any principal that can SET ITS OWN TAGS in an ABAC environment
   can grant itself access to every resource in every team."
   -> tag-write permission is an escalation primitive
      whenever ABAC is in use.                                 CRITICAL

  "Tag 'team' is used in 14 access-control policies and is
   writable by 340 principals."

  "Resource DST-1c4b lost its 'team' tag 6 days ago and is now
   accessible to nobody / to everybody."
```

The first is the important one, and it generalises: **in an ABAC environment, `iam:TagRole`, `ec2:CreateTags`, and their equivalents become escalation primitives.** Most tools never make that connection because it depends on knowing that the tag is load-bearing.

---

## 17. Non-human identity lifecycle

### 17.1 The scale reality

```
   Typical enterprise ratio:   1 human : 10-50 non-human identities

   10,000 employees  ->  100,000-500,000 NHIs
     service accounts, workload identities, app registrations,
     CI/CD identities, computer accounts, API integrations,
     and now agent identities

   Characteristics that make them worse than human identities:
     - no MFA, ever
     - credentials that rarely rotate
     - frequently no owner
     - permissions granted at creation and never reviewed
     - dormancy is invisible (nobody notices a bot not logging in)
     - often more privileged than any human
```

### 17.2 The NHI attributes to track

```
  ownership          owner identity or team; NULL is itself a finding
  creation           created_at, created_by, creation_method (console = smell)
  credentials        count, types, oldest_age, last_rotated,
                     stored_where (a link to SECRET nodes)
  usage              last_used, used_permission_count, usage_pattern
                     (regular schedule vs sporadic vs never)
  privilege          effective tier, crown jewels reachable,
                     escalation primitives held
  provenance         Terraform / manual / auto-created by a service
  federation         is it key-based or federated? (key-based is worse)
```

### 17.3 The findings that write themselves

```
  ORPHANED       privileged NHI whose creator has left the organisation
                 and which has no owner tag -> nobody will ever review it
  DORMANT        no usage in 90+ days but retains admin permissions
  STALE CREDS    access key older than 365 days, never rotated
  MULTI-KEY      more than one active credential (a rotation that never
                 completed — the old key still works)
  OVER-SCOPED    granted 340 permissions, used 12
  SHARED         one identity used from many disparate sources
                 (a shared credential, pasted somewhere)
  HUMAN-USED     a service account with interactive console logins
                 -> a human is using a bot identity to bypass controls
  KEY OVER FED   static keys where workload identity federation
                 is available
  AGENT NHI      an NHI that is an AI agent's runtime identity —
                 same risks, plus an untrusted instruction channel
```

### 17.4 The agent identity connection

This is where the IAM document meets the AI story, and it is worth being explicit because it is the strongest argument for putting both in one graph:

```
   An AI agent identity is a non-human identity with three differences:

   1. It can be INDUCED to act by anyone who can influence its input —
      a prompt, a document it retrieves, a tool result it reads.
      No credential theft required.

   2. Its action space is not fixed at deploy time. Add an MCP server,
      and its capabilities change without any IAM change.

   3. Its usage pattern looks like a human's (irregular, varied)
      but its privilege is a service account's (broad, standing).
      Behavioural baselining designed for either one fails on it.

   Therefore, in the risk model:
       exposure(agent_identity) = exposure(service_account)
                                + reachability_by_untrusted_input

   And the reachability term is what makes an agent with modest
   permissions more dangerous than a service account with the same ones.
```

---

## 18. Secrets as identity edges

### 18.1 The reframing

A leaked credential is not a "secrets finding." It is an **identity edge**: whoever can read the secret can act as the principal it authenticates.

```
   CONVENTIONAL VIEW
     Finding: "AWS access key found in repository payments-api."
     Severity: High. Assigned to the repo owner. Sits in a backlog.

   GRAPH VIEW
     REP-payments  CONTAINS  SEC-4a2f
     SEC-4a2f      AUTHENTICATES_AS  IDN-svc-deploy
     IDN-svc-deploy CAN_ASSUME  ROL-devopsadmin
     ROL-devopsadmin CAN_READ  DST-prod-payments  [PII, 4.2M]

     Therefore, synthesized:
     every identity with read access to REP-payments
       CAN_READ DST-prod-payments

     -> 340 contributors, 12 external collaborators, and every
        CI job in that repo inherit production database access.
```

The second framing gets fixed the same week. The first sits in a backlog for a quarter. **Same data, different model.**

### 18.2 What to collect

```
  WHERE SECRETS LIVE (all become CONTAINS edges)
    source repos (current tree AND git history — history is the hard part)
    CI/CD variables and secrets
    container images (layers, env, build args)
    IaC state files (Terraform state is a chronic offender)
    Kubernetes Secrets (base64 is not encryption)
    cloud secret managers (the intended place — track WHO CAN READ)
    config files on hosts (via the Agent)
    wikis, tickets, chat  (connector-dependent; often the worst offender)
    AI prompts  (via the AI Gateway — a secret pasted into a prompt
                 is an exfiltrated secret)

  WHAT TO RECORD  (never the value itself)
    secret type and provider
    fingerprint / hash for correlation across locations
    the principal it authenticates as   <- the critical link
    validity (active / revoked / unknown)
    age, exposure surface, first_seen
```

### 18.3 The correlation that only a graph can do

```
   The same secret fingerprint appears in:
     - repo payments-api (git history, commit 4a2f, 2024-11)
     - the CI variable DEPLOY_KEY
     - a container image layer in the registry
     - a config file on 14 hosts
     - one prompt to an external AI service          <-- exfiltrated

   It authenticates as svc-deploy, which is ADMIN, and it has
   never been rotated.

   Blast radius of this ONE secret: 3 crown jewels, 340 identities
   with access to at least one of its locations.
```

---

## 19. CIEM

### 19.1 What CIEM adds

Cloud Infrastructure Entitlement Management is the discipline of measuring and reducing the gap between granted and used. The system design has no concept of it; it should, because Overlook already collects everything required and it converts the product from *"here is your risk"* to *"here is the fix, pre-written."*

### 19.2 The data

```
  AWS    CloudTrail (management + data events)
         IAM GetServiceLastAccessedDetails (service and action granularity)
         Access Analyzer unused-access findings
  Azure  Activity Log, Entra sign-in logs,
         Entra Permissions Management (if licensed)
  GCP    Cloud Audit Logs, IAM Recommender
  K8s    audit log
  AD     4624/4768/4769 events, last logon timestamps
```

All of this is read and aggregated **at the Edge Collector**. Only the derived counts and the proposed policy leave — usage data is extremely sensitive and voluminous, so this is another case where the privacy architecture is also the efficient architecture.

### 19.3 The metrics

```
  PERMISSION UTILISATION  =  used / effective
       < 5%   severe over-provisioning (the common case)
       > 60%  well-scoped (rare)

  PRIVILEGE VELOCITY      =  rate of permission accumulation over time
       identities only ever gain permissions; nobody removes them.
       A steadily climbing curve on a service account is a finding.

  DORMANCY                =  days since last use, per identity and
                             per permission

  BLAST-RADIUS EFFICIENCY =  crown jewels reachable / crown jewels
                             actually accessed
       reaching 3 and using 1 means 2 are pure risk
```

### 19.4 Time windows

```
   30 days   too short — misses monthly processes
   90 days   the default. Balances coverage against staleness.
   365 days  needed for annual jobs (audits, renewals, year-end)

   Always report what a shorter window would have missed:
     "3 permissions were last used 91-180 days ago.
      A 90-day policy would have removed them."
```

That transparency is what makes an automated rightsizing recommendation safe to act on, and it is exactly what customers ask for before they will trust a generated policy.

---

# PART E — OUTPUT

---

## 20. The IAM findings catalog

What the IAM layer actually produces, grouped by class. This is the concrete output of everything above.

### 20.1 Structural exposure

```
  IAM-001  Privilege escalation path to administrator          CRITICAL
  IAM-002  Cross-account trust with no ExternalId              HIGH
  IAM-003  Trust policy with wildcard principal                CRITICAL
  IAM-004  OIDC trust with insufficient subject condition      CRITICAL
  IAM-005  Entra role reaching Azure resource plane            CRITICAL
  IAM-006  K8s RBAC to cloud role bridge                       HIGH
  IAM-007  Escalation via tag-write in an ABAC environment     HIGH
  IAM-008  SA impersonation chain longer than 3 hops           MEDIUM
  IAM-009  Permission boundary absent on a role with PassRole  HIGH
  IAM-010  SCP gap allowing a known escalation                 HIGH
```

### 20.2 Identity hygiene

```
  IAM-020  Orphaned privileged NHI (no owner)                  HIGH
  IAM-021  Dormant identity retaining admin permissions        HIGH
  IAM-022  Credential older than 365 days                      MEDIUM
  IAM-023  Multiple active credentials (incomplete rotation)   MEDIUM
  IAM-024  Service account with interactive logins             MEDIUM
  IAM-025  Static keys where federation is available           MEDIUM
  IAM-026  Human identity without MFA reaching a crown jewel   CRITICAL
  IAM-027  Break-glass account excluded from Conditional Access HIGH
  IAM-028  Shared credential used from disparate sources       HIGH
```

### 20.3 Entitlement

```
  IAM-040  Permission utilisation below 5% on a privileged identity  MEDIUM
  IAM-041  Unused escalation primitive held                    HIGH
  IAM-042  Standing privilege where eligible-only would suffice MEDIUM
  IAM-043  Eligible role never activated in 90 days            LOW
  IAM-044  Wildcard resource on a destructive action           HIGH
```

### 20.4 Directory

```
  IAM-060  DCSync rights held by a non-DC principal            CRITICAL
  IAM-061  Unconstrained delegation on a non-DC host           CRITICAL
  IAM-062  RBCD writable by a low-privilege principal          CRITICAL
  IAM-063  Shadow admin (privileged ACL, no group membership)  HIGH
  IAM-064  GPO writable by a non-admin, linked to servers      CRITICAL
  IAM-065  AD CS template permits requester-supplied SAN       CRITICAL
  IAM-066  Kerberoastable privileged account, old password     HIGH
  IAM-067  MachineAccountQuota > 0 with RBCD exposure          HIGH
  IAM-068  Nested group depth exceeding 8 with privilege at leaf MEDIUM
```

### 20.5 AI-specific

```
  IAM-080  AI Privilege Gap (user < agent privilege)           CRITICAL
  IAM-081  Agent identity with standing admin                  CRITICAL
  IAM-082  Agent identity reachable by untrusted input         HIGH
  IAM-083  MCP server credential grants more than the agent needs HIGH
  IAM-084  RAG retrieval identity broader than its query audience CRITICAL
  IAM-085  AI service credential in a repository                HIGH
```

Note that IAM-080 through IAM-085 are *IAM findings*, not AI findings. That is the point of one graph: the AI risk is expressed in the language of entitlements, which is a language the customer's identity team already speaks and already has a process for.

---

## 21. Remediation

### 21.1 Every IAM finding must ship a fix

An IAM finding without a concrete, applicable remediation is a to-do item. With one, it is a product.

```
  FINDING  IAM-001  Escalation to administrator
           svc-devops-ai holds iam:PassRole (Resource: *) +
           lambda:CreateFunction + lambda:InvokeFunction

  FIX 1 — Constrain PassRole  (recommended)
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::123456789012:role/lambda-exec-*",
      "Condition": {
        "StringEquals": { "iam:PassedToService": "lambda.amazonaws.com" }
      }
    }
    Effect: removes 41 synthesized CAN_ASSUME edges,
            eliminates 1,240 paths, resolves 3 crown-jewel exposures.
    Risk:   2 existing Lambda deployments pass roles outside the
            new pattern and will break. Listed below.

  FIX 2 — Permission boundary
  FIX 3 — Remove lambda:CreateFunction (used 0 times in 90 days)

  OUTPUT   [ Copy JSON ]  [ Terraform diff ]  [ Open Jira ]  [ Create PR ]
```

### 21.2 Generated least-privilege policies

The combination of *effective* and *used* produces a policy directly:

```
   INPUT   effective capability set  (what it can do)
         + 90-day usage              (what it did)
         + resource patterns actually touched
         + escalation primitives to exclude explicitly

   OUTPUT  a least-privilege policy document
         + a diff against current
         + a predicted graph delta
         + a list of usage in the 90-365 day window that the
           new policy would break                    <- always show this
```

### 21.3 Break Attack Path, applied to IAM

IAM is where choke points are most concentrated, because a single over-permissive role is typically the shared hop in thousands of paths:

```
   CHOKE POINT  ROL-ghadeploy CAN_ASSUME ROL-ec2app
   Appears in 1,240 paths, 9 reaching crown jewels.

   This is one line in one policy.
   Removing it is a 4-hour change.
   It eliminates 26% of the tenant's critical exposure.
```

That sentence — *one line, four hours, 26%* — is the product's value proposition in its most concentrated form, and it is only computable because the IAM layer is deep enough to know what the edge means.

---

## 22. Confidence in IAM edges

IAM inferences vary enormously in reliability, and hiding that variation is how a product loses an argument with a cloud engineer. Publish it:

```
  0.99  Verified by provider simulation API
  0.97  Static analysis, unconditional, single account, direct grant
  0.93  Static analysis with resolvable conditions
  0.90  Escalation primitive with all preconditions verified
  0.85  Escalation primitive, preconditions partially verified
  0.80  Cross-account, both sides collected
  0.70  Cross-account, only one side collected (inferred)
  0.65  ABAC expansion from current attribute values
  0.55  Inferred from audit-log activity only (we saw it happen,
        we could not find the grant that permitted it)
  0.40  Derived from a stale source outside its coverage window
```

The 0.55 case is interesting and worth building deliberately: if CloudTrail shows a principal performing an action Overlook's static analysis says it cannot perform, that is either a collection gap or an evaluation bug — and either way it is a **self-diagnostic signal** that the IAM engine should surface to engineering. Track the rate of these; it is the best available proxy for closure correctness.

---

## 23. What this implies for connectors

The IAM layer sets the connector requirements, not the other way round. Stated here as the input to the connector discussion:

### 23.1 Connectors required for a *credible* IAM graph

```
  TIER 0 — without these there is no product
    AWS Organizations + IAM + Access Analyzer + CloudTrail
    Azure ARM RBAC + Entra ID (Graph) + Activity Log
    GCP Cloud Resource Manager + IAM + Cloud Audit Logs
    Active Directory (LDAP)
    Okta or Entra as the workforce IdP

  TIER 1 — required for the flagship findings
    GitHub / GitLab (OIDC trusts, repo access, secrets in code)
    Kubernetes (EKS/AKS/GKE + RBAC + service account annotations)
    AWS Secrets Manager / Azure Key Vault / GCP Secret Manager
    AD CS (certificate templates)

  TIER 2 — depth
    AWS IAM Identity Center, HashiCorp Vault, CyberArk,
    ServiceNow (ownership data), Workday (joiner/mover/leaver),
    Terraform Cloud, Jenkins, Snowflake, Databricks, Salesforce
```

### 23.2 The multiplicative property

Connector value is **not additive**. Individual connectors produce inventory; *pairs* produce paths.

```
   AWS alone                       -> cloud-only paths
   AWS + Entra                     -> federated paths          (x)
   AWS + Entra + GitHub            -> supply-chain paths       (xx)
   AWS + Entra + GitHub + AD       -> full hybrid paths        (xxx)
   + Kubernetes                    -> the workload identity bridge
   + secrets manager               -> credential-derived paths

   Each addition multiplies the paths findable, because a path
   needs EVERY hop present. One missing connector in the middle
   makes the whole path invisible — not shorter, INVISIBLE.
```

This has a direct consequence for sales and for the POC: **a POC with one cloud connector demos inventory; a POC with cloud + IdP + repo demos the product.** The minimum credible demo set is not one connector, it is four.

That is the argument to carry into the connector discussion.

---

## 24. What the IAM layer must deliver

The IAM layer is built as one body of work, not staged. Everything below is in scope from the start, because each piece is load-bearing for the graph's *correctness* — and a graph that is broad but wrong is worse than one that is narrow and true.

### 24.1 The correctness spine

```
  effective permission closure for AWS, Azure and GCP
  escalation primitive engine + catalogs (AWS, Azure, GCP, Kubernetes)
  Entra ID depth: app registrations, SPs, Graph permissions,
      ownership, Conditional Access, the elevateAccess bridge
  Active Directory: ACLs, all three delegation types, AD CS,
      GPO, nested groups, shadow admins
  granted / effective / used, with CIEM rightsizing
  PIM standing vs eligible privilege
  ABAC conditional capabilities
  non-human identity lifecycle
  secrets as identity edges
  provider simulation verification for high-value edges
  the published confidence model
```

None of these is optional and none is deferrable, because each one, if absent, produces **silent false negatives** — attack paths that exist in the customer's environment and not in the graph. A missing connector is a visible gap that coverage reporting will show. A missing escalation primitive is an invisible one.

### 24.2 The lead-time ordering

The only sequencing that exists inside the IAM layer is dependency lead time, measured in weeks:

```
   capability schema + action-group mapping   leads everything      ~3 weeks
   permission closure per cloud               leads escalation      ~4 weeks
   canonical keys + Resolution Directory      leads all connectors  ~4 weeks
   effective permissions                      leads CIEM            ~2 weeks
   closure + audit-log ingestion              leads the used state  ~2 weeks

   Directory work (AD, Entra), ABAC, NHI lifecycle and the secrets
   bridge have no prerequisites beyond the capability schema and
   proceed fully in parallel.
```

### 24.3 The five findings this layer exists to produce

```
  1. AI PRIVILEGE GAP        user < agent privilege, reaching a crown jewel
  2. ESCALATION TO ADMIN     synthesized path no policy declares
  3. HYBRID PATH             AD -> Entra -> AWS -> production data
  4. CHOKE POINT             one policy line, N thousand paths
  5. RIGHTSIZE               340 permissions granted, 12 used, policy attached
```

None is available from any single tool the customer already owns. All are computable from the foundation connector set with no endpoint agent and no AI gateway — they are pure consequences of a correct IAM graph.

That is the product.

---

## Appendix — What this document adds to the system design

| Area | System design | This document |
|---|---|---|
| Permission model | "compute closure at the Edge" | Granted / effective / used, per-cloud precedence, capability sets |
| Escalation | not present | Primitive schema + catalogs for AWS, Azure, GCP, K8s |
| Active Directory | one connector line | ACLs, delegation ×3, AD CS, GPO, shadow admins, collection strategy |
| Entra ID | not distinguished from AD | App-centric privilege, Graph permissions, ownership, CA gaps, the resource-plane bridge |
| PIM / eligibility | not present | Standing vs eligible vs activated, with weights |
| ABAC | not present | Conditional capabilities, tag-write as an escalation primitive |
| NHI | mentioned | Full lifecycle model, findings, agent-identity connection |
| Secrets | a node type | Secret as an identity edge, cross-location correlation |
| CIEM | not present | Utilisation, dormancy, rightsizing, generated policies |
| Closure scale | not addressed | Action grouping, pattern preservation, bounded chains, lazy expansion |
| Build vs borrow | not addressed | Hybrid static + provider verification, free ground truth |
| Confidence | generic | IAM-specific scale with a self-diagnostic signal |
| Connectors | listed | Tiered by what the IAM graph requires; the multiplicative argument |

---

*End of document.*

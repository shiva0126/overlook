# E7 — Permission Closure Engine

**Series:** [Engine documentation](00-index.md) · **v1:** required · **one of the four hardest**

---

## 1. Purpose

The Permission Closure Engine answers one question for every principal in the environment: **what can this identity actually do, right now, after every layer of policy has been evaluated?**

It converts policy documents — which describe intent — into capabilities, which describe reality. Roughly 73% of TrustGraph edges originate here (`../02 §Why this document exists`), which makes it the engine that determines whether the graph is true.

---

## 2. Position

```
  INPUT   observations carrying grants and constraints
            AWS: identity policies, resource policies, SCPs, RCPs,
                 permission boundaries, session policies, trust policies
            Azure: role assignments, deny assignments, custom roles,
                 ABAC conditions, management group hierarchy
            GCP: allow policies, deny policies, hierarchy bindings,
                 CEL conditions
            AD: ACEs, group memberships, delegation attributes
            K8s: Roles, ClusterRoles, bindings, service accounts

  OUTPUT  capability sets per principal → converted to graph edges
          with predicate, resource pattern, conditions and confidence

  FEEDS   E8 Escalation Matcher (hard dependency, ordered)
          E12 Graph Engine
```

---

## 3. Mechanics

### 3.1 The pipeline is always the same

```
  GRANTS − CONSTRAINTS → CAPABILITIES → EDGES

  A connector collects grants and constraints.
  E7 computes capabilities.
  Only capabilities become edges.
```

### 3.2 Evaluation precedence — get this wrong and the graph lies

**AWS**, in strict order, where any explicit deny terminates immediately:

```
  1  explicit DENY in ANY policy type          → DENY, stop
  2  Service Control Policy                    → if not allowed, DENY
  3  resource-based policy                     → may ALLOW outright
                                                  (cross-account: usually
                                                   BOTH sides needed, but
                                                   a per-service lookup
                                                   table decides — S3, KMS,
                                                   Lambda, SNS, SQS and
                                                   Secrets Manager differ)
  4  permission boundary                       → if not allowed, DENY
  5  session policy                            → if not allowed, DENY
  6  identity-based policy                     → if allowed, ALLOW
  7  otherwise                                 → implicit DENY

  Subtleties that must be encoded:
    boundaries and SCPs CAP, they never GRANT
    SCPs do not apply to the management account or service-linked roles
    NotAction / NotResource invert set semantics — handle explicitly
```

**Azure** — two planes, plus the bridge between them:

```
  1  deny assignments      → DENY, beats even Owner
  2  Azure Policy (deny)   → blocks the operation
  3  role assignments      → union of all applicable, INHERITED DOWN
                             management group → subscription →
                             resource group → resource
  4  ABAC conditions on assignments
  5  otherwise             → DENY

  THE BRIDGE: Global Administrator can elevateAccess to become
  User Access Administrator at the root management group, which is
  Owner on every subscription. One toggle, total cloud compromise.
  This MUST be materialized as an edge.
```

**GCP** — hierarchy and impersonation:

```
  1  IAM Deny policies     → evaluated first
  2  union of allow policies across org → folder → project → resource
  3  CEL conditions on bindings
  4  otherwise             → DENY

  The GCP hazard is service account impersonation: getAccessToken,
  actAs, signJwt, signBlob and serviceAccountKeys.create are each a
  CAN_ASSUME edge, and SA-to-SA chains 4-5 deep are normal.
```

### 3.3 Capability representation

Store evaluation results as capability sets, not as parsed policy:

```jsonc
{
  "principal": "IDN-svc-devops-ai",
  "capabilities": [{
    "action_pattern":   "s3:Get*",
    "resource_pattern": "arn:aws:s3:::prod-payments-*/*",
    "effect": "ALLOW",
    "granted_via": ["identity_policy:pol-4f2a", "resource_policy:bkt-9c1e"],
    "capped_by":   ["boundary:DevBoundary"],
    "conditions": [{
      "key": "aws:SourceVpce", "op": "StringEquals",
      "satisfiability": "CONDITIONAL"
    }],
    "unconditional": false
  }]
}
```

### 3.4 Condition satisfiability

Conditions must survive the trip, classified rather than raw:

| Class | Meaning | Effect on edge weight |
|---|---|---|
| `ALWAYS` | No condition, or always true here | full |
| `CONDITIONAL` | Satisfiable from a position an attacker may already hold (inside the VPC, correct source IP) | reduced |
| `HARD` | Requires something unlikely to be obtained (`MultiFactorAuthPresent`, hardware-bound) | heavily reduced |
| `UNSATISFIABLE` | References a nonexistent principal or tag | **edge not created** |

Classifying satisfiability needs environmental context, which is why it happens on the appliance and ships as a class rather than a raw condition block.

### 3.5 Action-to-capability mapping

A permission string is not a capability. The mapping is versioned content:

```
  s3:GetObject, dynamodb:GetItem       → CAN_READ
  s3:PutObject, s3:DeleteObject        → CAN_WRITE
  sts:AssumeRole                       → CAN_ASSUME
  ssm:SendCommand, ssm:StartSession    → CAN_EXECUTE
  ec2:RunInstances                     → CAN_DEPLOY
  kms:Decrypt                          → CAN_READ   ← encryption is only
                                          a control if the principal
                                          lacks decrypt
  secretsmanager:GetSecretValue        → CAN_READ + an IDENTITY edge,
                                          because reading a credential
                                          means becoming what it
                                          authenticates as
  iam:PassRole                         → ESCALATION_INGREDIENT, not a
                                          capability on its own
```

### 3.6 Scale control — four techniques, applied together

Naive materialization is impossible: 400 accounts × 12,000 principals × 18,000 actions × unbounded resources is trillions of triples.

```
  1  ACTION GROUPING
     18,000 actions → 40 groups → 12 capabilities.
     Retain specific actions ONLY for escalation primitives,
     rightsizing output and evidence.          ≈ 400:1

  2  RESOURCE PATTERN PRESERVATION
     do NOT expand arn:aws:s3:::prod-* into 4,000 edges.
     Keep the pattern; expand lazily for crown jewels only.
     This is the largest saving, because wildcard grants are
     exactly the ones that would explode.

  3  BOUNDED ASSUME-CHAIN CLOSURE
     max depth 6 · prune below confidence 0.4 · collapse cycles
     · equivalence-class roles with identical policy hashes
       (380 identical Terraform-deployed roles → 1 class × 380)

  4  LAZY EXPANSION
     PRECOMPUTE: CAN_ASSUME closure, MEMBER_OF closure, escalation
                 edges, capabilities to crown jewels
     ON DEMAND:  full capability set for one principal, blast radius,
                 reachability to non-crown-jewel resources
```

### 3.7 Incremental recomputation

```
  IAM change detected
    → affected principal set = the named principal
                             + anyone who CAN_ASSUME them (reverse closure)
                             + group members if a group policy changed
    → recompute capabilities for that set only
    → diff → emit only changed edges

  BUDGET
    full closure, 400-account estate:  < 30 minutes
    incremental, typical policy edit:  < 1 second

  If those cannot be hit, the change feed cannot feel live, and
  the change feed is where most of the product's daily value is.
```

### 3.8 Build vs borrow

Reimplementing AWS's evaluation engine from scratch is a multi-year trap. The hybrid:

```
  STATIC ANALYSIS (build)      fast, complete, offline
    → produces the graph, confidence 0.85–0.95

  PROVIDER VERIFICATION (borrow)  authoritative, rate-limited, costly
    SimulatePrincipalPolicy / checkAccess / testIamPermissions
    → ONLY for high-value edges: anything reaching a crown jewel,
      any escalation primitive, any cross-account edge
    → confirms or refutes; raises confidence to 0.99 or drops the edge

  FREE GROUND TRUTH (borrow)      no cost, high value
    AWS Access Analyzer external-access and unused-access findings
    IAM GetServiceLastAccessedDetails for the USED state
```

Verify only what a human will be shown. Simulating everything is impossible; simulating nothing leaves you unable to defend a finding when a cloud engineer says it is wrong.

---

## 4. Considerations

**Granted, effective and used are three different things.** Effective drives the graph. Used drives remediation. Granted is only for compliance evidence. Conflating the first two produces a graph where everything reaches everything.

**Cross-account is a per-service lookup, not a rule.** Whether a resource policy alone suffices differs by service. This is a table, and it must be content rather than code.

**Trust policies are the other half of `CAN_ASSUME`.** A principal with `sts:AssumeRole` on a role whose trust policy does not permit them has no edge. Preconditions matter as much as permissions.

**ABAC breaks the static assumption.** When access depends on tags, a tag change creates or destroys thousands of edges with no policy change and no audit event anyone watches. Model conditional capabilities as predicates and expand only for crown jewels.

**PIM and eligibility are real edges at reduced weight.** Treating "eligible for Global Admin" as no edge understates risk badly — an attacker simply activates.

**Confidence must be published, per edge.** A cloud engineer will challenge a finding. "Static analysis, cross-account, one side inferred, 0.70" is defensible; an unqualified assertion is not.

**Self-diagnostic signal.** When audit logs show a principal performing an action closure says is impossible, that is either a collection gap or an evaluation bug. Track the rate — it is the best available proxy for closure correctness.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Wrong precedence order | **Confidently wrong graph.** The worst outcome | Conformance suite with known-answer policy fixtures per cloud |
| Missing SCP or boundary | Over-permissive graph, false positives | Fail closed: if org-level constraints are uncollected, mark the subgraph reduced-confidence rather than assuming none |
| Conditions ignored | Paths asserted that do not exist | Satisfiability classification is mandatory on every conditional capability |
| Closure explosion | Memory exhaustion, cycle stall | Depth bound, confidence pruning, equivalence classes, pattern preservation |
| Trust policy not collected | `CAN_ASSUME` edges asserted without the other half | Precondition check required before edge creation |
| Provider API unavailable for verification | High-value edges stay at static confidence | Degrade gracefully; show the confidence, do not block |
| Policy language changes | Silent parse degradation | Version the evaluator; monitor unparsed-statement rate |

---

## 6. Contracts

```
  MUST GUARANTEE
    evaluation follows documented provider precedence exactly
    boundaries and SCPs never grant
    no CAN_ASSUME edge without a satisfied trust precondition
    every capability carries its granted_via and capped_by lineage
    every edge carries a confidence and its derivation method
    incremental recomputation produces the same result as a full run
```

---

## 7. Scope

```
  BUILD IN V1
    AWS evaluation with full precedence
    Entra + Azure RBAC including the elevateAccess bridge
    action grouping and resource pattern preservation
    bounded CAN_ASSUME closure with equivalence classes
    incremental recomputation
    provider verification for crown-jewel and escalation edges
    Access Analyzer ingestion as free ground truth

  DEFER
    GCP evaluation          (after AWS and Azure are correct)
    Kubernetes RBAC         (with the K8s connector)
    ABAC conditional expansion
    AD ACL closure          (needed for the AD connector — but AD is
                             a different evaluator, effectively its own
                             variant of this engine)
```

---

## 8. Example: Meridian, closure across 42 accounts

```
  INPUT, band 2, EDGE-CLD
    41 AWS accounts (one circuit-broken)
      8,400 roles · 2,100 users · 21,000 policies
      ~180,000 policy statements · 4 SCPs · 62 permission boundaries
      ~18 GB of policy documents

  STEP 1 — PARSE AND CANONICALISE
    statement ordering normalized, NotAction expanded,
    wildcards preserved, conditions extracted

  STEP 2 — EVALUATE PER PRINCIPAL
    for svc-devops-ai in account 123456789012:

      explicit denies       none
      SCP                   MeridianBaseline denies non-approved regions
                            → all capabilities scoped to 4 regions
      resource policies      2 bucket policies grant cross-account read
      permission boundary   none attached      ← noted, matters for E8
      identity policies      AdministratorAccess ("*" on "*")

      RESULT: 8,400 effective actions across 210 services,
              region-scoped by the SCP

  STEP 3 — ACTION GROUPING
    8,400 actions → 31 action groups → 9 capabilities:
      CAN_READ, CAN_WRITE, CAN_EXECUTE, CAN_DEPLOY, CAN_ASSUME,
      CAN_MODIFY, plus 3 escalation ingredients retained at
      action granularity (iam:PassRole, lambda:CreateFunction,
      lambda:InvokeFunction)

  STEP 4 — RESOURCE PATTERNS
    CAN_READ arn:aws:s3:::*              ← kept as a pattern
    expanded ONLY for the 47 crown-jewel datastores
    → 47 explicit CAN_READ edges + 1 pattern capability
    NOT 8,900 bucket edges

  STEP 5 — TRUST PRECONDITIONS
    svc-devops-ai CAN_ASSUME GHADeployRole?
      trust policy on GHADeployRole permits:
        - the GitHub OIDC provider, subject condition "repo:meridian/*"
        - arn:aws:iam::123456789012:role/svc-devops-ai
      → precondition satisfied. Edge created, confidence 0.97

    A different principal has sts:AssumeRole on 40 roles but appears
    in only 6 trust policies.
      → 34 edges NOT created. The permission exists; the capability
        does not.

  STEP 6 — CONDITION CLASSIFICATION
    "aws:SourceVpce": "vpce-0abc"     → CONDITIONAL
                                        (an attacker inside the VPC
                                         satisfies it) — weight reduced
    "aws:MultiFactorAuthPresent"      → HARD — weight heavily reduced
    "aws:PrincipalTag/team" = resource tag
                                      → ABAC. Conditional capability,
                                        4,120 current matches, expanded
                                        only for crown jewels

  STEP 7 — CHAIN CLOSURE
    CAN_ASSUME transitive closure, depth 6
    380 identical Terraform-deployed CloudWatch roles across accounts
      → collapsed to 1 equivalence class × 380

  OUTPUT
    ~2.1 million capability entries
    → ~310,000 graph edges after grouping and pattern preservation

  TIMING (EDGE-CLD, profile M)
    full closure          26 minutes
    incremental, one policy edit at 14:22 the next day:
      affected set = 1 principal + 11 who can assume it
      → 640 ms, 3 changed edges emitted

  VERIFICATION
    4,100 edges qualify as high-value (crown-jewel reaching,
    escalation-related, or cross-account).
    SimulatePrincipalPolicy called for each, rate-governed:
      4,038 confirmed  → confidence 0.99
        58 refuted     → EDGES REMOVED. Static analysis had
                         mis-evaluated a resource-policy interaction
                         on 3 S3 buckets.
         4 inconclusive → left at 0.93 with the reason recorded

    Those 58 removals are why verification is worth its cost: every
    one would have been a false path shown to an analyst.
```

### 8.1 What the closure found that no scanner reported

```
  svc-devops-ai
    granted     AdministratorAccess           2 lines of JSON
    effective   8,400 actions / 210 services  after SCP region scoping
    used (90d)  12 actions / 3 services       from CloudTrail

  → 8,388 unused actions, of which 41 are escalation ingredients
    and 6 reach crown jewels.

  This is the input to E8, and to the rightsizing recommendation
  that eventually resolves the AI Privilege Gap finding.
```

### 8.2 The Azure bridge, materialized

```
  Entra: 3 identities hold Global Administrator.

  E7 evaluates the elevateAccess path:
    Global Administrator
      → can toggle "Access management for Azure resources"
      → becomes User Access Administrator at the ROOT management group
      → Owner on all 18 subscriptions

  → 3 principals × 18 subscriptions = 54 synthesized CAN_ASSUME edges
    weight 0.95, confidence 0.98

  Without this single mechanism encoded, every Azure attack path at
  Meridian is understated, and the two Global Admins excluded from
  the MFA Conditional Access policy look like a directory hygiene
  issue rather than a route to every production subscription.
```

---

*Next: [Escalation Matcher](07-escalation-matcher.md)*

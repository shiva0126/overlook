# E8 — Escalation Matcher

**Series:** [Engine documentation](00-index.md) · **v1:** required

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

The Escalation Matcher **synthesizes edges that exist in no policy document.**

No policy anywhere says *"this principal can become an administrator."* It says `iam:CreatePolicyVersion`, and a human who knows AWS understands that this is game over. E8 is the encoded form of that knowledge: patterns which, when matched against a principal's effective capabilities and verified against real environmental preconditions, create a `CAN_ASSUME` edge nobody granted.

Without it, the path engine finds only what somebody deliberately granted — which is, by definition, the permissions nobody is worried about.

---

## 2. Position

```
  INPUT   capability sets from E7 (hard dependency — ordered)
          collected environmental state for precondition checks
          the escalation primitive catalog (shipped content)

  OUTPUT  synthesized relationship observations, each labelled
          with the primitive that produced it and a human-readable
          rationale

  FEEDS   E12 Graph Engine
```

E8 cannot run before E7. Everything else in the derivation stage is parallel; this one pair is ordered.

---

## 3. Mechanics

### 3.1 Primitives are content, not code

Shipped and versioned through the content pipeline, independently testable, independently rollback-able.

```yaml
primitive:
  id: aws.privesc.passrole_lambda
  version: 3
  provider: aws
  severity: critical

  requires:
    all_of:
      - action: "iam:PassRole"
        resource: "$TARGET_ROLE"
      - action: "lambda:CreateFunction"
      - any_of:
          - action: "lambda:InvokeFunction"
          - action: "lambda:CreateEventSourceMapping"

  preconditions:
    - "$TARGET_ROLE trust policy permits lambda.amazonaws.com"
    - "no SCP denies lambda:CreateFunction in this account"

  produces:
    predicate: CAN_ASSUME
    from: "$PRINCIPAL"
    to: "$TARGET_ROLE"
    weight: 0.85
    confidence: 0.92
    synthesized: true
    rationale: >
      Principal can create a Lambda function, pass $TARGET_ROLE to it,
      and invoke it — executing arbitrary code with that role's
      credentials.

  remediation:
    - "Scope iam:PassRole with a Condition on iam:PassedToService"
    - "Restrict the Resource of iam:PassRole to specific role ARNs"

  verification:
    method: provider_simulation
    actions: ["iam:PassRole", "lambda:CreateFunction"]
```

### 3.2 The matching loop

```
  for each principal:
    for each primitive applicable to its provider:
       1  CAPABILITY MATCH
          does the principal hold every required action?
          resolve $TARGET_ROLE from the resource pattern —
          a wildcard PassRole binds to EVERY role whose trust
          policy admits the service

       2  PRECONDITION CHECK           ← where correctness lives
          evaluate each precondition against COLLECTED STATE,
          not against assumption

       3  SYNTHESIZE
          emit the edge, labelled with primitive id and version

       4  VERIFY (high-value edges only)
          provider simulation confirms or refutes
```

### 3.3 Preconditions matter more than permissions

`iam:PassRole` + `lambda:CreateFunction` escalates **only if the target role's trust policy actually permits Lambda to assume it.** Skipping precondition checks is how these engines generate false positives at scale — and one confident false *"you have an admin escalation path"* costs more trust than ten missed findings.

### 3.4 Generalise, do not enumerate

```
  WRONG: 11 separate primitives for PassRole + each compute service

  RIGHT: iam:PassRole scoped to Resource "*" combined with ANY
         compute-creation permission escalates to EVERY role in the
         account whose trust policy admits that service.

  Model it that way: more accurate, and it produces far fewer,
  better findings instead of eleven near-duplicates.
```

### 3.5 Every synthesized edge must be interrogable

```jsonc
{
  "predicate": "CAN_ASSUME",
  "attributes": {
    "synthesized": true,
    "primitive_id": "aws.privesc.passrole_lambda",
    "primitive_version": 3,
    "rationale": "iam:PassRole(*) + lambda:CreateFunction + InvokeFunction",
    "preconditions_verified": ["trust policy admits lambda.amazonaws.com"]
  },
  "confidence": 0.92
}
```

Non-negotiable, for three reasons: an analyst will not believe an edge they cannot explain; a cloud engineer will demand to know why a read-only role is being called admin; and when a primitive turns out to be wrong, every edge it produced must be findable and retractable in one operation.

---

## 4. Considerations

**The catalog is the moat.** ~60 patterns across AWS, Azure, GCP, Kubernetes and AD. It is a living dataset, and maintaining it well is what separates an engine that finds real escalations from one that finds trivia (`../02 §7`).

**Severity is not the primitive's alone.** A primitive is `critical` in the abstract; the *finding* severity depends on what the target role reaches. E8 emits the edge; E12 and the path engine decide how much it matters.

**Cheap to run, expensive to get wrong.** Matching over capability sets is fast — the cost is entirely in precondition correctness and catalog quality.

**Versioning must support retraction.** When primitive v3 is found to be over-eager, every edge stamped `primitive_id + version` must be removable in one operation, and the affected findings withdrawn with an explanation.

**Not every primitive is a `CAN_ASSUME`.** Some produce `CAN_EXECUTE` (SSM SendCommand on an instance), some produce `CAN_READ` (secret access), some produce a `FINDING` with no edge at all (a wildcard trust policy is a start condition, not a relationship).

**ABAC turns tag-write into an escalation primitive.** In an environment using tag-based access control, `iam:TagRole` or `ec2:CreateTags` becomes an escalation path — but only where the tag is load-bearing. That dependency must be detected, not assumed.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Precondition unchecked | Confident false positive — the most damaging output this engine can produce | Preconditions mandatory in the schema; a primitive without them fails validation |
| Catalog gap | Silent false negative. A real escalation path is invisible | Catalog coverage tracked per provider; community and research input |
| Over-eager primitive | Many false edges at once | Version stamping enables bulk retraction; staged content rollout with canary |
| Target resolution too broad | Wildcard PassRole binding to roles whose trust policy excludes the service | Trust precondition check filters them |
| Runs before E7 completes | Matches against partial capabilities → both false positives and negatives | Hard ordering enforced by the derivation stage |
| Provider verification unavailable | Edges stay at static confidence | Degrade gracefully, show the confidence |

---

## 6. Contracts

```
  MUST GUARANTEE
    no edge synthesized without its preconditions verified against
      collected state
    every synthesized edge carries primitive id, version and rationale
    edges from a retracted primitive version are bulk-removable
    it never runs against an incomplete capability set
    catalog updates are versioned, staged and rollback-able
```

---

## 7. Scope

```
  BUILD IN V1
    the matching engine and primitive schema
    AWS catalog     (direct IAM manipulation, PassRole family,
                     modify-existing-compute, credential access,
                     org/trust)
    Azure catalog   (elevateAccess, Graph app permissions, ownership,
                     runCommand, Key Vault access policy)
    AD catalog      (ACL abuse, all three delegation types, DCSync,
                     GPO write, shadow credentials)
    precondition evaluation
    provider verification for the highest-severity matches

  DEFER
    GCP catalog          after AWS and Azure
    Kubernetes catalog   with the K8s connector
    AD CS templates      with the AD CS connector
    ABAC tag-write detection
```

---

## 8. Example: Meridian, 270 edges nobody granted

```
  INPUT
    capability sets from E7 for ~10,500 principals across
    41 AWS accounts, 18 Azure subscriptions and 2 AD forests

  RUN TIME  4 minutes on COL-CLD, 90 seconds on COL-DC1
```

### 8.1 AWS matches

```
  iam:PassRole(*) + lambda:CreateFunction + InvokeFunction     41 principals
    → for each, target resolution binds to every role whose trust
      policy admits lambda.amazonaws.com
    → account 123456789012 has 84 such roles
    → but 6 of the 41 principals have a permission boundary that
      excludes iam:PassRole → PRECONDITION FAILS → no edge
    → 35 principals × their eligible target roles = 892 edges

  iam:CreatePolicyVersion on an attached policy                  6
  ssm:SendCommand on instances with privileged roles            28
    → produces CAN_EXECUTE, not CAN_ASSUME
  secretsmanager:GetSecretValue on DB credential secrets        94
    → produces CAN_READ + an IDENTITY edge, because the secret
      authenticates as something
  s3:GetObject on the Terraform state bucket                    12
    → Meridian's state bucket contains RDS master passwords in
      plaintext. This is a credential-access edge, not a data edge.
  GitHub OIDC trust with subject condition "repo:meridian/*"      3   ← CRITICAL
    → this is a START CONDITION as well as an edge: any workflow
      in any Meridian repo can assume these roles
```

### 8.2 Azure matches

```
  Microsoft.Authorization/roleAssignments/write                   9
  virtualMachines/runCommand on VMs with managed identities     61
    → CAN_EXECUTE as the VM's managed identity
  Application.ReadWrite.All held by 2 service principals          2
    → can add credentials to ANY service principal → become it
  App registration OWNERS who are ordinary users                 11
    → ownership is not a role assignment and is missed by
      role-based reviews entirely
  Dynamic group whose membership rule reads a user-writable
    attribute, and which holds a role assignment                  1
    → any user can grant themselves that role by editing their
      own profile
```

### 8.3 AD matches — COL-DC1

```
  WriteDacl on the Domain Admins group, held by a service account
    → WriteDacl is transitive to full control
    → synthesize CAN_ASSUME on every Domain Admins member          1

  RBCD writable (msDS-AllowedToActOnBehalfOfOtherIdentity)
    + MachineAccountQuota = 10 (Meridian never changed the default)
    → any authenticated user can create a computer object and use
      it to take over the target
    → 8 computers, CRITICAL                                        8

  Unconstrained delegation on non-DC hosts                        12
    → CAN_ASSUME any identity that authenticates to them

  GetChanges + GetChangesAll (DCSync) on a non-DC principal        1
    → CAN_ASSUME the domain itself

  GPO write access on a policy linked to the server OU             2
    → CAN_EXECUTE on all 340 computers in that OU

  Shadow credentials: WriteProperty on msDS-KeyCredentialLink     19

  TOTAL AD                                                        43
```

### 8.4 What did NOT get synthesized, and why

```
  ✕ 214 principals hold iam:PassRole but no compute-creation
    permission                          → requirement not met

  ✕ 6 principals hold the full PassRole+Lambda combination but sit
    under a permission boundary that excludes PassRole
                                        → PRECONDITION FAILED

  ✕ 31 principals hold sts:AssumeRole on roles whose trust policies
    do not name them
                                        → PRECONDITION FAILED

  ✕ 40 constrained-delegation accounts whose msDS-AllowedToDelegateTo
    targets are all decommissioned hosts
                                        → target does not resolve

  Those 291 non-edges are the engine working correctly. Every one
  of them would have been a false "you have an escalation path"
  shown to a Meridian engineer who would then have stopped
  believing the product.
```

### 8.5 Verification

```
  Of 270 synthesized edges, 84 qualify as high-value —
  reaching a crown jewel or rated critical.

  SimulatePrincipalPolicy called for each, rate-governed:
     79 CONFIRMED     → confidence raised 0.92 → 0.99
      4 REFUTED       → EDGES REMOVED
                        (a resource-policy interaction the static
                         evaluator had mis-read)
      1 INCONCLUSIVE  → retained at 0.92, reason recorded
```

### 8.6 And the one that mattered

```
  PRIMITIVE   aws.oidc.subject_condition_too_broad
  MATCHED ON  role/GHADeployRole, account 123456789012

    trust policy condition:
      "token.actions.githubusercontent.com:sub": "repo:meridian/*"

    PRECONDITION CHECK
      does an OIDC provider for GitHub exist in this account?  ✓
      is the condition narrower than repo + ref?               ✗
      → matched

    SYNTHESIZED
      PIPELINE(any workflow in any meridian repo)
        CAN_ASSUME  ROL-ghadeploy
      confidence 0.95 · severity CRITICAL

  Combined with the PassRole+Lambda edge already synthesized from
  GHADeployRole to effective administrator, and E7's CAN_READ edge
  from there to prod-payments-db, this completes the path that
  started on Priya's laptop.

  Three of the four hops in Meridian's critical path are
  SYNTHESIZED edges. None of them appears in any policy document.
  None of them would be found by reading configuration, running a
  scanner, or searching a log.
```

---

*Next: [Posture, Correlation, Classification](08-posture-correlation-classification.md)*

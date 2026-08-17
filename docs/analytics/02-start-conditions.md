# Start Conditions — Where an Attacker Begins

**Series:** [Analytics](00-index.md)

---

## 1. Purpose

A path needs two ends. `01` defines the destination; this defines the origin.

A start condition is a node from which an attacker can plausibly begin, with an **exposure weight** expressing how plausible. Reverse BFS from crown jewels terminates when it reaches one — everything else is an intermediate hop.

Google SCC models exactly one attacker: **external, starting from the public internet.** That is a defensible simplification for a cloud-only product. It is the wrong model for a hybrid estate, where most real compromise begins with a person or a credential rather than a listening port.

---

## 2. Position

```
  INPUTS
    exposure data          internet-facing, from cloud + network
    vulnerability data     CVE + exploitability, as properties
    identity attributes    human, MFA state, privilege
    secret findings        leaked credentials and where they were found
    third-party trust      federation, OAuth apps, delegated admin
    AI entities            agents reachable by untrusted input

  OUTPUT
    a set of nodes marked as start conditions, each with a class
    and an exposure weight

  CONSUMED BY
    03 path engine    termination condition for reverse BFS
    04 scoring        the EXPOSURE factor
```

---

## 3. The six classes

```
  S1  INTERNET-EXPOSED AND EXPLOITABLE          weight 0.95
      reachable from the internet AND carrying a vulnerability
      with a known exploit
      → SCC's entire attacker model; ours is one of six

  S2  PHISHABLE HUMAN IDENTITY                  weight 0.70
      a human user with credentials to something
      modifiers:  MFA enrolled          × 0.45
                  privileged            × 1.20
                  excluded from CA      × 1.30
                  external / guest      × 1.15

  S3  EXPOSED CREDENTIAL                        weight 0.90
      a SECRET found in a repository, CI variable, container image,
      Terraform state, wiki or config file
      modifiers:  in public repo        × 1.30
                  never rotated         × 1.10
                  git history only      × 0.80

  S4  THIRD-PARTY AND SUPPLY CHAIN              weight 0.65
      OAuth apps with write scopes · GDAP partner relationships ·
      CI/CD OIDC trusts with loose conditions · MCP servers from
      unvetted packages · self-hosted runners

  S5  UNTRUSTED INPUT REACHING AN AI AGENT      weight 0.60   ← ours
      an AI_AGENT that can be influenced by content an attacker
      controls, and whose runs_as identity holds real privilege
      → no CVE, no misconfiguration, no credential theft

  S6  ALREADY COMPROMISED                       weight 1.00
      not a discovered condition — an ANALYST ASSERTION, used for
      blast radius during incident response (07 §2)
```

---

## 4. S5 in detail — the one nobody else models

```
  AN AGENT IS A START CONDITION WHEN THREE THINGS HOLD:

  1  IT CONSUMES CONTENT AN ATTACKER CAN INFLUENCE
       a RAG application retrieving from a source with external
       contributors · an agent reading tickets, email or public
       issues · an MCP tool returning attacker-controllable output ·
       a document store indexed from an untrusted origin

  2  ITS RUNS_AS IDENTITY HOLDS PRIVILEGE
       effective_privilege(runs_as) > NONE

  3  IT CAN ACT, NOT ONLY ANSWER
       INVOKES at least one tool with a write, execute or egress
       capability

  ALL THREE, or it is not a start condition. An agent that reads
  untrusted content and can only reply is a content-safety problem,
  not an exposure path.
```

**Why the weight is 0.60 rather than higher.** Indirect prompt injection is real and demonstrated, but it is less reliable than exploiting a CVE or using a stolen credential. The weight says: plausible, not certain. What makes these paths score highly is not the start weight — it is what sits at the other end, because agent identities are frequently over-privileged.

---

## 5. Exposure weight is not severity

```
  EXPOSURE WEIGHT answers: how easily can an attacker GET HERE?
  It says nothing about consequence. Consequence comes from the
  destination (01) and the traversal (03).

  A phishable human with no MFA is a HIGH-exposure start and may
  reach nothing. A hardened bastion is a LOW-exposure start and may
  reach everything.

  Both are needed, and multiplying them is the whole point (04).
```

---

## 6. Considerations

**Exposure is a property of the environment, not the node.** The same server is S1 in a DMZ and not a start condition at all in an isolated segment. This depends on network reachability data, which means without domain 05 connectors, S1 degrades to "has a public IP" — much cruder, and the Controller should say so.

**MFA is the single largest modifier and the most frequently stale.** Registered ≠ enforced. An identity with a registered method but excluded from Conditional Access is *more* exposed than one with none, because the organisation believes it is protected. The `× 1.30` for CA exclusion is deliberately above baseline.

**S3 needs the secret-to-identity bridge.** A credential in a repo is only a start condition if we know what it authenticates as (`../13-contracts.md §XI.8`). Without that link it is a hygiene finding, not a path origin.

**Do not let every human become a start condition.** 12,000 employees at S2 means 12,000 origins and a combinatorial explosion. Mitigations: cluster by equivalence (same groups, same access, same MFA state), and prune paths whose start-weight × traversal probability falls below threshold. In practice a few hundred distinct human start classes cover a 12,000-person estate.

**S6 is asserted, never derived.** An analyst marks a node compromised during an incident. It must be time-bounded and clearly temporary, or an incident from March silently inflates scores in August.

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| No network data | S1 degrades to "has a public IP" | Report the degradation in coverage; do not silently proceed |
| Every human is a start condition | Combinatorial explosion | Equivalence clustering plus weight-budget pruning |
| MFA state stale | Exposure weights wrong in both directions | Freshness threshold on identity attributes; degrade weight confidence |
| Secret with no identity link | S3 unusable | Secret-to-identity bridge is a hard dependency |
| S6 left set after an incident | Permanent score inflation | Mandatory TTL, default 7 days, visible countdown |
| S5 fires on read-only agents | Noise, and the class loses credibility | All three conditions required, not any |

---

## 8. Example: Meridian

```
  START CONDITIONS DISCOVERED                          total 1,847

  S1  internet-exposed and exploitable                        14
      12 EC2 instances with exploitable CVEs behind an ALB
       2 on-prem servers published through the Palo Alto

  S2  phishable humans (after equivalence clustering)        312
      from 12,000 identities → 312 distinct access classes
      of which  41 privileged
                 4 privileged AND excluded from the MFA CA policy
                   → weight 0.70 × 1.20 × 1.30 = 1.09, capped to 1.00

  S3  exposed credentials                                     94
      31 in MCP configs on endpoints        (agent)
      12 in Terraform state buckets          (aws.s3)
      28 in CI variables                     (github)
      23 in git history                      (github secret scanning)

  S4  third-party and supply chain                            37
       3 GitHub OIDC trusts with "repo:meridian/*"  ← also S1-adjacent
      11 OAuth apps with write scopes
      19 self-hosted runners
       4 MCP servers from unvetted npm packages

  S5  untrusted input reaching an AI agent                     8
      of 47 discovered agents, 8 satisfy all three conditions

  S6  already compromised                                      0
```

### 8.1 One S5, tested against the three conditions

```
  AI_AGENT  support-triage-agent

  1  CONSUMES ATTACKER-INFLUENCEABLE CONTENT?
       ✓ reads Jira issues, including those created by external
         reporters through the customer portal

  2  RUNS_AS HOLDS PRIVILEGE?
       ✓ svc-support-ai → CAN_READ on prod-customers-db
         (criticality 88), CAN_WRITE on the ticket store

  3  CAN ACT?
       ✓ INVOKES jira-mcp (write), email-send (external egress)

  ALL THREE → START CONDITION, weight 0.60

  AND THE PATH THAT FOLLOWED
    external reporter files a ticket containing hidden instructions
      → agent reads it via jira-mcp
      → agent invokes its database tool as svc-support-ai
      → reads customer records
      → sends them out via email-send

    No CVE. No misconfiguration. No credential theft.
    Score 79 — HIGH, not critical, because the start weight is 0.60
    and the destination is criticality 88 rather than 95.
```

### 8.2 And one that was correctly rejected

```
  AI_AGENT  docs-search-assistant

  1  consumes attacker-influenceable content?   ✓ public docs corpus
  2  runs_as holds privilege?                   ✓ read-only on the
                                                  docs index
  3  can act?                                   ✗ no write, no execute,
                                                  no egress tools

  → NOT a start condition.

  It would have been one under a rule of "any agent reading
  untrusted content." That rule would have produced 47 start
  conditions instead of 8, and the class would have been
  dismissed as noise within a week.
```

---

*Next: [Path engine](03-path-engine.md)*

# 11 — The Response Plane

**Series:** [The Edge Collector](00-index.md) · **LLD:** §4.3, §55–59, §75, §84

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. LLD §84 places
> the Response Engine in **Phase 3**. This document exists now because the
> plane's existence changes the collector's security posture from day one,
> and the trust boundaries have to be designed before anything is built
> against them.
> Budget: **shared with the control plane, 0.25 vCPU · 1 GB RAM.**

---

## 1. Purpose

Let SaaS cause something to happen inside the customer's environment —
quarantine a host, revoke a token, block a destination — without that capability
becoming the most attractive target in the product.

**This is the only plane where Overlook writes rather than reads.** Everything
else in the collector observes. This acts, and the blast radius of getting it
wrong is the customer's production estate.

---

## 2. Position

```
  INPUTS
    signed response commands from SaaS (LLD §57)
    agent registrations and heartbeats (LLD §55, §56)

  OUTPUTS
    executed actions, via cloud connectors or endpoint agents
    execution results → SaaS, over the same outbound session (§60)
    audit records → SQLite, append-only (LLD §54)

  COMPONENTS (LLD §4.3)
    Response Gateway · Authorization · Policy Validation ·
    Command Validation · Audit Logging · Agent Gateway
```

---

## 3. The trust inversion

Every other data path in the collector is **outbound and read-only**. This one
inverts both, and the consequences are worth stating before the mechanism.

```
  COLLECTION            we pull or receive. Worst case on
                        compromise: we see data we should not.

  RESPONSE              SaaS instructs; we execute with privilege
                        inside the customer's environment.
                        Worst case on compromise: an attacker who
                        controls the response path can quarantine
                        every endpoint at a bank, simultaneously.

  → THE RESPONSE PLANE IS A LATERAL MOVEMENT PATH INTO THE
    CUSTOMER, AND IT IS ONE WE BUILT AND SHIPPED.

  This is exactly the class of relationship Overlook's own product
  exists to find. A design that would score badly in our own graph
  should not ship.
```

LLD §75's *"No shell execution through SaaS"* is the single most important
control in the document, and it must not erode. The moment an action becomes
"run this command", the collector is a remote execution service with a security
product wrapped around it.

---

## 4. Command validation

LLD §58 defines the chain. Each step exists to defeat a specific attack.

```
  Verify command signature      forged commands
          ↓
  Verify tenant                 cross-tenant command injection
          ↓
  Verify Collector              a command for col-sg-01 replayed
                                at col-sg-02
          ↓
  Verify target                 the target exists, belongs to this
                                tenant, and is in scope for this
                                collector
          ↓
  Verify response policy        the CUSTOMER'S policy — see §5
          ↓
  Verify user authorization     the requesting operator was
                                entitled to ask
          ↓
  Check expiry                  a command captured today replayed
                                next month
          ↓
  Audit                         BEFORE execution, not after — §6
          ↓
  Execute
```

### 4.1 What LLD §57's command needs beyond what it has

```json
  {
    "command_id":   "rsp-29182",
    "action":       "quarantine_asset",
    "target":       { "type": "agent", "id": "agt-server-001" },
    "requested_by": "user-101",
    "reason":       "Suspected compromise",
    "expires_at":   "…",
    "signature":    "…",

    "nonce":            "…",        // PROPOSED
    "collector_id":     "col-sg-01",// PROPOSED
    "tenant_id":        "tenant-acme", // PROPOSED
    "approval":         { … },      // PROPOSED — §5.2
    "signature_key_id": "…"         // PROPOSED
  }
```

```
  nonce               LLD §75 requires replay protection. A signature
                      plus an expiry is not sufficient — a command is
                      replayable within its validity window. A
                      single-use nonce, tracked in SQLite until
                      expiry, is what makes replay protection real.

  collector_id
  + tenant_id         must be INSIDE the signed payload. If they are
                      only transport metadata, a valid command can be
                      redirected to another collector.

  signature_key_id    key rotation is impossible without it.
```

---

## 5. Response policy is the customer's, not ours

The most important design decision in this plane, and it is not in the LLD.

```
  LLD §58 has "Verify response policy". It does not say WHOSE.

  IT MUST BE THE CUSTOMER'S, HELD LOCALLY, AND NOT MODIFIABLE
  FROM SAAS.

  A policy that SaaS can change is not a control — it is a
  suggestion that the party being constrained is free to rewrite.
```

### 5.1 What the local policy governs

```
  ALLOWED ACTIONS      which of LLD §59's actions are permitted at
                       all. Most customers will enable a subset.

  SCOPE                which assets, accounts, subnets, tags.
                       "quarantine is permitted on workstations,
                        never on domain controllers or the payments
                        subnet"

  BLAST RADIUS CAP     the control the LLD is missing entirely.
                       max N targets per command
                       max N actions per hour, per action type
                       → a compromised SaaS cannot quarantine 12,000
                         endpoints. It can quarantine 5, and then it
                         is rate-limited and alarming.

  APPROVAL REQUIREMENT which actions need a human at the customer,
                       not only a requester at the MSSP — §5.2

  TIME WINDOWS         "no automated response during month-end
                       close" is a request every financial services
                       customer makes, and refusing it is how the
                       whole plane gets disabled.
```

### 5.2 Two-party approval for the destructive set

```
  LLD §57 has requested_by — ONE identity, at the MSSP.

  For actions that stop production, one MSSP operator should not be
  sufficient:

    quarantine an asset      ← stops a machine working
    disable a cloud access key ← can stop a pipeline
    apply a quarantine SG    ← can partition a service

  vs the reversible, low-blast-radius set:

    terminate a connection · kill a process ·
    disable a user session

  → the destructive set requires an approval object signed by a
    customer-side identity, and the collector verifies BOTH.

  This is also the answer to the MSSP's own liability question:
  "who authorized quarantining the trading desk at 09:29?" has a
  cryptographic answer rather than a ticket.
```

---

## 6. Audit before execution

LLD §58 places Audit immediately before Execute. That ordering is deliberate and
worth protecting.

```
  AUDIT-BEFORE      the intent is recorded even if execution
                    crashes, hangs or is interrupted. An action
                    that half-happened is visible.

  AUDIT-AFTER       an action that crashed mid-execution leaves no
                    trace. The endpoint is quarantined and nothing
                    says why.

  → write the intent, execute, write the outcome. TWO records,
    correlated by command_id.

  APPEND-ONLY AND TAMPER-EVIDENT (LLD §75). Hash-chained, with the
  chain head periodically anchored to SaaS — because the plausible
  attacker here is someone with access to the collector, and an
  audit log they can edit proves nothing.
```

---

## 7. The agent gateway

LLD §55 and §56. The registration flow is sound; two things need pinning down.

```
  REGISTRATION (LLD §55)
    Agent starts → bootstrap token → collector validation →
    certificate issued → identity registered → persistent mTLS

  ⚠ THE BOOTSTRAP TOKEN IS THE WEAK POINT.

  A shared bootstrap token pasted into a deployment script is a
  credential that will end up in a git repository — and per
  ../analytics/02 §3 that is precisely an S3 exposed-credential
  start condition. We would be manufacturing the finding.

  → single-use tokens, short TTL, scoped to an expected host or an
    expected count, issued from the control plane and audited.
```

```
  AGENTS ARE BOTH A TELEMETRY SOURCE AND A RESPONSE TARGET.
  The same mTLS session carries both.

  → the agent must distinguish them. A compromised collector should
    not be able to send an arbitrary action to an agent merely
    because the agent trusts the session.

  → response commands are verified BY THE AGENT against the SaaS
    signature, not merely accepted because they arrived on a
    trusted channel. The collector is a relay for the signature,
    not the source of authority.

  This is defence in depth, and it is what makes a collector
  compromise recoverable rather than total.
```

---

## 8. Considerations

**Ship it off by default.** The response plane should be disabled at install,
enabled per action, per scope, by the customer, after they have read what it
does. A security product that arrives with remote quarantine already enabled will
be found by the customer's own security review and it will not go well.

**Every action needs an inverse, and the inverse needs to be as easy.**
Quarantine implies unquarantine; disabling a key implies re-enabling it. An
operator who cannot undo an action at 03:00 will pull the network cable instead,
and the next conversation is about removing the collector.

**Capability discovery, not capability assumption.** LLD §59 correctly notes that
available actions depend on the connected agent or API. The collector must report
what it can actually do — a cloud connector with read-only credentials cannot
revoke a token, and SaaS must not offer a button that will fail.

**Rate limits belong here as much as at ingress.** LLD §39's rate limiting is
about telemetry volume. The response plane needs its own, and for the opposite
reason: not to protect the collector, but to protect the customer *from* the
collector.

**Do not let response state contaminate collection.** A quarantined asset is
still an asset and still generates telemetry. Response outcomes are facts about
what Overlook did, not observations about the estate, and they belong in a
separate stream so that an action never looks like a change the customer made.

**Phase 3 is the right place for this, and the trust boundaries are not.** The
signature scheme, the local policy model, the two-party approval and the agent's
independent verification all constrain the agent protocol and the SaaS command
API — both of which are built earlier. Designing them in Phase 3 means retrofitting
them into shipped interfaces.

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Shell execution added "just for one case" | The collector becomes a remote execution service | LLD §75 — absolute, no exceptions |
| No nonce | Commands replayable inside their validity window | Single-use nonce tracked to expiry |
| `collector_id` outside the signature | A valid command redirected to another collector | Inside the signed payload |
| Response policy modifiable from SaaS | The constrained party rewrites its own constraints | Local, customer-owned, §5 |
| No blast radius cap | A compromised SaaS quarantines 12,000 endpoints at once | Per-command and per-hour caps |
| One-party authorization for destructive actions | An MSSP operator can stop a customer's production alone | Two-party approval, §5.2 |
| Audit written after execution | A crashed action leaves no trace | Audit before, outcome after |
| Audit editable on the collector | An insider's actions are unprovable | Hash chain anchored to SaaS |
| Shared bootstrap token | A long-lived credential ends up in a git repository | Single-use, short TTL, scoped |
| Agent trusts the channel, not the signature | A collector compromise becomes an estate compromise | Agent verifies the SaaS signature |
| No inverse action | The operator pulls the cable instead | Every action has an easy undo |
| Enabled by default | Found by the customer's security review, badly | Off at install, opt-in per action |

---

## 10. Example: Meridian

### 10.1 A quarantine, end to end

```
  09:14  SaaS correlation raises a confirmed compromise on
         AST-lt-4471 (the laptop from 07 §9.1).
  09:16  an MSSP analyst requests quarantine.
  09:16  ⚠ quarantine_asset is in Meridian's DESTRUCTIVE set.
         The command is held pending customer approval.
  09:19  Meridian's on-call security engineer approves in their own
         console. A second signature is attached.

  09:19  the command reaches COL-mer-02 over the existing outbound
         session (LLD §60):

    signature        ✓ SaaS key rsp-2026-08
    tenant           ✓ tenant-meridian
    collector        ✓ col-mer-02, inside the signed payload
    nonce            ✓ unseen, recorded
    target           ✓ agt-lt-4471 registered here, this tenant
    policy           ✓ quarantine permitted on workstations
                     ✓ AST-lt-4471 is tagged workstation
                     ✓ NOT in the payments subnet exclusion
                     ✓ 1 target, cap is 5
                     ✓ 1 quarantine this hour, cap is 10
                     ✓ 09:19 is outside the month-end freeze
    approval         ✓ customer-side signature, eng-mday
    authorization    ✓ analyst-101 holds respond:quarantine
    expiry           ✓ 09:34, 15 minutes out

    AUDIT WRITTEN — intent, both identities, full policy evaluation

  09:19  relayed to the agent. THE AGENT VERIFIES THE SAAS
         SIGNATURE ITSELF and does not rely on the collector.
  09:20  quarantine applied. Outcome audited.

  09:20  a RESPONSE OUTCOME fact is emitted — on its own stream, so
         the network isolation that follows is not mistaken for a
         change Meridian's team made.
```

### 10.2 The blast radius cap earning its place

```
  A DRILL. Meridian's red team is given the SaaS analyst credential
  and asked to see how much damage it buys.

  09:00  they request quarantine on all 12,000 endpoints.

  09:00  SaaS accepts and signs — it is a valid request from a valid
         analyst.

  09:00  COL-mer-02 evaluates. Policy:
           max_targets_per_command  5
           → REJECTED. 12,000 > 5. Audited. Alarmed.

  09:02  they retry as 2,400 commands of 5 targets each.
           max_actions_per_hour  10
           → 10 execute. 2,390 rejected.
           → the customer's on-call is paged at command 11.

  DAMAGE: 50 endpoints, in under two minutes, with an alarm.
  WITHOUT THE CAP: 12,000 endpoints, and a bank stops trading.

  THE FINDING THAT WENT INTO THE REPORT
    the cap is the control that matters, not the signature. The
    signature was valid throughout — every step of LLD §58 passed
    except the one the LLD does not specify.
```

---

*Next: [The SaaS Side](12-saas-side.md)*

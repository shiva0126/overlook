# E14 — Privacy Gate

**Series:** [Engine documentation](00-index.md) · **v1:** Mode 2 only

---

## 1. Purpose

The Privacy Gate is the boundary. Everything that leaves the customer's environment passes through it, and nothing bypasses it.

It is the smallest engine in the appliance and the one with the most consequential failure mode: every other engine can be wrong and produce a bad finding; this one can be wrong and produce a breach.

It is also the mechanism behind the only competitive claim that survived the market survey — *residency is not blindness* (`../01 §2.3`). That claim is true because of what happens in these few hundred lines, or it is not true at all.

---

## 2. Position

```
  INPUT   Security Facts from E13, in plaintext canonical keys

  OUTPUT  Security Facts, tokenized, bucketed, stripped, validated
          → Sign + Queue → transport

  ALSO    quarantined facts that failed validation (never transmitted)
          the outbound inspection feed for the Controller

  DEPENDS ON
    the deployment_key           (customer KMS/HSM-wrapped)
    the privacy policy           (customer-configurable, versioned)
    the Security Fact schema     (validation target)
    the token map                (written here, read by resolve)
```

---

## 3. Mechanics

### 3.1 Four operations, in order

```
  1  TOKENIZE     replace identifying values with deterministic
                  pseudonyms
  2  BUCKET       replace precise numbers with ranges
  3  STRIP        remove every field not on the allow-list
  4  VALIDATE     confirm the result against the schema; FAIL CLOSED
```

Order matters. Validation last, so it verifies what will actually be transmitted rather than an intermediate form.

### 3.2 Tokenization

```
  token = TYPE_PREFIX + "-" +
          truncate( HMAC-SHA256( deployment_key, canonical_key ), 16 )

  e.g.  "email:priya.s@meridian.com"
        → IDN-9f3a7c21e845b0d6

  PROPERTIES
    deterministic   every Edge Node in the deployment produces the
                    same token for the same entity — which is what
                    makes the hybrid graph join at all
    irreversible    without the key, the console cannot recover the
                    canonical key
    tenant-isolated each deployment has an independent key, so a
                    key compromise is contained to one customer
    rotatable       with a dual-token transition window, painfully
```

**The key never reaches Overlook.** Generated at enrollment, wrapped by the customer's KMS/HSM, distributed to additional Edge Nodes through customer-controlled enrollment. If it ever transited us, the entire claim collapses (`../09 §2.2`).

### 3.3 The allow-list — and why it must not be a deny-list

```
  DENY-LIST         "remove these sensitive fields"
                    FAILS OPEN. A connector author adds a field;
                    it ships to the console; nobody notices.

  ALLOW-LIST        "only these fields may leave"
                    FAILS CLOSED. A new field is dropped until
                    someone deliberately permits it.

  This is the single most important implementation detail of the
  privacy claim. Everything else is mechanism; this is the posture.
```

Allow-lists are declared **per fact type**, versioned, and rendered in the Controller as a human-readable document the customer can export.

### 3.4 What is tokenized, cleared, bucketed, or never sent

```
  SENT IN CLEAR — carries no customer-identifying information,
  and correlation across deployments needs it
    CVE IDs · cloud provider and region names · public IPs of
    external services (chatgpt.com) · model names (gpt-4o) ·
    MCP tool names · misconfiguration rule IDs · port numbers ·
    protocol names

  TOKENIZED
    usernames · emails · hostnames · internal IPs · ARNs ·
    resource names · group names · repository names · file paths ·
    database names · bucket names · department names (customer's choice)

  BUCKETED
    record counts · data volumes · session durations · request
    rates · financial values
    → nobody decides differently at 4,187,332 records versus
      "1M-10M". Bucketing is a privacy control that costs almost
      no analytical value.

  NEVER SENT, IN ANY FORM
    raw log lines · file contents · prompt or response text ·
    query text · credential values · private keys · personal data ·
    exact record counts · precise geolocation
```

### 3.5 Fail closed

```
  A fact that fails schema validation is QUARANTINED LOCALLY,
  never transmitted, and surfaced in the Controller with the reason.

  If the deployment_key is unavailable — KMS unreachable, HSM
  offline — outbound HALTS ENTIRELY. Facts accumulate locally.
  Alarm raised.

  There is NO fallback to sending plaintext. Not degraded, not
  temporary, not "just this once."
```

### 3.6 The outbound inspection feed

E14 emits a record of everything that passed, in final form, alongside what it stripped. This drives the Controller's proof screen (`../05 §18`), which shows a before/after diff per fact.

That screen converts the privacy claim from a promise into something the customer verifies themselves — and it is worth more in a procurement review than any whitepaper.

---

## 4. Considerations

**Tokenization is only irreversible if the keyspace is large.** Email addresses are a small, guessable space; a leaked `deployment_key` makes every token dictionary-attackable. This is why the key is customer-held and per-deployment — the compromise is contained, and it requires compromising the customer, at which point the tokens are the least of their problems.

**Determinism across Edge Nodes is a hard requirement, not an optimisation.** Meridian's two nodes must produce identical tokens for Priya or the hybrid path does not exist. This is the same reason the canonical key rules are shipped configuration.

**Bucket boundaries are a policy decision with analytical consequences.** Too coarse and crown-jewel scoring degrades; too fine and the bucket is identifying. They belong in the customer-visible policy, not in code.

**Evidence references leave; evidence does not.** A `sha256` hash crosses the boundary. The 4 KB policy document it points at stays on the appliance, retrievable only through the Controller with RBAC and audit.

**Policy changes need a preview.** An operator changing the allow-list should see the effect on the last 24 hours of facts before committing — what would additionally be stripped, what console capability would be lost.

**Key rotation is real and must be designed, not deferred.** Rotation re-tokenizes the entire graph. A dual-token transition window — the appliance emits both old and new tokens for one sync cycle, the console merges — is the workable approach.

---

## 5. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Deny-list instead of allow-list | **Silent data leak** on every new connector field | Allow-list only; new fields dropped by default |
| Validation bypassed for "urgent" facts | Breach | No bypass path exists in the code |
| Key unavailable | Outbound halts | Halt is correct; alarm loudly; never fall back to plaintext |
| Non-deterministic tokenization | Graph splits between Edge Nodes, silently | Config hash asserted on sync; mismatch alarms |
| Bucket too fine | The bucket itself identifies | Bucket boundaries in customer-visible policy |
| Token map compromised | De-tokenization possible by whoever holds it | Separately encrypted, access-audited, on the appliance only |
| Policy change with no preview | Console capability silently lost | Preview against recent facts before apply |

---

## 6. Contracts

```
  MUST GUARANTEE
    no field leaves that is not on the allow-list for its fact type
    the deployment_key never transits Overlook infrastructure
    identical canonical keys produce identical tokens on every node
    validation failure quarantines; it never degrades to sending
    every transmitted fact is reproducible in the inspection feed
    key unavailability halts transmission rather than weakening it
```

---

## 7. Scope

```
  NOT IN V1 — Mode 1 sends nothing.

  BUT DESIGNED NOW:
    the allow-list schema per fact type
    the tokenization scheme and key custody model
    bucket definitions
    the outbound inspection record format

  Because Mode 2 is a switch, not a rewrite — and because the
  privacy policy schema is one of the eight contracts that must
  exist before anything else is built (../04 §29).
```

---

## 8. Example: Meridian, one fact crossing the boundary

```
  02:06 — E14 processes 2,914 facts from E13.
```

### 8.1 One fact, before and after

```
  IN — from E13, plaintext

    fact_type   RELATIONSHIP
    subject     canonical_key: "email:priya.s@meridian.com"
                display: "Priya S", dept: "Engineering"
    predicate   CAN_ASSUME
    object      canonical_key:
                "arn:aws:iam::123456789012:role/GHADeployRole"
                display: "GHADeployRole", account_name: "Production"
    attributes  mechanism: oidc_federation
                conditions: ["sub:repo:meridian/*"]
                privilege_level: ELEVATED
                synthesized: true
                primitive: aws.oidc.subject_condition_too_broad v2
    evidence    trust policy document, 4.2 KB, sha256:8a1f…c4d2
    confidence  0.95

  TOKENIZE
    "email:priya.s@meridian.com"
      → HMAC-SHA256(deployment_key, …) → IDN-9f3a7c21e845b0d6
    "arn:aws:iam::123456789012:role/GHADeployRole"
      → ROL-2b8e4f19a70c5d33

  BUCKET
    nothing numeric in this fact

  STRIP — allow-list for RELATIONSHIP permits:
    subject, predicate, object, mechanism, conditions,
    privilege_level, synthesized, primitive_id, primitive_version,
    confidence, first_seen, last_seen, observation_count,
    evidence_ref, sources

    → DROPPED: display names, dept, account_name, the 4.2 KB
      trust policy document itself
    → KEPT: the sha256 reference to it

  VALIDATE
    against overlook.fact.v1 → PASS

  OUT — what actually crosses the boundary

    { "fact_type": "RELATIONSHIP",
      "subject": {"type":"PIPELINE","token":"PIP-4c8d1e77a3b09f52"},
      "predicate": "CAN_ASSUME",
      "object":  {"type":"ROLE","token":"ROL-2b8e4f19a70c5d33"},
      "attributes": {
        "mechanism": "oidc_federation",
        "conditions": ["subject_condition_broad"],
        "privilege_level": "ELEVATED",
        "synthesized": true,
        "primitive_id": "aws.oidc.subject_condition_too_broad",
        "primitive_version": 2 },
      "confidence": 0.95,
      "evidence": {"ref": "sha256:8a1f…c4d2"},
      "first_seen": "2026-05-22T09:14:22Z",
      "last_seen":  "2026-08-14T01:47:11Z" }

    218 bytes. The console can compute an attack path from it.
    It cannot learn who Priya is, what the role is called, which
    account it lives in, or what the trust policy says.
```

### 8.2 A fact that did not pass

```
  Three facts quarantined this cycle.

  ✕ PROPERTY on DST-1c4b
      field "db_hostname": "prod-oracle-01.corp.meridian.com"
      → NOT on the allow-list for PROPERTY facts

    A connector author had added a hostname property to help with
    console display. Under a deny-list, it would have shipped.
    Under the allow-list, it was dropped and the fact quarantined
    because the schema validator flagged an unpermitted field.

    Controller shows:
      ✕ PROPERTY on DST-1c4b — field `db_hostname` not permitted
        [ view ]  [ add field to policy ]  [ discard ]

    The operator chose "discard." The hostname was never needed —
    the token identifies the datastore perfectly well.
```

### 8.3 The night the KMS was unreachable

```
  03:14  Meridian's KMS endpoint becomes unreachable during a
         network change.

  03:14  E14 cannot unwrap the deployment_key.
         → OUTBOUND HALTS. Zero facts transmitted.
         → facts accumulate in the local queue.
         → Controller raises a critical alarm.

  03:14–07:40  Collection, resolution, closure, escalation matching
         and the graph all continue normally. Mode 1 functionality
         is entirely unaffected — the Controller still shows
         findings and paths, in plaintext, locally.

  07:40  KMS reachable. Key unwrapped. Queue drains oldest-first,
         rate-limited. 4h 26m of facts transmitted in 90 seconds.
         Facts are idempotent, so nothing duplicated.

  WHAT DID NOT HAPPEN
    No fallback to sending untokenized facts.
    No "degraded privacy mode."
    No silent weakening.

    A four-hour delay in console freshness is an inconvenience.
    Four hours of plaintext on our servers would be the end of
    the product's only surviving claim.
```

### 8.4 The proof screen

```
  Meridian's CISO, during the quarterly review, opens
  Controller → Privacy → Outbound.

    OUTBOUND · last 24h · 11.8 MB · 2,914 facts · 3 quarantined

    02:04:11  RELATIONSHIP  CAN_ASSUME
      PIP-4c8d1e77a3b09f52 → ROL-2b8e4f19a70c5d33
      [ show what this was before the gate ▾ ]
        BEFORE   any workflow in github.com/meridian/*
                 → arn:aws:iam::123456789012:role/GHADeployRole
                 evidence: 4.2 KB trust policy document
        AFTER    PIP-4c8d… → ROL-2b8e…
                 evidence: sha256:8a1f…c4d2  (hash only)
        STRIPPED repo org name, ARN, account id, role name,
                 policy document

    [ Export for audit ]

  2.2 TB arrived that day. 11.8 MB left. The CISO can see, fact by
  fact, exactly what was withheld — and export it.
```

---

*Next: [Sign and Transport](12-sign-and-transport.md)*

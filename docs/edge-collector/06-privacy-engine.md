# 6 — The Privacy Engine

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 6. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **0.25 vCPU · 1 GB RAM.** The cheapest
> service in the collector and the one the product is sold on.

---

## 1. Purpose

Make every fact safe to leave the customer's environment, without making it
useless when it arrives.

Those two requirements pull against each other, and the whole design of this
service is the resolution: **destroy what has no analytical value, and make what
does have value unreadable but still joinable.**

---

## 2. Position

```
  INPUTS
    Security Facts from the Fact Engine (05)
    the tenant tokenization key, sealed, local (§5)
    the field classification policy (§4)

  OUTPUTS
    minimized facts → Metadata Forwarder (07)
    the token↔value map → Local Store, NEVER shipped (08 §3)

  CONSUMED BY
    07 metadata forwarder
    the Controller's de-tokenization endpoint (§6)

  ⚠ THE PERSISTENCE LINE
    Nothing downstream of this service holds plaintext. Nothing
    upstream can avoid it (04 §1).
```

---

## 3. Redaction is not tokenization

This is the one substantive disagreement between the service architecture as
handed over and this series, and it is worth stating plainly.

The handed-over design says `Remove raw payload │ Mask secrets │ Minimize PII`.
Masking and removal are **destructive**. That is safe, and it breaks the product
the moment there is more than one collector — which the 12/64/1 TB ceiling
guarantees, since Meridian needs four.

```
  DESTRUCTIVE — masking or dropping

    COL-mer-02 sees   john.smith@meridian.com   → j***@meridian.com
    COL-mer-03 sees   john.smith@meridian.com   → j***@meridian.com

    SaaS receives two masked strings. They happen to be equal, but
    only because the masking is naive — mask the local part of
    jane.smith@meridian.com and you get j***@meridian.com too.

    And against  arn:aws:iam::4471:user/jsmith  there is no
    correspondence at all.

    → the same person appears as three unrelated nodes
    → the identity graph fragments
    → attack paths that cross a collector boundary DO NOT EXIST
    → Meridian's flagship path — laptop MCP config (COL-mer-02)
      → GitHub token (COL-mer-03) → AWS role (COL-mer-03) —
      spans two collectors and would never be found

  DETERMINISTIC TOKENIZATION — HMAC

    token = base32( HMAC-SHA256( tenant_key, type || ":" || value )
                    truncated to 128 bits )

    john.smith@meridian.com  →  t_id_7QK3M9F2XB4NRWZ8
    always, on every collector, forever, for this tenant

    → SaaS joins on tokens without ever seeing a plaintext value
    → cross-collector resolution works exactly as if it had them
    → the graph is intact
    → SaaS holds no personal data, which is the compliance claim
```

**Cost of the change:** replace a mask function with an HMAC. At ~8 tokenizable
fields × 10,000 EPS that is 80,000 HMAC-SHA256 operations per second — roughly
1.5% of one core. It is the cheapest thing in this document and it decides
whether the graph works.

**Escalation E1.** This is the decision that must be made before the Fact Engine
schema is frozen, because it changes the shape of every identifier field in it.

---

## 4. Field classification

Four treatments. Every field in the common schema is assigned one, and the
assignment is content in the bundle, not code.

```
  ┌─ TOKENIZE ────────────────────────────────────────────────────┐
  │ identifiers that must JOIN across collectors and sources      │
  │                                                               │
  │   email · UPN · username · SID · ARN · hostname · FQDN ·      │
  │   private IP · MAC · device id · serial · repo name ·         │
  │   bucket name · database name · certificate subject ·         │
  │   agent id · principal id                                     │
  │                                                               │
  │ → HMAC, deterministic, tenant-keyed, type-prefixed            │
  └───────────────────────────────────────────────────────────────┘

  ┌─ DROP ────────────────────────────────────────────────────────┐
  │ things with no analytical value and catastrophic leak cost    │
  │                                                               │
  │   passwords · API keys · private keys · session cookies ·     │
  │   bearer tokens · connection strings · full command lines ·   │
  │   HTTP bodies · file contents · email bodies · SQL text ·     │
  │   THE ENTIRE RAW PAYLOAD                                      │
  │                                                               │
  │ → never tokenized. A token for a secret is a persistent       │
  │   oracle: an attacker who can submit guesses to the           │
  │   tokenizer can confirm a secret by comparing tokens.         │
  │   Secrets are DESTROYED.                                      │
  │                                                               │
  │ → the FACT that a secret was found IS retained:               │
  │   "a credential of type aws_access_key was found in           │
  │    <tokenized repo> at <path hash>, matching <tokenized       │
  │    principal>" — which is everything ../analytics/02 §3        │
  │   needs for an S3 start condition, with no secret in it       │
  └───────────────────────────────────────────────────────────────┘

  ┌─ PASS ────────────────────────────────────────────────────────┐
  │ structural, non-identifying, and needed for reasoning         │
  │                                                               │
  │   timestamps · event types · actions · ports · protocols ·    │
  │   outcomes · counts · durations · severities · confidence ·   │
  │   PUBLIC IPs · country · ASN · cloud region · resource type · │
  │   permission names · policy structure                         │
  └───────────────────────────────────────────────────────────────┘

  ┌─ GENERALIZE ──────────────────────────────────────────────────┐
  │ identifying in the specific, useful in the general            │
  │                                                               │
  │   exact geo coordinates    → country + ASN                    │
  │   exact file path          → depth + extension + a path hash  │
  │   exact record count       → a bucket: 1K–10K, 1M–10M         │
  │   full user agent          → browser family + major version   │
  └───────────────────────────────────────────────────────────────┘
```

```
  PUBLIC IPS PASS. PRIVATE IPS ARE TOKENIZED.

  A public IP is not customer data — it is internet infrastructure,
  and threat intel, ASN and geo correlation all need it in the clear.

  A private IP describes internal topology, which is exactly what an
  attacker wants and exactly what needs to join across collectors.
```

---

## 5. Key management

The key is what makes the privacy claim true, so where it lives is the claim.

```
  ONE KEY PER TENANT. NOT PER COLLECTOR.

  Per-collector keys would produce different tokens for the same
  value on COL-mer-02 and COL-mer-03, which is exactly the
  fragmentation of §3 with extra steps.

  GENERATION   at tenant onboarding, in the customer's environment
  STORAGE      sealed at rest — HSM, KMS, TPM, or an encrypted
               keystore whose passphrase is not on the collector
  DISTRIBUTION to sibling collectors over mTLS, direct,
               collector-to-collector — NEVER via SaaS
  ROTATION     see §5.1

  ⚠ SAAS NEVER HOLDS THE KEY. If it did, "SaaS cannot see
    customer data" would be a policy statement rather than a
    structural fact, and the entire differentiation would rest on
    a promise instead of on cryptography.
```

### 5.1 Rotation breaks every join, and must be designed for

```
  Rotating the key changes every token. Old facts and new facts no
  longer correlate. The graph splits at the rotation boundary.

  THEREFORE: EPOCHS.

    token = base32(HMAC(key_epoch_N, ...))   prefixed  t2_id_...

    on rotation
      1  a new epoch key is generated and distributed
      2  the collector emits, for every entity in its registry,
         a REMAP FACT:  { epoch_1_token, epoch_2_token }
      3  SaaS applies the remap to rewrite the graph
      4  the epoch-1 key is destroyed once remap is confirmed

    Meridian: 2.9M entities → 2.9M remap facts → ~90 MB.
    A one-off, hours, and the graph survives.

  ROTATION IS AN OPERATION, NOT A SETTING. It needs a runbook, it
  needs to be tested before the first customer, and it is the
  correct answer to "what happens if the key is compromised".
```

---

## 6. De-tokenization — the analyst still needs names

A graph of `t_id_7QK3M9F2XB4NRWZ8` is unusable. Resolution happens in the
**browser**, not in SaaS.

```
  1  the SaaS UI renders a path. Every node is a token.
  2  the browser collects the visible tokens — typically 20–200
  3  the browser calls the CUSTOMER'S OWN COLLECTOR directly,
     over mTLS, on the local network or via the customer's VPN:
       POST https://col-mer-01.meridian.internal/detokenize
       { tokens: [ ... ] }
  4  the collector checks the operator's authorization, looks the
     tokens up in the Local Store map, returns plaintext
  5  the browser substitutes the labels in the DOM

  SAAS NEVER SEES THE RESPONSE. The plaintext exists in SaaS's
  page, in the analyst's browser, on the customer's own network,
  and nowhere else.
```

```
  CONSEQUENCES THAT HAVE TO BE ACCEPTED

  · an analyst outside the customer's network sees tokens.
    For an MSSP SOC this is a real operational constraint and
    needs a deliberate answer — a per-tenant jump path, or a
    de-tokenization proxy the customer runs and authorizes.

  · every de-tokenization is logged, per operator, per token.
    That log is the customer's evidence of who looked at what,
    and it is a feature worth selling.

  · exports and PDFs are tokenized unless generated through a
    de-tokenizing client. This will surprise people. Say it in
    the onboarding documentation rather than in a support ticket.
```

---

## 7. Considerations

**Tokenize with a type prefix.** `t_id_`, `t_host_`, `t_arn_`. Without it, a
hostname and a username that happen to be the same string produce the same
token, and SaaS silently merges an asset with an identity. The prefix costs four
bytes and prevents a whole class of wrong merges.

**A token space is a rainbow-table target and the key is the only defence.**
Hostnames and usernames have low entropy — `DESKTOP-A4F91K` is guessable, and an
attacker who obtains the fact stream but not the key can still confirm guesses
if the HMAC key is weak or reused. 256-bit random key, per tenant, never
derived from anything guessable.

**The token↔value map is the most sensitive object in the collector.** It is a
complete inventory of the customer's estate with the mapping to make it
readable. Encrypted at rest, in the Local Store only, never in Redis, never in
a backup that leaves the environment, and never in the outbound spool.

**Free text is dropped, not scanned.** There will be pressure to run PII
detection over log messages and mask what is found. Do not — a PII detector that
is 99% accurate leaks 1% of everything, forever, and there is no way to know
which 1%. Free text carries no analytical value the structured fields do not
already have. Drop it, keep a bounded excerpt only where evidence requires it,
and let the evidence path (`08 §3`) be the controlled exception.

**Verify the invariant, do not assume it.** A test that asserts *no plaintext
identifier of any classified type appears anywhere in the outbound stream*
should run continuously against a sample of real outbound batches, not once at a
phase gate. The failure mode is a new field added to a parser that nobody
classified, and it will not announce itself.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Masking instead of tokenizing | Cross-collector paths do not exist; the graph fragments | Deterministic HMAC, §3 |
| Per-collector keys | Same fragmentation, harder to spot | One key per tenant, §5 |
| Key stored in or reachable from SaaS | The privacy claim becomes a promise | Sealed local storage, direct distribution |
| Secrets tokenized rather than dropped | A persistent confirmation oracle | Secrets are destroyed, §4 |
| No type prefix | Assets silently merged with identities | `t_<type>_` prefix |
| Weak or reused key | Low-entropy values brute-forced from the fact stream | 256-bit random, per tenant |
| Rotation without remap | The graph splits at the rotation boundary | Epochs + remap facts, §5.1 |
| Token map in Redis or in a backup | The whole estate leaks in a cache dump | Local Store only, encrypted |
| PII detection over free text | Silent, unquantifiable leakage | Drop free text |
| A new unclassified field | Plaintext ships unnoticed | Continuous outbound assertion, §7 |
| Analyst outside the network | The UI is unusable for a remote SOC | Answer it in the deployment design, not later |

---

## 9. Example: Meridian

### 9.1 A fact, before and after

```
  BEFORE — leaving the Fact Engine

  {
    type          RELATIONSHIP
    subject       { type: IDENTITY, id: "john.smith@meridian.com",
                    dept: "Finance", mfa: true, ca_excluded: true }
    predicate     AUTHENTICATES_TO
    object        { type: ASSET, id: "LT-4471.meridian.local",
                    ip: "10.4.22.81", os: "Windows 11",
                    owner: "CN=John Smith,OU=Finance,DC=meridian" }
    context       { method: "password", src_port: 51422,
                    raw: "<134>Aug 18 09:14:22 FortiEDR ..." }
    confidence    0.94
    coverage      { CON-fortiedr-01, 09:00–09:05, FULL }
  }

  AFTER — leaving the Privacy Engine

  {
    type          RELATIONSHIP
    subject       { type: IDENTITY, id: "t_id_7QK3M9F2XB4NRWZ8",
                    dept: "Finance", mfa: true, ca_excluded: true }
    predicate     AUTHENTICATES_TO
    object        { type: ASSET, id: "t_host_M2WV8DQ4KJ7YT3PL",
                    ip: "t_ip_B9XN4KF2QM8WR6ZT",
                    os: "Windows 11",
                    owner: "t_id_7QK3M9F2XB4NRWZ8" }   ← same token
    context       { method: "password" }
    confidence    0.94
    coverage      { CON-fortiedr-01, 09:00–09:05, FULL }
  }

  WHAT SURVIVED AND WHY
    dept, mfa, ca_excluded    non-identifying, and mfa+ca_excluded
                              are what make this an S2 start
                              condition with the ×1.30 modifier
    os                        needed for vulnerability correlation
    method                    needed to weight AUTHENTICATES_TO
    confidence, coverage      needed to score and to retract safely

  WHAT DIED
    src_port                  ephemeral, pure cardinality
    raw                       the entire payload

  AND NOTE
    the AD owner DN resolved to the SAME TOKEN as the subject.
    SaaS can now see that the person who authenticated owns the
    machine — without knowing who either of them is.
```

### 9.2 The cross-collector join that only tokenization allows

```
  COL-mer-02  (endpoint, CrowdStrike + FortiEDR + Scalefusion)
    finds an MCP configuration file on LT-4471 containing a
    GitHub personal access token

    FACT   FINDING · credential_in_config
           asset       t_host_M2WV8DQ4KJ7YT3PL
           principal   t_id_7QK3M9F2XB4NRWZ8
           cred_type   github_pat
           cred_ref    t_cred_R4TM8XQ2KP9WN7VB   ← the TOKEN's token,
                                                   never the token
  COL-mer-03  (cloud, AWS + GCP + GitHub + DLP)
    reads GitHub audit logs and sees that same PAT used to
    authenticate a workflow run

    FACT   RELATIONSHIP · AUTHENTICATES_TO
           subject     t_cred_R4TM8XQ2KP9WN7VB   ← IDENTICAL
           object      t_repo_L8KF3QN9XW2MTV4B

  AT SAAS
    the two tokens are byte-identical, so the join is trivial —
    an equality match, no fuzzy logic, no probabilistic merge

    → the flagship path connects:
        laptop MCP config  →  GitHub token  →  OIDC trust
        →  GHADeployRole  →  PassRole+Lambda  →  prod-payments-db

  WITH MASKING INSTEAD
    COL-mer-02 ships  gh***REDACTED***
    COL-mer-03 ships  gh***REDACTED***
    → not joinable — every redacted GitHub PAT in the estate
      produces the same string, so a join would connect this
      laptop to all 340 of them
    → the path is either invisible or catastrophically wrong

  THIS PATH IS THE PRODUCT DEMO. It requires tokenization, and
  it requires the key to be one key for the tenant.
```

### 9.3 What a SaaS-side breach would yield

```
  An attacker obtains a full dump of Overlook SaaS for Meridian.

  THEY GET
    2.9M opaque tokens and the edges between them
    a graph shape: 47 crown jewels, 31,400 paths, 3 reachable
    metadata: departments, OS versions, MFA states, timestamps
    that the shape is a mid-size financial services company

  THEY DO NOT GET
    a single username, hostname, IP address, ARN, repository
    name, bucket name or database name
    any credential, any log line, any file content
    any means of reversing a token — the key is in Meridian's
    datacentre and was never transmitted

  THE HONEST RESIDUAL RISK
    the graph SHAPE is itself sensitive. Knowing that a
    Finance-department laptop reaches a PCI database in six hops
    is useful to an attacker even without the names. Tokenization
    is a very strong control, not a complete one, and this is the
    right way to describe it to a customer — because the first
    security architect who reads the design will notice, and
    having said it first is worth more than the claim.
```

---

*Next: [Metadata Forwarder](07-metadata-forwarder.md)*

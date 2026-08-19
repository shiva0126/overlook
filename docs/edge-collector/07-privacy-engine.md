# 7 — The Privacy Engine

**Series:** [The Edge Collector](00-index.md) · **LLD:** §26, §27, §1, §88

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. **This document
> carries the collector's most consequential escalation** —
> [ESC-3](13-escalations.md), that LLD §26's *"hash identifiers where
> configured"* is not a privacy control.
> Budget: **0.25 vCPU · 1 GB RAM** — the cheapest module in the collector
> and the one the product is sold on.

---

## 1. Purpose

Make every fact safe to leave the customer's environment without making it
useless when it arrives.

Those two requirements pull against each other, and this module is the
resolution: **destroy what has no analytical value, and make what does have value
unreadable but still joinable.**

LLD §1 and §88 both state the goal. §26 is where the mechanism has to earn it.

---

## 2. Position

```
  INPUTS
    Security Facts, Entities, Relationships (06)
    the privacy policy (LLD §7 internal/privacy/policy.go)
    the tenant key — PROPOSED, does not exist in the LLD (§5)

  OUTPUTS
    minimized objects → Forwarder (08)
    token↔value map → SQLite, NEVER shipped (09 §3)

  CONSUMED BY
    08 forwarder
    the local API's de-tokenization endpoint (§6) — PROPOSED

  ⚠ THE PERSISTENCE LINE
    Nothing downstream holds plaintext. Nothing upstream can avoid
    it (05 §1).
```

---

## 3. What the LLD gets right

LLD §27's secret handling is correct and complete, and should not be weakened:

```
  password · passwd · secret · token · access_token ·
  refresh_token · authorization · api_key · private_key ·
  session_token · cookie · credential

  "Values should never be forwarded unless explicitly allowed by
   policy."                                            — LLD §27
```

```
  SECRETS ARE DESTROYED, NEVER TOKENIZED.

  A token for a secret is a persistent ORACLE: an attacker able to
  submit guesses to the tokenizer can confirm a secret by comparing
  tokens. Destruction is the only correct treatment, and §27 has it.

  THE FACT THAT A SECRET WAS FOUND IS RETAINED:
    "a credential of type aws_access_key was found in <repo> at
     <path hash>, matching <principal>"
  — everything ../analytics/02 needs for an exposed-credential
  start condition, with no secret in it.
```

LLD §26's *remove raw payload* is also right, and it is what makes the collector
a privacy boundary rather than a relay.

---

## 4. ⚠ Hashing is not tokenization

**ESC-3.** LLD §26 says *"Hash identifiers where configured."* Three problems,
in ascending order of seriousness.

### 4.1 An unkeyed hash of a low-entropy value is reversible

```
  hostnames    DESKTOP-A4F91K · PROD-WEB-01 · LT-4471
  usernames    jsmith · john.smith · svc-etl-nightly
  emails       first.last@meridian.com — the domain is known and an
               organisation's name list is often public
  private IPs  10.0.0.0/8 is 16,777,216 values

  SHA-256 OF 16.7 MILLION IPs IS A SUB-SECOND COMPUTATION.

  Hashing an RFC1918 address provides no confidentiality whatsoever.
  Hashing a hostname or a corporate email is barely better.
```

### 4.2 "Where configured" means off by default

The shipped behaviour is plaintext identifiers leaving the customer environment,
which contradicts LLD §1 and §88. Privacy that must be switched on is privacy
most customers will not have.

### 4.3 There is no way back — the UI shows hashes forever

The LLD specifies no de-tokenization path anywhere. With §26 enabled, the SaaS
UI renders `a3f9c2...` for every identity, asset and resource, and there is no
specified mechanism to resolve them. **The feature as written makes the product
unusable rather than private.**

### 4.4 And it breaks the graph across collectors

The consequence that is easiest to miss and most expensive.

```
  Meridian has FOUR collectors, because LLD §71's ceiling forces
  scale-out. 412,006 entities appear on more than one of them.

  MASKING or a per-collector salt
    the same person produces different values on COL-mer-02 and
    COL-mer-03 → three unrelated nodes → the identity graph
    fragments → CROSS-COLLECTOR ATTACK PATHS DO NOT EXIST

  Meridian's flagship path spans three collectors:
    laptop MCP config (COL-mer-02) → GitHub token (COL-mer-03)
    → OIDC trust → GHADeployRole → prod-payments-db

  Under LLD §26 as written it is not discoverable.
```

### 4.5 The proposed mechanism

```
  token = base32( HMAC-SHA256( tenant_key, type || ":" || value ) )[:128]
  prefixed by type:   t_id_ · t_host_ · t_ip_ · t_arn_ · t_repo_

  john.smith@meridian.com  →  t_id_7QK3M9F2XB4NRWZ8
  always, on every collector, forever, for this tenant

  DETERMINISTIC   SaaS joins by BYTE EQUALITY across collectors
  IRREVERSIBLE    without the key, enumeration is useless
  KEYED           the key never leaves the customer environment
                  and is never sent to SaaS

  COST: identical to the SHA-256 §26 already specifies.
        ~8 fields × 10,000 EPS = 80,000 HMAC/sec ≈ 1.5% of one core.
```

**The type prefix matters.** Without it a hostname and a username that happen to
be the same string produce the same token, and SaaS silently merges an asset with
an identity. Four bytes prevents a whole class of wrong merge.

---

## 5. Field classification

Four treatments. Every field in the LLD §21 schema and every `ext.` field
(`04 §5`) is assigned one, and the assignment is content in the signed bundle,
not code.

```
  ┌─ TOKENIZE ─ identifiers that must JOIN across collectors ──────┐
  │  email · UPN · username · SID · ARN · hostname · FQDN ·        │
  │  PRIVATE IP · MAC · device id · serial · repo · bucket ·       │
  │  database name · cloud.resource_id · agent id · cert subject   │
  └────────────────────────────────────────────────────────────────┘

  ┌─ DROP ─ no analytical value, catastrophic leak cost ───────────┐
  │  everything in LLD §27 · full command lines · HTTP bodies ·    │
  │  file contents · email bodies · SQL text · THE RAW PAYLOAD ·   │
  │  free-text log messages                                        │
  └────────────────────────────────────────────────────────────────┘

  ┌─ PASS ─ structural, non-identifying, needed to reason ─────────┐
  │  timestamps · event.* · ports · protocols · outcomes · counts ·│
  │  severity · confidence · PUBLIC IPs · country · ASN ·          │
  │  cloud.provider · cloud.region · resource type ·               │
  │  permission names · policy structure                           │
  └────────────────────────────────────────────────────────────────┘

  ┌─ GENERALIZE ─ identifying in the specific, useful in general ──┐
  │  exact geo → country + ASN                                     │
  │  exact file path → depth + extension + path hash               │
  │  exact record count → a bucket (1K–10K, 1M–10M)                │
  │  full user agent → browser family + major version              │
  └────────────────────────────────────────────────────────────────┘
```

```
  PUBLIC IPs PASS. PRIVATE IPs ARE TOKENIZED.

  A public IP is internet infrastructure, not customer data, and
  threat intel, ASN and geo correlation all need it in the clear.
  A private IP describes internal topology — exactly what an
  attacker wants and exactly what must join across collectors.
```

**Free text is dropped, not scanned.** There will be pressure to run PII
detection over log messages and mask what is found. A detector that is 99%
accurate leaks 1% of everything, forever, and there is no way to know which 1%.
Free text carries no analytical value the structured fields do not already have.

---

## 6. Key management and de-tokenization

**All of this is PROPOSED — none of it exists in the LLD, and ESC-3 is not
implementable without it.**

### 6.1 The key

```
  ONE KEY PER TENANT. NOT PER COLLECTOR.
  Per-collector keys reproduce the fragmentation of §4.4 exactly.

  GENERATION    at tenant onboarding, inside the customer environment
  STORAGE       the encrypted keystore LLD §7 and §8 already specify
                (internal/crypto/keystore.go), sealed at rest
  DISTRIBUTION  collector-to-collector over mTLS — NEVER via SaaS
  ROTATION      §6.3

  ⚠ SAAS NEVER HOLDS THE KEY. If it did, "SaaS cannot see customer
    data" would be a policy statement rather than a structural fact,
    and the differentiation would rest on a promise instead of on
    cryptography.
```

### 6.2 De-tokenization happens in the browser

```
  1  the SaaS UI renders a path. Every node is a token.
  2  the browser collects visible tokens — typically 20–200
  3  the browser calls the CUSTOMER'S OWN COLLECTOR, over mTLS,
     on the local network or via the customer's VPN:
       POST https://col-sg-01.acme.internal:8443/api/v1/detokenize
     (a new route on the LLD §46 local API)
  4  the collector authorizes the operator, looks the tokens up in
     SQLite, returns plaintext
  5  the browser substitutes labels in the DOM

  SAAS NEVER SEES THE RESPONSE. Plaintext exists in the analyst's
  browser, on the customer's own network, and nowhere else.
```

```
  CONSEQUENCES THAT MUST BE ACCEPTED

  · an analyst OUTSIDE the customer network sees tokens. For an
    MSSP SOC this is a real operational constraint needing a
    deliberate answer — a per-tenant jump path, or a customer-run
    de-tokenizing proxy.

  · every de-tokenization is logged per operator per token, into
    LLD §54's audit trail. That log is the customer's evidence of
    who looked at what, and it is worth selling.

  · exports and PDFs are tokenized unless generated through a
    de-tokenizing client. Say this in onboarding, not in a support
    ticket.
```

### 6.3 Rotation breaks every join

```
  Rotating the key changes every token. Old and new facts no longer
  correlate. The graph splits at the rotation boundary.

  → EPOCHS.  t2_id_… carries the epoch in the prefix.

    on rotation
      1  generate and distribute the new epoch key
      2  emit a REMAP FACT per entity: { epoch1_token, epoch2_token }
      3  SaaS applies the remap to rewrite the graph
      4  destroy the epoch-1 key once remap is confirmed

    Meridian: 2.9M entities → 2.9M remap facts → ~90 MB. Hours.

  ROTATION IS AN OPERATION, NOT A SETTING. It needs a runbook, it
  must be tested before the first customer, and it is the answer to
  "what happens if the key is compromised".
```

---

## 7. Considerations

**The token map is the most sensitive object in the collector.** It is a complete
inventory of the customer's estate with the mapping to make it readable.
Encrypted at rest, in SQLite only, never in a cache, never in the spool, never in
a backup that leaves the environment (`09 §3.3`).

**A weak key defeats the whole mechanism.** Low-entropy values mean an attacker
with the fact stream can confirm guesses if the key is guessable or reused. 256
bits, random, per tenant, generated in a key ceremony rather than derived from a
tenant name.

**Verify the invariant continuously, not at a gate.** A test asserting *no
plaintext identifier of any classified type appears anywhere in the outbound
stream*, run against a sample of real outbound batches on every build. The
failure mode is a new `ext.` field nobody classified, and it will not announce
itself.

**Privacy runs after the Fact Engine, and that is correct.** Enrichment needs
plaintext (`05 §1`), and facts are built from enriched events. The cost is that
the aggregation windows in `06 §4` hold plaintext in memory — which is acceptable
because they are memory-resident and short-lived, and unacceptable if they are
ever checkpointed to disk unencrypted (`06 §8` says every 30 s; that checkpoint
must be encrypted).

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Unkeyed hash of identifiers | Reversible in seconds; no confidentiality | ESC-3 — HMAC with a tenant key |
| Hashing off by default | Plaintext identifiers ship; contradicts LLD §1 | On by default |
| No de-tokenization path | The UI shows hashes forever; the product is unusable | Browser → collector endpoint, §6.2 |
| Per-collector key or salt | Cross-collector paths do not exist | One key per tenant |
| Key reachable from SaaS | The privacy claim becomes a promise | Sealed local keystore, direct distribution |
| Secrets tokenized rather than dropped | A persistent confirmation oracle | LLD §27 — destroy |
| No type prefix on tokens | Assets silently merged with identities | `t_<type>_` |
| Weak or reused key | Low-entropy values brute-forced from the fact stream | 256-bit random per tenant |
| Rotation without remap | The graph splits at the boundary | Epochs + remap facts, §6.3 |
| Token map in the spool or a backup | The whole estate leaks in one file | SQLite only, encrypted |
| PII detection over free text | Silent, unquantifiable leakage | Drop free text |
| A new unclassified `ext.` field | Plaintext ships unnoticed | Continuous outbound assertion, §7 |
| Fact-window checkpoint unencrypted | Plaintext at rest, bypassing this module entirely | Encrypt the checkpoint |

---

## 9. Example: Meridian

### 9.1 One relationship, before and after

```
  BEFORE — leaving the Fact Engine

    subject    { type: identity, id: "john.smith@meridian.com",
                 dept: "Finance", mfa: true, ca_excluded: true }
    relation   AUTHENTICATES_TO
    target     { type: asset, id: "LT-4471.meridian.local",
                 ip: "10.4.22.81", os: "Windows 11",
                 owner: "CN=John Smith,OU=Finance,DC=meridian" }
    context    { method: "password", src_port: 51422,
                 raw: "<134>Aug 18 09:14:22 FortiEDR ..." }
    confidence 0.94

  AFTER — leaving the Privacy Engine

    subject    { type: identity, id: "t_id_7QK3M9F2XB4NRWZ8",
                 dept: "Finance", mfa: true, ca_excluded: true }
    relation   AUTHENTICATES_TO
    target     { type: asset, id: "t_host_M2WV8DQ4KJ7YT3PL",
                 ip: "t_ip_B9XN4KF2QM8WR6ZT",
                 os: "Windows 11",
                 owner: "t_id_7QK3M9F2XB4NRWZ8" }   ← SAME TOKEN
    context    { method: "password" }
    confidence 0.94

  SURVIVED  dept, mfa, ca_excluded — non-identifying, and the last
            two are what make this a start condition with a ×1.30
            modifier · os for vulnerability correlation · method to
            weight the edge · confidence to score it

  DIED      src_port (ephemeral, pure cardinality) · raw (everything)

  AND NOTE  the AD owner DN resolved to the SAME TOKEN as the
            subject. SaaS can now see that the person who
            authenticated owns the machine — without knowing who
            either of them is.
```

### 9.2 The cross-collector join

```
  COL-mer-02  finds an MCP config on LT-4471 containing a GitHub PAT
    FINDING  credential_in_config
             asset      t_host_M2WV8DQ4KJ7YT3PL
             principal  t_id_7QK3M9F2XB4NRWZ8
             cred_type  github_pat
             cred_ref   t_cred_R4TM8XQ2KP9WN7VB
                        ↑ a token OF the credential. The credential
                          itself was destroyed per LLD §27.

  COL-mer-03  reads GitHub audit logs; that PAT authenticates a run
    RELATIONSHIP  AUTHENTICATES_TO
             subject    t_cred_R4TM8XQ2KP9WN7VB   ← IDENTICAL
             target     t_repo_L8KF3QN9XW2MTV4B

  AT SAAS   byte-identical tokens. An equality match. No fuzzy
            logic, no probabilistic merge, no confidence penalty.

  → the flagship path connects across two collectors.

  WITH MASKING INSTEAD
    both ship gh***REDACTED*** — the same string every redacted PAT
    in the estate produces. A join would connect this laptop to all
    340 of them. The path is either invisible or catastrophically
    wrong.
```

### 9.3 What a SaaS breach would yield

```
  An attacker takes a full dump of Overlook SaaS for Meridian.

  THEY GET
    2.9M opaque tokens and the edges between them
    the graph SHAPE: 47 crown jewels, 31,400 paths, 3 reachable
    metadata: departments, OS versions, MFA states, timestamps

  THEY DO NOT GET
    one username, hostname, IP, ARN, repository, bucket or database
    any credential, any log line, any file content
    any means of reversing a token — the key is in Meridian's
    datacentre and was never transmitted

  THE HONEST RESIDUAL
    the graph shape is itself sensitive. Knowing a Finance laptop
    reaches a PCI database in six hops is useful even without names.

    Tokenization is a very strong control, not a complete one, and
    that is how it should be described to a customer — because the
    first security architect who reads the design will notice, and
    having said it first is worth more than the claim.
```

---

*Next: [Forwarder](08-forwarder.md)*

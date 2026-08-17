# Sign, Queue and Transport

**Series:** [Engine documentation](00-index.md) · **v1:** Mode 2 only

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

The last stage. Tokenized facts are made tamper-evident, made durable, batched, compressed and delivered to the MSSP console over a channel the customer's firewall team will approve without a security review.

It is plumbing rather than an engine — it makes no decisions about content — but three of its properties are load-bearing for the whole architecture: **outbound-only**, **durable before delivery**, and **idempotent on replay**.

---

## 2. Position

```
  INPUT   tokenized, validated facts from E14

  OUTPUT  signed batches delivered to the console
          acknowledgements consumed, queue pruned

  ALSO    the command channel — the only way the console can ask
          the collector to do anything
```

---

## 3. Mechanics

### 3.1 Sign the batch, hash each fact

```
  per-fact Ed25519 signatures are wasteful at volume.

  INSTEAD
    each fact carries its own SHA-256 content hash
    the batch carries a manifest of those hashes
    the MANIFEST is signed, once, with the Edge Collector's Ed25519 key

  → per-fact integrity verification, one signature per batch
  → the console can verify any individual fact against the manifest
```

The signing key is generated on the collector at enrollment, backed by TPM/HSM where available, and never leaves.

### 3.2 The outbound queue

```
  durable segmented files, ordered, encrypted at rest
  facts appended after signing, pruned only after the console acks

  BATCHING
    flush on 1,000 facts, or 5 MB, or 60 seconds — whichever first
    (bounded latency AND bounded overhead)

  COMPRESSION
    zstd over the newline-delimited batch body
```

### 3.3 Transport

```
  TLS 1.3, mutual authentication, port 443, OUTBOUND ONLY.
  The collector always initiates. The console NEVER connects in.

  BATCH FORMAT
    header  deployment_id, collector_id, batch_id, schema_version,
            fact_count, compression, sig_alg
    body    zstd(newline-delimited facts)
    footer  manifest signature, SHA-384 of the body

  ACK
    per-fact accept/reject with reasons. Rejected facts are
    quarantined locally with the reason and surfaced in the
    Controller. They are NOT retried blindly forever.
```

**Outbound-only is a procurement feature, not just a security one.** "You do not need to open a port for us" removes a firewall change request and a security-architecture review that can take months in an organisation like Meridian.

### 3.4 Offline behaviour

```
  console unreachable
    → queue grows on local disk
    → default capacity: 7 days at expected fact rate, or 20% of disk

    at 60%   warn in the Controller
    at 80%   warn + notify the operator
    at 95%   shed EVENT_SUMMARY facts first
    at 98%   shed PROPERTY refreshes, keep first-observations
    at 100%  keep only FINDING and RELATIONSHIP

    RELATIONSHIP is never shed. It is the graph.

  on reconnect
    drain oldest-first, rate-limited so the backlog does not
    overwhelm ingest. Facts are idempotent, so replay is safe.
```

Everything else keeps working during an outage: collection, resolution, closure, the local graph, the Controller. Only console freshness and response are affected, and the UI says so specifically rather than showing a generic error.

### 3.5 The command channel

Since the console cannot connect inbound, control flows through a polled queue.

```
  Edge polls  GET /v1/commands?after=<cursor>  every 30s
              long-poll, 25s hold, for near-real-time response

  COMMAND TYPES
    CONFIG_UPDATE     connector settings, policy, retention
    CONTENT_UPDATE    parsers, primitives, rules, fingerprints
    RESPONSE_REQUEST  a response action for local validation
    DIAGNOSTIC        collect redacted diagnostics
    UPGRADE           staged collector upgrade

  EVERY command is signed by the console with a key the collector
  pinned at enrollment, carries a nonce and an expiry, and is
  validated locally before execution.

  LOCAL POLICY MAY REFUSE ANY COMMAND CLASS OUTRIGHT, and the
  console cannot override that.
```

### 3.6 Enrollment

```
  1  operator requests a new Edge Collector in the console
     → one-time enrollment token, 24h TTL, single-use, bound to
       the deployment and a chosen node name

  2  operator runs the installer, pastes the token

  3  collector generates an Ed25519 keypair — private key generated
     in and never leaving the node

  4  collector presents the token + a CSR over TLS

  5  console validates, issues a client certificate bound to
     (deployment_id, collector_id), 90-day lifetime

  6  the deployment_key is obtained:
       FIRST node   generates locally, wraps with customer KMS
       LATER nodes  fetch from the customer's KMS using their own
                    cloud identity, OR operator-mediated transfer
       NEVER through Overlook

  7  certificates auto-renew at 2/3 lifetime over the established
     channel; revocation checked on every connection
```

Step 6 is the one that must not be shortcut. It is the only step where a convenience would let Overlook see the key.

---

## 4. Considerations

**Idempotency is the transport's contract with the console.** After a partition the collector resends. Semantic identity plus upsert on the console side makes replay harmless; without it, every outage produces duplicate edges.

**Rejected facts must not retry forever.** A schema-version mismatch or a validation failure on the console side is permanent until something changes. Quarantine, surface, stop — do not build an infinite retry loop that masks the problem.

**Certificate expiry is a silent killer.** Ninety-day certs with auto-renewal at sixty days works until the collector has been offline for thirty-one. Surface expiry countdown in the Controller and in the fleet plane.

**Clock skew breaks nonces and expiries.** Monitor drift explicitly; a command rejected for being "expired" because the collector clock is wrong is a confusing failure.

**Queue sizing should be expressed in days, not bytes.** "7 days of buffer" is meaningful to an operator; "40 GB" is not, because they do not know the fact rate.

**Shedding order is a policy decision worth showing.** An operator should be able to see what would be shed first, before it happens.

---

## 5. Failure modes

| Failure | Behaviour |
|---|---|
| Console unreachable | Queue, warn at thresholds, shed by class at 95%+. Everything else continues |
| Queue full | Shed lowest-value classes; RELATIONSHIP never shed; alarm |
| Certificate expired | Connection refused. Loud, specific error. Renewal path documented in the Controller |
| Clock drift | Nonce and expiry validation fail. Monitor drift as a first-class metric |
| Facts rejected by the console | Quarantine with the reason, surface, do not retry blindly |
| Signature verification fails at the console | Batch rejected entirely, alarm on both sides — this indicates key compromise or corruption |
| Partial batch delivery | Batch is atomic; either fully acked or fully retried |
| Enrollment token leaked | Single-use and 24h TTL limit exposure; revocation available |

---

## 6. Contracts

```
  MUST GUARANTEE
    the collector always initiates; no inbound port is ever required
    facts are durable before delivery is attempted
    replay is idempotent
    every batch is signed and verifiable per fact
    the deployment_key never transits this channel
    local policy can refuse any command class, unconditionally
```

---

## 7. Scope

```
  NOT IN V1 — Mode 1 has no console to talk to.

  BUT DESIGNED NOW:
    the batch and manifest format
    the enrollment flow and key custody
    the command envelope and local refusal model

  Because Mode 2 is a switch, and because enrollment determines
  key custody — the one thing that cannot be retrofitted.
```

---

## 8. Example: Meridian, one day on the wire

```
  BOTH EDGE NODES → MSSP CONSOLE

  02:07  COL-CLD flushes the nightly batch
           1,842 facts · 7.1 MB after zstd
           manifest of 1,842 SHA-256 hashes, signed Ed25519
           mTLS 443 outbound, 3.2 seconds
           ACK: 1,842 accepted, 0 rejected

  02:08  COL-DC1 flushes
           1,072 facts · 4.7 MB
           ACK: 1,071 accepted, 1 REJECTED
             reason: schema_version overlook.fact.v1 field
             `primitive_version` expects integer, received string
           → quarantined locally with the reason
           → Controller attention item raised
           → a content-pipeline bug, fixed in the next bundle.
             NOT retried in a loop.

  THROUGHOUT THE DAY
    incremental emissions every 60s where changes exist
    most flushes are empty — a stable environment produces no
    traffic, which is the emission policy working

    total for the day: 2,914 facts · 11.8 MB
```

### 8.1 The four-hour outage

```
  14:22  Meridian's egress proxy is reconfigured. The console
         becomes unreachable from both Edge Collectors.

  14:22  queue begins growing. Controller shows:

         ⚠ NOT CONNECTED TO OVERLOOK CONSOLE — 0h 02m
           Still working: collection, parsing, resolution, closure,
             the local graph, evidence, all configuration
           Unavailable: cross-node correlation, console freshness,
             response actions, content updates
           Queue: 2% of 7-day capacity

  18:41  proxy fixed. Reconnect.
         queue drains oldest-first, rate-limited to 500 facts/s so
         the console ingest is not overwhelmed.
         1,204 facts delivered in 4 seconds.

         Six facts had been emitted twice — once before the outage
         was detected, once on drain. The console upserted on
         semantic identity. No duplicates in the graph.

  WHAT MERIDIAN LOST
    four hours of console freshness.

  WHAT MERIDIAN DID NOT LOSE
    any collection, any analysis, any local capability, or any data.
```

### 8.2 A command, refused

```
  09:14  An MSSP analyst, investigating the critical path, clicks
         "Quarantine host LT-4471" in the console.

  09:14  console constructs a RESPONSE_REQUEST:
           action, target token, ttl 4h, nonce, expiry,
           issued_by, approved_by, justification
         signs it, places it in the command queue

  09:14  COL-DC1 long-poll returns the command.
         Local validation:
           signature against the pinned console key    ✓
           nonce unused                                ✓
           not expired                                 ✓
           LOCAL POLICY: is RESPONSE_REQUEST enabled?  ✗

         Meridian deployed Overlook read-only. The response
         master switch is OFF.

         → command REFUSED. Not executed. Not queued for later.
         → refusal logged in the local audit trail, and reported
           to the console so the analyst sees why.

         Console shows: "Refused by the customer's Edge Collector —
         response actions are disabled by local policy."

  The console issued a valid, signed, approved command.
  The collector said no. That is the design working exactly as
  intended: the customer's local policy beats the vendor's cloud,
  always.
```

---

*End of the engine series. Back to the [index](00-index.md).*

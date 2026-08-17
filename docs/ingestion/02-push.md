# Push Ingestion

**Series:** [Ingestion](00-index.md) · **Sources:** webhooks · event grids · cloud event notifications

---

## 1. What it is

The source calls us. A single event arrives unsolicited over HTTPS, and **if we drop it, it is gone** — most webhook senders retry a handful of times and then give up permanently.

Low volume, high value density, and the strictest durability contract in the system.

---

## 2. How a delivery works, step by step

```
  1  RECEIVE
     TLS termination. Per-source path routing:
       POST /webhook/github/<instance-id>
       POST /webhook/gitlab/<instance-id>
     The path identifies the source before any parsing.

  2  VERIFY SIGNATURE          ← BEFORE ANYTHING ELSE
     HMAC over the raw body with the per-source secret, or mTLS
     client certificate.
     Fail → 401, and DO NOT journal. An unauthenticated delivery
     is not ours to keep.
     ⚠ verify over the RAW BYTES, before any JSON parsing. Parsing
       first and re-serialising to verify is how signature checks
       get quietly broken.

  3  REPLAY GUARD
     delivery-ID seen-set + timestamp window.
     Already seen → return 200 immediately, do not journal twice.
     Outside the window → 400, log it.

  4  JOURNAL
     Append the raw body plus the provenance envelope.

  5  FSYNC
     Wait for it. This is the whole contract.

  6  RETURN 200
     ONLY NOW is the event ours.

  7  HAND TO PIPELINE
     Stage 2 onwards, asynchronously.
```

---

## 3. The ack contract, and the bug it prevents

```
  Returning 200 before fsync is the classic data-loss bug in this
  class.

  The sender treats 200 as "delivered, safe to forget."
  If we crash between the 200 and the durable write, the event is
  gone and NOBODY KNOWS — the sender has no reason to retry and we
  have no record that anything was missed.

  So: fsync, then 200. Always. Even when it is slow.
```

**When fsync is slow, hold the connection.** A 340 ms ack is fine; an early ack is not. Webhook senders tolerate latency far better than they tolerate silent loss, and none of them will tell you about the loss.

---

## 4. Signature verification

```
  HMAC (most common)
    secret per source instance, from the credential broker
    computed over the RAW REQUEST BODY
    constant-time comparison — a timing-variable compare on a
    signature is a real, if unglamorous, vulnerability

  mTLS (preferred where the source supports it)
    client certificate pinned at configuration time
    no shared secret to rotate

  WHAT VERIFICATION IS FOR
    it is not integrity checking — TLS already does that.
    It is AUTHENTICATION: proving the delivery came from the
    configured source and not from anyone who discovered the URL.
    A webhook endpoint is, by construction, internet-reachable in
    many deployments.
```

---

## 5. Replay protection

```
  TWO MECHANISMS, BOTH NEEDED

  DELIVERY-ID SEEN-SET
    most senders include a unique delivery identifier.
    Keep a bounded set — last 24 hours, or last N per source.
    A repeat means the sender retried because our ack was lost.
    → return 200 immediately. IDEMPOTENT, not an error.

  TIMESTAMP WINDOW
    the signature covers a timestamp; reject deliveries outside
    ±5 minutes.
    → stops a captured delivery being replayed later by an
      attacker who recorded it.

  Neither alone is sufficient. The seen-set handles honest retries;
  the window handles malicious replay.
```

---

## 6. Backpressure

```
  The journal is behind, or the pipeline is saturated.

  RESPOND 503 with Retry-After.
  Do NOT accept and buffer in memory — an in-memory buffer that
  outlives the process is a lie about durability.

  Most senders honour Retry-After. Those that do not will retry
  on their own schedule, which is why the replay guard exists.

  ⚠ 503 is a real signal to the sender. GitHub, for example, will
    disable a webhook after repeated failures. Backpressure must
    be rare and short, or the source disables itself and nobody
    notices until a coverage gap appears.
```

---

## 7. Coverage semantics

```
  PUSH NEVER EMITS A COVERAGE WINDOW.

  A stream of events cannot prove it saw everything — "I did not
  receive it" is indistinguishable from "it was not sent."

  So push sources can only ADD to the graph. They can never
  retract, and they can never be the reason something is
  tombstoned.

  This is why webhooks complement, never replace, the pull
  collector for the same source. GitHub webhooks tell us a repo
  became public within seconds; the GitHub pull collector is what
  authorises removing a repo that no longer exists.
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| 200 before fsync | Silent, permanent data loss | fsync then ack, always |
| Signature over parsed body | Verification quietly broken | Verify over raw bytes |
| Non-constant-time compare | Signature forgeable by timing | Constant-time comparison |
| No replay guard | Duplicate processing, or malicious replay | Seen-set plus timestamp window |
| In-memory buffer under pressure | Durability claim is false | 503 with Retry-After instead |
| Sustained 503 | Source disables the webhook | Backpressure must be rare; alert if it recurs |
| Unbounded seen-set | Memory growth | Bounded by time or count per source |
| Coverage window emitted | Wrongful tombstoning | Structurally impossible for this class |

---

## 9. Considerations

**A webhook endpoint is an attack surface.** It is reachable, it accepts POST bodies, and it is the only inbound path with a body worth parsing. Signature verification before parsing is not defence in depth — it is the defence.

**Per-source paths, not one endpoint.** A single `/webhook` receiving everything means the parser must guess the source before it can verify anything. Path-based routing establishes identity first.

**Webhooks are a latency optimisation, not a completeness mechanism.** They tell us about a change in seconds instead of hours. The pull collector remains the source of truth and the only thing that can emit a coverage window.

**Secrets rotate.** Support two active secrets per source during rotation, accepting either, so a rotation does not drop deliveries.

---

## 10. Example: Meridian, one hour of GitHub webhooks

```
  09:00-10:00, EDGE-CLD, 3 GitHub organisations

  41 deliveries received on
     POST /webhook/github/meridian-eng
     POST /webhook/github/meridian-data
     POST /webhook/github/meridian-platform

  BREAKDOWN
    39 verified, journaled, fsync'd, 200 returned
     2 duplicate delivery IDs — GitHub retried because an earlier
       ack was slow
       → replay guard matched → 200 returned immediately
       → NOT journaled twice
     0 signature failures

  ONE SLOW ACK
    09:41  a delivery arrives while the disk is briefly saturated
           by the nightly Parquet compaction
           fsync takes 340 ms
           → connection held, ack returned late
           → GitHub is content; the delivery is durable

    Had we acked early and crashed, that repository-visibility
    change would have been lost with no record that anything was
    missing.

  WHAT THEY CARRIED
    12  push events           → pipeline definitions changed
     9  repository events     → one visibility change, public
     8  member events         → collaborator added
     7  secret_scanning_alert → 2 new alerts
     5  workflow_run          → OIDC token minted

  THE ONE THAT MATTERED
    09:14  repository.public on meridian-eng/internal-tooling
           → within 90 seconds, the graph had the repo marked
             public, E9 raised a finding, and the change feed
             recorded it with attribution
           → the GitHub pull collector would have caught it at
             13:00, on its 4-hourly cycle
           → 3 hours 46 minutes earlier, because of a webhook
```

---

*Next: [Stream ingestion](03-stream.md)*

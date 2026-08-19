# 1 — The Ingestion Gateway

**Series:** [The Edge Collector](00-index.md) · **LLD:** §12, §13, §36–39, §61

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary and takes
> precedence over this document. Divergences are recorded in
> [13-escalations.md](13-escalations.md).
> Budget: **1.5 vCPU · 2 GB RAM** (`00 §4`).

---

## 1. Purpose

The single front door. Everything the customer's estate sends or offers arrives
here, and the gateway's job is to get it into `OVERLOOK_RAW` — or refuse it in a
way the sender understands.

It is the only module with an externally reachable attack surface, and the only
one whose failure is visible to the customer's own infrastructure.

---

## 2. Position

```
  INPUTS
    the thirteen collection methods of LLD §12

  OUTPUTS
    envelope-wrapped records (LLD §13) → overlook.raw.<type>
    an ACK, a 503, or silence — per class, per §4
    per-connector health and rate telemetry → LLD §50, §52

  CONSUMED BY
    02 durable event buffer
    10 control plane, for the connector health view
```

---

## 3. Thirteen methods, four recoverability contracts

LLD §12 lists thirteen collection methods. They collapse into **four durability
contracts**, and that collapse is the only structural variation in the collector
— everything else that looks like a difference between sources is a parser or a
field mapping.

```
  PULL      we fetch, on a checkpoint (LLD §10 PollingConnector, §44)
            METHODS   REST API polling · cloud API · database polling ·
                      object storage · message queue · file ingestion
            LOSS      none. The checkpoint is not advanced until the
                      batch is durable in OVERLOOK_RAW.
            BACKPRESSURE  stop calling. Trivial, lossless.

  PUSH      the source sends and waits for an acknowledgement
            METHODS   REST API push · webhook · syslog TCP · syslog TLS
            LOSS      IRRECOVERABLE if we ack and then lose it.
                      → fsync BEFORE ack. Always. See §3.1.
            BACKPRESSURE  withhold the ack; the sender retries.

  STREAM    the source fires and forgets
            METHODS   syslog UDP · streaming API
            LOSS      silent, unless counted. See §4.3.
            BACKPRESSURE  NONE EXISTS.

  AGENT     the source holds its own buffer
            METHODS   agent telemetry (LLD §55)
            LOSS      only if the agent's own buffer overflows first
            BACKPRESSURE  send a hold signal; the agent is built for it.
```

### 3.1 The ack rule

```
  THE INGRESS ACK IS NOT RELEASED UNTIL JETSTREAM CONFIRMS THE WRITE.

  This single rule is what makes the PUSH contract real. It is also the
  thing most likely to be quietly broken during optimisation, because
  acking early makes every benchmark look better and nothing fails
  until a power cut.

  Assert it in the performance gate, not only in review.
```

---

## 4. Flow control

LLD §36 lists the inputs to flow control; §37 defines four pressure levels; §38
defines five event priorities. This section is how those three compose, because
individually none of them says what actually happens to a packet.

### 4.1 Pressure levels (LLD §37)

```
  GREEN    queue < 50% · CPU < 70% · disk < 70%    normal
  YELLOW   queue 50–75%                            scale workers up (LLD §17)
  ORANGE   queue 75–90%                            throttle low priority
  RED      queue > 90%                             protect P0/P1 only
```

### 4.2 What each ingress class does at each level

This is the table the LLD does not have, and without it "throttle low priority"
is not implementable — UDP cannot be throttled, only dropped.

```
  LEVEL    PULL              PUSH               AGENT           STREAM
  ─────────────────────────────────────────────────────────────────────────
  GREEN    normal            normal             normal          normal
  YELLOW   P4 poll interval  normal             normal          normal
           ×2
  ORANGE   P3/P4 polling     delay ack 200 ms   hold signal     ⚠ P4 dropped
           suspended         for P3/P4          for P3/P4         and counted
  RED      all polling       503 + Retry-After   hold all        ⚠ P2/P3/P4
           suspended         for P2/P3/P4        below P2          dropped
                             P0/P1 still acked                    P0/P1 kept
                                                                  where the
                                                                  source is
                                                                  distinguishable

  DATA LOSS      none         none               none            YES
```

**Priority is only usable at ingress where it can be determined from the source.**
LLD §13's envelope carries `priority`, and for PULL and AGENT it is known per
connector. For syslog it is *not* known until parsing, which is downstream of
the buffer. In practice a syslog listener sheds by connector — which is a
per-port, per-source-IP decision, and it must be configured with that limitation
stated rather than assumed to be per-event.

### 4.3 STREAM loss must be counted, not absorbed

The most important paragraph in this document.

```
  A DROPPED UDP PACKET IS INDISTINGUISHABLE FROM A QUIET NETWORK.

  If 40,000 firewall events are dropped and nothing records it:
      the firewall looks quiet
      fewer connections are observed
      fewer relationships are asserted
      the customer's exposure score IMPROVES

  Measure silence directly (extending LLD §50):

    overlook_events_received_total{connector}
    overlook_events_dropped_total{connector,reason}
        reason ∈ buffer_full · rate_limited · malformed · oversize
    overlook_source_gap_seconds{connector}

  PROPOSED — each drop should also shorten the connector's COVERAGE
  WINDOW, so SaaS does not read the gap as evidence that nothing
  happened. This is escalation ESC-5.
```

---

## 5. Authentication, per class

LLD §75 requires mTLS and TLS 1.3. That is achievable for three classes and
impossible for one.

```
  PULL      outbound only. No inbound surface. Credentials come from
            the vault by reference (LLD §11, §45) — never inline.

  PUSH      webhook       HMAC signature over the body, per source
            syslog TLS    port 6514, mTLS where the sender supports it
            syslog TCP    port 514 — plaintext, no authentication
            file drop     filesystem permissions on a watched directory

  STREAM    ⚠ NONE IS POSSIBLE. syslog/UDP cannot be authenticated,
            and source-IP allowlisting is weak because spoofing UDP
            is trivial.
            → THE NETWORK PATH IS THE CONTROL. This belongs in the
              deployment guide as a requirement, not an assumption.

  AGENT     mTLS with per-agent certificates issued at enrolment
            (LLD §55), rotated on schedule.
```

**The STREAM gap cannot be engineered away at the gateway.** An attacker on the
customer's network can inject syslog. Mitigations are network segmentation, and
carrying lower confidence on STREAM-derived facts (`06 §7`).

---

## 6. The event envelope

LLD §13 defines it. Three additions are proposed, each because the information
exists only here and cannot be reconstructed later.

```json
{
  "event_id":       "evt-019289",
  "tenant_id":      "tenant-acme",
  "collector_id":   "col-sg-01",
  "connector_id":   "aws-prod",
  "connector_type": "aws",
  "received_at":    "2026-08-18T10:30:10Z",
  "format":         "json",
  "priority":       1,
  "payload":        {},

  "ingress_class":  "PULL",          // PROPOSED — §3
  "source_addr":    "10.4.1.20",     // PROPOSED — provenance for STREAM
  "sequence":       184471           // PROPOSED — per-connector monotonic
}
```

```
  ingress_class    the recoverability contract, needed by every
                   downstream backpressure and confidence decision

  source_addr      the only provenance an unauthenticated syslog
                   record has

  sequence         gaps detected at SaaS reveal loss without
                   trusting any single hop
```

```
  received_at IS NOT event_time.

  The event carries its own timestamp, which the parser extracts
  (03 §7) and normalization maps to `timestamp` (LLD §21). The two
  routinely disagree by hours — batched exports, buffered agents,
  wrong timezones, unsynchronised clocks.

  Both are carried the whole way. Collapsing them here destroys the
  only evidence that a source's clock is wrong.
```

---

## 7. Rate limiting

LLD §39 requires limits per connector, source IP, API, agent, tenant and event
type, with token buckets.

```
  THE ONE THAT MATTERS MOST IS PER CONNECTOR.

  One firewall left in debug mode will otherwise consume the budget
  of forty other sources. The limit should derive from the
  connector's declared expected rate plus a headroom multiple —
  typically 3× — rather than from a global default, because
  "normal" for a FortiGate and for an Entra audit poll differ by
  four orders of magnitude.

  A rate-limited record is DROPPED AND COUNTED, never silently
  discarded, and it is a `reason` value in §4.3's counter.
```

---

## 8. Considerations

**The gateway must be the least clever module in the collector.** It has the only
external attack surface and it is in the path of every byte. Parsing and any
content-dependent logic belong downstream, behind the buffer. A gateway that
understands log formats is a gateway that can be crashed by a malformed log —
and in a modular monolith (LLD §5) that takes everything else with it.

**Validation is structural only.** Size caps, encoding, framing, envelope
completeness. Not schema, not content. A record failing structural validation is
dropped and counted; a record that is merely strange goes to the parser and, if
necessary, to dead-letter (LLD §20).

**Recover at the boundary.** LLD §5's monolith means a panic here kills
collection entirely. Every listener goroutine needs `recover()`, and a panic must
dead-letter one record rather than propagate.

**The 503 must carry Retry-After.** Without it, well-behaved HTTP senders retry
immediately and backpressure becomes a self-inflicted denial of service.

**File ingestion needs move-on-complete.** A file appearing in a watched
directory is not a file that has finished being written. The contract is: the
sender writes `.tmp`, then renames. The gateway reads only completed names.

**Ports are a deployment negotiation, not a default.** LLD §61 lists 514, 6514,
8443, 9443. In a regulated environment each one is a firewall change request.
Ship the list in the pre-deployment questionnaire, not in the installer.

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Ack before JetStream confirms | PUSH data lost on crash, silently | Ack released only on publish-ack; asserted in the gate |
| STREAM drops uncounted | Coverage looks complete; score improves on blindness | §4.3 counters + ESC-5 |
| Priority assumed available for syslog | "Throttle P4" silently does nothing | Shed by connector; state the limitation |
| Binary accept/reject | Sawtooth, thundering herd | Graduated pressure levels, §4.2 |
| Global rate limit | One noisy source starves forty | Per-connector token buckets, §7 |
| `received_at` used as `event_time` | Clock skew invisible; ordering wrong | Both carried end to end |
| Panic in a listener | Whole collector dies (monolith) | `recover()` at every boundary |
| Gateway parses content | Malformed input crashes the front door | Structural validation only |
| 503 without Retry-After | Self-inflicted DoS under load | Mandatory header |
| File read before write completes | Truncated records, silent corruption | Move-on-complete |
| UDP source spoofing | Injected facts | Segmentation + lower confidence on STREAM |

---

## 10. Example: Meridian

### 10.1 What arrives at COL-mer-01

```
  CONNECTOR              CLASS    RATE        TRANSPORT     PRIORITY
  ────────────────────────────────────────────────────────────────
  con-fortigate-dc-01    STREAM   6,200 EPS   udp/514       P3
  con-fortigate-dc-02    STREAM   3,100 EPS   udp/514       P3
  con-paloalto-perim     STREAM   1,400 EPS   udp/514       P3
  con-nsx-dfw            PUSH       180 EPS   https hook    P2
  con-fortimanager       PULL     ~4 req/min  rulebase      P2
                                  ──────────
                                  ~10,900 EPS

  97% OF THIS COLLECTOR'S EPS IS STREAM — the class with no
  authentication and no backpressure. That is COL-mer-01's risk
  profile in one line, and it is why it sits on an isolated
  collection VLAN with no route to the corporate network.
```

### 10.2 A pressure event, end to end

```
  14:02:11  parser workers restart to load a new parser bundle
  14:02:11  OVERLOOK_RAW depth climbing — nothing is consuming

  14:02:38  depth 61%  → YELLOW
              worker pools scale toward max (LLD §17)
              con-fortimanager P2 polling unaffected

  14:02:52  depth 77%  → ORANGE
              PULL polling suspended for P3/P4 — none here
              NSX (P2) acks delayed 200 ms; NSX slows. No loss.
              ⚠ nothing can be done for the three UDP sources

  14:03:19  depth 96%  → RED
              NSX receives 503, Retry-After: 30. It will resend.
              ⚠ FortiGate STREAM sources begin dropping

              overlook_events_dropped_total{con-fortigate-dc-01,
                                            buffer_full} += 4,180
              coverage window SHORTENED for all three
              UI: "COL-mer-01 · RED · degraded collection · UDP"

  14:03:41  parsers back, draining at 14,000 EPS
  14:04:56  depth 58% → GREEN

  RESULT
    PULL     0 lost   checkpoint resumed
    PUSH     0 lost   NSX retried successfully
    STREAM   ~11,400 lost, COUNTED AND ATTRIBUTED

  The 11,400 are gone. What matters is that SaaS knows a 90-second
  hole exists in FortiGate coverage and will not assert that a
  connection stopped happening during it.
```

### 10.3 The same 90 seconds without coverage windows

```
  No counters. No window shortening.

  SaaS observes  3 firewall connections last seen 14:02, not since
             →   marks 3 CONNECTS_TO relationships removed
             →   2 attack paths disappear
             →   exposure score 58 → 54

  Nothing changed in Meridian's estate. A parser restart improved
  their security posture by four points.

  → escalation ESC-5.
```

---

*Next: [Durable Event Buffer](02-durable-event-buffer.md)*

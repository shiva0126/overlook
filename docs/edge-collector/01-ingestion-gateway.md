# 1 — The Ingestion Gateway

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 1. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **1.5 vCPU · 2 GB RAM.**

---

## 1. Purpose

The single front door. Everything a customer's estate sends or offers arrives
here, and the gateway's job is to get it into the durable buffer — or to refuse
it in a way the sender understands.

It is the only service that talks to anything outside the collector, which makes
it the only service with an attack surface, and the only one whose failure is
visible to the customer's own infrastructure.

---

## 2. Position

```
  INPUTS
    PULL     connector workers fetching from APIs on a schedule
    PUSH     syslog/TCP, HTTP webhooks, file drops
    STREAM   syslog/UDP, netflow, SNMP traps
    AGENT    Overlook agents with local buffering

  OUTPUTS
    validated, framed, envelope-wrapped records → JetStream (02)
    an ACK, a 503, or silence — per class, per §4

  CONSUMED BY
    02 durable event buffer
    the Controller, for per-source health and rate telemetry
```

---

## 3. The four ingress classes

Everything else in this document follows from these four having genuinely
different durability contracts. They are the **only** structural variation in
the collector; every other apparent difference between sources is a parser or
a field mapping.

```
  PULL      we fetch, on a cursor
            LOSS MODEL   refetch. The cursor is not advanced until
                         the batch is durable.
            EXAMPLES     AWS CloudTrail, GitHub audit, Scalefusion,
                         CrowdStrike Falcon, Azure Graph
            BACKPRESSURE trivially — stop calling.

  PUSH      the source sends and waits for an acknowledgement
            LOSS MODEL   IRRECOVERABLE. If we ack and then lose it,
                         it is gone. fsync BEFORE ack, always.
            EXAMPLES     syslog/TCP with RELP, HTTP webhooks,
                         file drop with move-on-complete
            BACKPRESSURE refuse the ack — the sender retries.

  STREAM    the source fires and forgets
            LOSS MODEL   we drop, silently, unless we count it.
            EXAMPLES     syslog/UDP, netflow, SNMP traps
            BACKPRESSURE NONE EXISTS. See §4.3.

  AGENT     the source holds its own buffer
            LOSS MODEL   the agent retries; loss only if its buffer
                         overflows first
            EXAMPLES     the Overlook endpoint agent
            BACKPRESSURE tell the agent to hold. It is built for this.
```

### 3.1 One gateway, four adapters

Each class gets its own listener and its own ack path. They converge on one
validation and envelope stage, and one JetStream publisher.

```
  4 listeners
    │
    ├─ auth        per class, per §5
    ├─ validate    size, encoding, framing, structure
    ├─ envelope    wrap with collection metadata (§6)
    │
    └─► publish to JetStream, subject raw.<class>.<connector_id>
            │
            └─ on publish-ack ──► release the ingress ack per class
```

**The ack does not leave the gateway until JetStream confirms the write.** This
single rule is what makes the PUSH contract real, and it is the thing most
likely to be quietly broken during optimisation, because acking early makes
every benchmark look better.

---

## 4. Flow control, and the one class that cannot have it

When JetStream approaches its limit — because the Parser Engine has fallen
behind, or the disk is filling — the gateway must apply backpressure. What that
means is different for each class, and getting it wrong loses data.

### 4.1 The propagation table

```
  CLASS    WHEN THE BUFFER IS FULL             DATA LOSS?
  ──────────────────────────────────────────────────────────────
  PULL     stop scheduling fetches.            NONE
           The cursor stays where it is.
           The API still holds the data.

  PUSH     stop acking.                        NONE, if the sender
           TCP: stop reading, let the window     is well behaved
           close. HTTP: return 503 with
           Retry-After.

  AGENT    send a HOLD signal on the control   NONE, until the
           channel. The agent buffers.           agent's own buffer
                                                 overflows.

  STREAM   ⚠ DROP. There is no other option.   YES — and this is
           UDP has no acknowledgement and no     the case §4.3
           flow control by design.               exists for.
```

### 4.2 The thresholds

```
  buffer < 60%     normal
  60–75%           PULL scheduling slows to 50%
  75–85%           PULL stops. AGENT hold signal sent.
  85–95%           PUSH acks slow — deliberate delay before ack,
                   which throttles well-behaved senders without
                   failing them
  > 95%            PUSH returns 503. STREAM is dropped and counted.
                   The Controller raises a degraded-coverage alarm.
```

The graduated response matters. A binary "accept / reject" switch produces a
sawtooth: the buffer fills, everything is rejected, senders retry in unison,
the buffer fills again. Slowing before refusing gives the parser time to catch
up without a thundering herd.

### 4.3 STREAM loss must be counted, not absorbed

This is the most important paragraph in this document.

```
  A DROPPED UDP PACKET IS INDISTINGUISHABLE FROM A QUIET NETWORK.

  If 40,000 firewall events are dropped and nothing records it, then
  downstream:
      · the firewall looks quiet
      · fewer connections are observed
      · fewer relationships are asserted
      · the customer's exposure score IMPROVES

  Silence must therefore be measured directly:

    stream_received_total{connector}
    stream_dropped_total{connector,reason}     ← buffer_full, malformed,
                                                 rate_limited, oversize
    stream_gap_seconds{connector}              ← time since last packet

  and every drop shortens the COVERAGE WINDOW for that connector (05 §6),
  which is what stops absence of data being read as evidence of absence.
```

**PROPOSED** — coverage windows are not in the handoff. Escalation E3. The
gateway is where the counters originate, which is why it is raised here rather
than only in `05`.

---

## 5. Authentication, per class

```
  PULL      outbound. The gateway holds no inbound surface.
            Credentials are the connector's, sealed in Local Store (08 §3).

  PUSH      HTTP webhooks    HMAC signature over the body, per source,
                             plus optional mTLS
            syslog/TCP       mTLS where the sender supports it —
                             most enterprise syslog does not
            file drop        filesystem permissions plus a watched
                             directory owned by the collector

  STREAM    ⚠ NONE IS POSSIBLE. syslog/UDP and SNMP traps cannot be
            authenticated. Source-IP allowlisting is the only control,
            and it is a weak one — spoofing UDP is trivial.
            → the network path is the control. This must be stated in
              the deployment guide, not assumed.

  AGENT     mTLS with per-agent client certificates, issued at
            enrolment, rotated on a schedule.
```

**The STREAM gap is real and cannot be engineered away at the gateway.** An
attacker on the customer's network can inject syslog. The mitigations are
network segmentation, and treating STREAM-derived facts with lower confidence
than PULL-derived ones (`05 §7`).

---

## 6. The collection envelope

Every record leaves the gateway wrapped in metadata that no later stage can
reconstruct. Losing any of it here loses it permanently.

```
  {
    collector_id      COL-mer-01
    tenant_id         TEN-meridian
    connector_id      CON-fortigate-dc-01
    ingress_class     STREAM
    received_at       collector clock, RFC3339 with nanoseconds
    source_addr       10.4.1.20
    transport         udp/514
    raw_bytes         1104
    sequence          per-connector monotonic counter
    trace_id          for end-to-end debugging
  }
```

```
  received_at IS NOT event_time.

  The event carries its own timestamp, which the parser extracts, and
  the two routinely disagree by hours — batched exports, buffered
  agents, wrong timezones, unsynchronised clocks. Both are carried the
  whole way. Collapsing them here destroys the only evidence that a
  source's clock is wrong.

  sequence is per-connector and monotonic. Gaps in it at the SaaS end
  are how missing data is detected without trusting any single hop.
```

---

## 7. Considerations

**The gateway must be the least clever service in the collector.** It has the
only externally reachable attack surface and it is in the path of every byte.
Parsing, enrichment and any content-dependent logic belong downstream, behind
the durable buffer. A gateway that understands log formats is a gateway that
can be crashed by a malformed log.

**Validation is structural only.** Size caps, encoding checks, framing
correctness, envelope completeness. Not schema, not content. A record that
fails structural validation is dropped and counted; a record that is merely
strange is passed through and dealt with by the parser's quarantine path
(`03 §6`).

**Rate limits are per connector, not global.** One misconfigured firewall in
debug mode must not consume the budget of forty other sources. Per-connector
token buckets, with the limit derived from the connector's declared expected
rate plus a headroom multiple.

**The 503 must carry Retry-After.** Without it, well-behaved HTTP senders retry
immediately and turn backpressure into a denial of service against yourself.

**File drop needs move-on-complete, never read-on-appear.** A file appearing in
a watched directory is not a file that has finished being written. The contract
is: the sender writes to `.tmp`, then renames. The gateway only reads completed
names.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Acking before JetStream confirms | PUSH data lost on crash, silently | Ack is released only on publish-ack; assert in the Phase gate |
| STREAM drops uncounted | Coverage looks complete; score improves on blindness | Drop counters + coverage window shortening (§4.3) |
| Binary accept/reject | Sawtooth, thundering herd | Graduated thresholds (§4.2) |
| Global rate limit | One noisy source starves forty | Per-connector token buckets |
| `received_at` used as `event_time` | Clock skew becomes invisible; ordering wrong | Both carried end to end |
| Gateway parses content | Malformed input crashes the front door | Structural validation only |
| 503 without Retry-After | Self-inflicted DoS under load | Mandatory header |
| File read before write completes | Truncated records, silent corruption | Move-on-complete contract |
| UDP source spoofing | Injected facts | Network segmentation + lower confidence on STREAM |

---

## 9. Example: Meridian

### 9.1 What arrives at COL-mer-01

```
  CONNECTOR                CLASS    RATE        TRANSPORT
  ─────────────────────────────────────────────────────────────
  CON-fortigate-dc-01      STREAM   6,200 EPS   udp/514
  CON-fortigate-dc-02      STREAM   3,100 EPS   udp/514
  CON-paloalto-perim       STREAM   1,400 EPS   udp/514
  CON-nsx-dfw              PUSH       180 EPS   https webhook
  CON-fortimanager         PULL     ~4 req/min  rulebase, hourly
                                     ──────────
                                     ~10,900 EPS

  97% of this collector's EPS is STREAM, and STREAM is the class
  with no authentication and no backpressure. That is the risk
  profile of COL-mer-01 in one line, and it is why it was placed
  on an isolated collection VLAN.
```

### 9.2 A backpressure event, end to end

```
  14:02:11  parser engine restarts to load a new parser bundle
  14:02:11  JetStream depth begins climbing — nothing is consuming

  14:02:38  depth 61%
              → CON-fortimanager PULL scheduling slowed to 50%
              → no loss; the API still holds everything

  14:02:52  depth 77%
              → PULL stopped entirely, cursor frozen
              → no AGENT connectors on this collector

  14:03:04  depth 88%
              → NSX webhook acks delayed 400 ms each
              → NSX slows its send rate. No loss.

  14:03:19  depth 96%
              → NSX receives 503, Retry-After: 30. It will resend.
              → ⚠ 3 FortiGate STREAM sources begin dropping

              stream_dropped_total{CON-fortigate-dc-01} += 4,180
              coverage window for CON-fortigate-dc-01 SHORTENED
              Controller: "COL-mer-01 · degraded collection · UDP"

  14:03:41  parser back up, draining at 14,000 EPS
  14:04:56  depth 58%, all classes restored

  RESULT
    PULL     0 lost   cursor resumed
    PUSH     0 lost   NSX retried successfully
    STREAM   ~11,400 events lost, COUNTED AND ATTRIBUTED

  The 11,400 are gone. What matters is that SaaS knows a 90-second
  hole exists in FortiGate coverage and will not assert that a
  connection stopped happening during it.
```

### 9.3 What this would have looked like without coverage windows

```
  Same 90 seconds. No counters, no window shortening.

  SaaS observes: 3 firewall connections last seen at 14:02
                 and not seen since
             →   marks 3 CONNECTS_TO relationships as removed
             →   2 attack paths disappear
             →   exposure score 58 → 54

  Nothing changed in Meridian's estate. A parser restart
  improved their security posture by four points.
```

---

*Next: [Durable Event Buffer](02-durable-event-buffer.md)*

# Agent Ingestion

**Series:** [Ingestion](00-index.md) · **Sources:** Overlook Agent · AI Gateway

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. What it is

Our own software reporting in. The easiest ingress class to make correct, because we control both ends — and the one carrying the most differentiated data in the catalog (`../connectors/10`).

The defining property: **the agent buffers its own output**, so retry lives at the source rather than in our durability contract.

---

## 2. How it works, step by step

```
  1  AGENT COLLECTS LOCALLY
     4-hourly AI config scan, 60-second heartbeat.
     Output written to a bounded local buffer:
       24 hours OR 200 MB, whichever comes first.

  2  AGENT DIALS OUT
     mTLS to the collector's agent gateway, port 8443.
     AGENT-INITIATED, ALWAYS. The agent has no listening port,
     which removes an entire vulnerability class.

  3  MUTUAL AUTHENTICATION
     Agent presents its client certificate, issued at enrollment.
     Collector presents its own. Both verify. Revocation checked.

  4  BATCH UPLOAD
     Records batched, zstd-compressed, with per-agent sequence
     numbers so gaps are detectable.

  5  COLLECTOR JOURNALS + FSYNC

  6  COLLECTOR ACKS, with the highest sequence number accepted

  7  AGENT PRUNES its local buffer up to that sequence number
     ONLY on ack. Never before.

  8  COLLECTOR RETURNS A PACING HINT (optional)
     Under load, tells the agent to extend its batch interval.

  9  AGENT POLLS FOR COMMANDS on the same connection
     Config updates, content updates, response requests (Mode 2).
```

---

## 3. Enrollment

```
  1  operator generates an enrollment token in the Controller
     single-use · 24h TTL · bound to the deployment

  2  agent installed with the token (via MDM, GPO, config
     management, or manually)

  3  agent generates a keypair LOCALLY. The private key never
     leaves the endpoint, and is TPM/Secure-Enclave backed where
     available.

  4  agent presents the token plus a CSR over TLS

  5  collector validates the token, issues a client certificate
     bound to (deployment_id, agent_id), 90-day lifetime

  6  ongoing: auto-renew at 2/3 lifetime over the established mTLS
     channel; revocation list checked on every connection
```

**The 90-day certificate with 60-day renewal has a failure mode worth stating.** An agent offline for more than 90 days cannot renew and must re-enroll. Laptops go on parental leave. Surface certificate expiry per agent in the Controller, and provide a re-enrollment path that does not require a technician.

---

## 4. The buffer, and why it changes the contract

```
  Every other ingress class assumes the source will not retry
  usefully. This one assumes it will.

  AGENT BUFFER
    bounded: 24 hours or 200 MB
    persisted to disk, encrypted
    pruned only on ack
    FIFO with priority — if the buffer fills, the OLDEST
    LOW-PRIORITY records are dropped first, and the drop is
    RECORDED and reported on the next successful connection

  CONSEQUENCE
    a laptop offline for a week reconnects and delivers a week of
    scans. A laptop offline for a month delivers the most recent
    24 hours and reports what it dropped.

  This is why the collector's contract is light: journal, fsync,
  ack. If the ack is lost, the agent resends. Duplicates are
  absorbed by sequence numbers and downstream idempotency.
```

---

## 5. Sequence numbers and gap detection

```
  Each agent maintains a monotonic sequence per collector.
  Every batch declares [first_seq, last_seq].

  THE COLLECTOR CAN THEREFORE DETECT
    duplicate batch   last_seq ≤ highest already accepted
                      → ack immediately, do not journal twice
    gap               first_seq > highest accepted + 1
                      → the agent dropped records under buffer
                        pressure. Recorded as a coverage gap for
                        that agent, not as an error.
    out of order      → accept, ordering is not required

  Gaps are DATA. "This endpoint dropped 4 hours of scans while
  offline and over-buffered" is a coverage fact, and it belongs in
  the coverage view rather than being silently absorbed.
```

---

## 6. Pacing, not backpressure

```
  Other classes push back by refusing (503) or shedding.
  This one negotiates.

  Under load the collector returns a pacing hint:
    { "extend_interval_to": 900, "reason": "ingest_backlog" }

  The agent extends its batch interval rather than dropping data,
  because it has a buffer and can afford to wait.

  This is strictly better than refusal: nothing is lost, the
  collector recovers, and 8,500 agents naturally de-synchronise
  instead of retrying in lockstep.
```

**Jitter is applied at the agent, not the collector.** 8,500 agents on a 4-hour cadence with no jitter is 8,500 connections in the same second, four times a day. Each agent jitters ±25% of its interval, deterministically seeded by its own ID so the spread is stable across restarts.

---

## 7. Coverage semantics

```
  AGENT COLLECTORS EMIT COVERAGE WINDOWS, PER HOST.

  { "collector": "agent.mcp_configs",
    "scope": "host:AST-lt-4471",
    "enumeration_complete": true,
    "paths_checked": 7, "paths_known": 7 }

  A host that reported completely → its MCP servers may be
  retracted if absent.
  A host that did not report      → STALE, nothing retracted.
  A host with a TCC-denied path   → NO window. Partial coverage
                                    would tombstone servers that
                                    still exist.

  A CLOSED LAPTOP IS NOT A REMOVED CONFIGURATION.
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Ack lost in transit | Agent resends | Sequence numbers make the duplicate free |
| Agent buffer full | Oldest low-priority records dropped | Drop recorded and reported as a coverage gap |
| Agent offline > 90 days | Certificate cannot renew | Expiry surfaced per agent; self-service re-enrollment |
| No jitter | 8,500 simultaneous connections | Deterministic per-agent jitter, ±25% |
| Collector overloaded | Agents retry in lockstep | Pacing hints, not refusal |
| Partial local scan (TCC denied) | Wrongful tombstoning | No coverage window for that host |
| Agent compromised | Malicious data injected | mTLS identity, per-agent anomaly baselines, and the agent has no privileged capability to abuse |
| Clock skew on the endpoint | Timestamps wrong | Collector records receive time alongside agent time |

---

## 9. Considerations

**Agent-initiated, outbound-only, no listening port.** This is worth more than it appears. A remotely-exploitable agent listener has ended companies. It also makes firewall approval trivial — there is no inbound rule to request.

**The agent must not be the reason a laptop is slow.** Published, self-enforced limits: CPU under 1% average, RAM under 150 MB, disk IO under 5 MB/s during scans, network under 50 MB/day. A watchdog throttles or suspends collection when the host is under load, and the Controller shows a resource-usage report so the customer can verify.

**Response is a separate package.** The collection agent has no execution capability. A customer can deploy the entire fleet with provably zero ability to act, and verify it in the Controller.

**Volume is small and value is high.** 102 MB/day from 8,500 endpoints. The scope discipline — collecting only what no API can provide — is what makes the class cheap. Collecting process trees here instead of from the EDR would make it ~2.4 billion records a day.

---

## 10. Example: Meridian, one hour of agents

```
  09:00-10:00, COL-DC1, 8,500 enrolled agents

  CONNECTIONS
    ~2,100 agents reported this hour
    (4-hour cadence with ±25% jitter spreads 8,500 across the day)

  VOLUME
    ~4.2 MB total, zstd-compressed
    → journaled + fsync
    → acked with the highest sequence accepted
    → each agent pruned its buffer

  ONE PACING EVENT
    09:14  the collector is briefly loaded — the nightly Parquet
           compaction overlapped a large AD delta
    → pacing hint returned: extend to 900s
    → 340 agents in that window extended their interval
    → nothing dropped, nothing refused
    → normal cadence resumed at 09:22

  ONE GAP
    AST-lt-8891 reconnects after 9 days offline (annual leave)
      first_seq 44,102 · highest previously accepted 43,110
      → gap of 992 records
      → the agent's buffer held 24h; the remaining 8 days were
        dropped at source, oldest-first
      → recorded as a coverage gap for that host
      → its MCP servers marked STALE, NOT tombstoned

  ONE PARTIAL SCAN
    AST-mb-2204 (macOS) reports mcp_configs with
      paths_checked 5 · paths_known 7
      two paths returned TCC permission denied
      → NO coverage window for that host
      → surfaced as a DEPLOYMENT GAP with the PPPC remediation,
        not as a collection error

  THREE SILENT AGENTS
    reported nothing for 48 hours, no error, certificate valid
      → SILENT state, surfaced in the attention inbox
      → two are decommissioned laptops; one is a genuine service
        failure on a build server

  WHAT ARRIVED
    340 hosts with MCP configurations
     47 mcp-filesystem servers rooted at work directories
     31 configs holding a credential (type and presence only)
    120 hosts running local models
  4,200 hosts with IDE AI extensions
```

**4.2 MB an hour, and it contains the layer no competitor collects.** Compare 89 GB from four firewalls in the same hour, producing ~90 edges a day.

---

*Next: [The ingest journal](05-journal.md)*

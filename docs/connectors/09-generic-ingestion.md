# Domain 09 — Generic Ingestion

**7 connectors · 24 collectors** · [Index](00-index.md)

Continuous — outside the banded cycle entirely. These are not vendor integrations; they are **protocol terminations and configurable adapters** that convert an unbounded long tail into a configuration exercise rather than an engineering queue.

⚠ **The strategic point.** Stellar Cyber builds connectors per customer request and ships them on the release train (`../../08-connector-benchmark-and-alignment.md §2.3`). As an MSSP with N customers, each with one unusual internal system, that model would consume all engineering capacity. This domain is the alternative.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · Syslog receiver

```
  api_surface   log_stream
  auth          source IP allow-list + optional TLS client certs
  coverage      never emits coverage windows — a stream can never
                prove a complete enumeration, so it can NEVER drive
                retraction
  ⚠             UDP loses data silently and cannot be made reliable.
                Recommend TCP/TLS; where UDP is unavoidable, measure
                and DISPLAY estimated loss from sequence gaps and
                receiver drop counters rather than implying the feed
                is complete.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `rfc5424` | Structured syslog with the modern framing | typed records | continuous | ★ |
| `rfc3164` | Legacy BSD syslog | typed records | continuous | ★ |
| `cef` | ArcSight CEF payloads | typed records, vendor-normalised | continuous | ★ |
| `leef` | QRadar LEEF payloads | typed records | continuous | |
| `json_lines` | JSON-over-syslog | typed records | continuous | ★ |
| `source_registry` | Which IP:port is which source, and its fingerprint | source identification, parse-rate baselines | 12h | ★ |

**6 collectors.**

The `source_registry` is what makes the rest usable — it holds the `(src_ip, port) → vendor/product/version` bindings that E2 Fingerprint establishes, plus the per-source parse-rate and field-presence baselines that catch silent degradation.

---

## 2 · Webhook receiver

```
  api_surface   push
  auth          per-source HMAC signature or mTLS
  ⚠             ack ONLY after fsync. Returning 200 before the record
                is durable is the classic data-loss bug — the sender
                will never send it again.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `generic_receiver` | Signed HTTP POSTs on a per-source path | raw records with provenance | continuous | ★ |
| `signature_verification` | HMAC/mTLS validation per source | rejects unauthenticated deliveries | continuous | ★ |
| `replay_guard` | Delivery-ID seen-set plus timestamp window | idempotency, replay protection | continuous | ★ |

**3 collectors.**

---

## 3 · Message bus (Kafka / Event Hubs / Pub-Sub)

```
  For customers who already centralise telemetry. Consuming their
  existing bus is far cheaper than re-collecting from every source,
  and it respects an investment they already made.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `kafka` | Topics with consumer-group offset tracking | raw records | continuous | ★ |
| `event_hubs` | Azure Event Hubs with checkpointing | raw records | continuous | ★ |
| `pubsub` | Google Pub/Sub subscriptions | raw records | continuous | ★ |
| `schema_registry` | Avro/Protobuf schemas where present | typed decoding without inference | 12h | |

**4 collectors.**

Offset and checkpoint management is the whole difficulty here: a consumer group that resets loses position, and one that never commits reprocesses forever.

---

## 4 · Cloud log-bucket puller

```
  Archived logs in S3, Azure Blob or GCS. Common for customers who
  route everything to storage before anything reads it.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `s3_puller` | Objects by prefix, with a cursor | raw records | 15m | ★ |
| `blob_puller` | Azure Blob containers by prefix | raw records | 15m | ★ |
| `gcs_puller` | GCS buckets by prefix | raw records | 15m | ★ |

**3 collectors.** Unlike live streams, these **can** emit coverage windows — an object listing over a bounded prefix is a complete enumeration.

---

## 5 · Generic REST poller

```
  The long-tail answer. A customer-configurable connector defined
  entirely in configuration: endpoint, auth, pagination, mapping.

  ⚠ THE MOST IMPORTANT CONNECTOR IN THIS DOMAIN. Every enterprise
    has 5-10 internally-built systems that will never justify a
    bespoke connector. This turns each of them into an afternoon of
    configuration instead of a ticket in our backlog.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `rest_poller` | Any JSON REST endpoint, paginated | raw records | configurable | ★ |
| `auth_adapters` | Basic, bearer, OAuth2 client credentials, API key, mTLS | authenticated access | — | ★ |
| `mapping_dsl` | Declarative field mapping to the Overlook schema | entities and relationships from arbitrary JSON | — | ★ |
| `pagination_adapters` | Cursor, offset, link-header, token schemes | complete enumeration | — | ★ |

**4 collectors.**

The `mapping_dsl` is the piece that makes this more than a log shipper: it lets a customer declare *"the `owner_email` field in this response is an IDENTITY canonical key, and `resource_arn` is an ASSET it CAN_READ"* — producing real graph edges from a system we have never seen.

---

## 6 · File drop (CSV / JSON / spreadsheet)

```
  Deliberately mundane, and used more than expected. Asset lists,
  crown-jewel designations, exception registers, ownership
  spreadsheets — the things every organisation maintains by hand.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `csv_import` | Watched directory or upload, with a column mapping | entities, properties, ownership | on-demand | ★ |
| `crown_jewels` | Customer-declared critical asset list | **criticality designation** | on-demand | ★ |
| `exceptions` | Accepted-risk register with expiry dates | finding suppression with a reason | on-demand | ★ |

**3 collectors.**

`crown_jewels` is load-bearing out of proportion to its sophistication. The path engine computes *toward* crown jewels; if a customer can only designate them through a UI one at a time, they will designate three. A CSV import means they designate two hundred, and the path engine finally has the right targets.

---

## 7 · SNMP / network device discovery

```
  For unmanaged network gear with no API — the switches, printers,
  UPSs and collectors that exist in every estate and appear in no
  inventory.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `device_discovery` | SNMP walk of system and interface MIBs | `ASSET`, unmanaged-device findings | 24h | ★ |
| `interface_topology` | Interface tables, LLDP/CDP neighbours | `NETWORK`, physical topology | 24h | |

**2 collectors.**

---

## Domain summary

| Connector | Collectors | Emits coverage windows? |
|---|---|---|
| Syslog receiver | 6 | ✕ never |
| Webhook receiver | 3 | ✕ never |
| Message bus | 4 | ✕ never |
| Cloud log-bucket puller | 3 | ✓ yes |
| Generic REST poller | 4 | ✓ yes, if pagination completes |
| File drop | 3 | ✓ yes, per import |
| SNMP discovery | 2 | ✓ yes, per scanned range |
| **Total** | **25** | |

### The coverage-window rule, stated plainly

```
  A STREAM can never prove it saw everything.

  syslog · webhooks · message bus
    → these connectors can only ADD to the graph. They can never
      retract, because "I did not see it" is indistinguishable from
      "it was not sent."

  pullers · pollers · imports · scans
    → these enumerate a bounded set and CAN emit a coverage window,
      and therefore can safely drive retraction.

  Conflating the two is how a graph silently deletes half of itself
  after a quiet weekend.
```

### Why this domain exists at all

Two reasons, and both are strategic rather than technical:

**It bounds the connector backlog.** Without it, every unusual customer source is an engineering ticket. With it, most become configuration — the difference between an MSSP that can onboard a customer this month and one that cannot.

**It respects existing investment.** A customer who already ships everything to Kafka, or archives to S3, or maintains an asset spreadsheet, has done work we can consume rather than duplicate. Asking them to let us re-collect from forty sources they already centralised is both wasteful and a poor first impression.

---

*Next: [The Overlook Agent](10-agent.md)*

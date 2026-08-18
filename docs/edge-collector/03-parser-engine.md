# 3 — The Parser Engine

**Series:** [The Edge Collector](00-index.md)

---

> **⚠ THIS SERIES IS THE EDGE COLLECTOR AS HANDED OVER.**
> Specifies handoff §6.1 service 3. Hard ceiling:
> **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**
> Budget for this service: **4.0 vCPU · 8 GB RAM.** The largest
> allocation in the collector, and correctly so.

---

## 1. Purpose

Turn bytes into a structured, normalized record with named fields, or admit
that it could not and say so loudly.

This is the hot path. It is the only stage that touches every byte of every
event; everything downstream operates on progressively less data. It is also
where the collector's genericness is won or lost — if a new source needs new
code here, the catalog cannot grow.

---

## 2. Position

```
  INPUTS
    enveloped raw records, from JetStream consumers (02 §6.1)
    parser content bundles, loaded at runtime (§4)

  OUTPUTS
    normalized records → Enrichment Engine (04)
    reduced records    → Local Analytics (08 §4)
    unparsed records   → quarantine subject (§6)
    parse telemetry    → the Controller

  CONSUMED BY
    04 enrichment engine
    08 local analytics
    the auto-parser feedback loop (../autoparser/)
```

---

## 3. The four stages

```
  DETECT      what format is this?
  PARSE       apply the parser, produce fields
  VALIDATE    are the required fields present and well-typed?
  NORMALIZE   map source fields onto the common schema
```

### 3.1 Detect

Format detection precedes parser selection and is cheap, structural and
content-independent.

```
  ORDER OF ATTEMPT — first match wins, all are O(first few bytes)

    1  JSON            leading { or [ , balanced
    2  CEF             "CEF:0|" prefix
    3  LEEF            "LEEF:1.0|" or "LEEF:2.0|" prefix
    4  syslog RFC5424  "<PRI>1 " structured header
    5  syslog RFC3164  "<PRI>" then BSD timestamp
    6  key=value       ≥3 unquoted k=v pairs
    7  CSV/TSV         consistent delimiter count across a sample
    8  XML             leading <
    9  PLAIN           none of the above

  DETECTION IS A HINT, NOT A DECISION.

  A connector with a declared parser binding skips detection
  entirely. Detection exists for unknown sources, for sources that
  change format after an upgrade, and to notice when a bound
  parser's assumption has stopped being true.
```

### 3.2 Parser selection

```
  1  CONNECTOR BINDING     CON-fortigate-dc-01 → parser fortinet.fortios.v7
                           explicit, versioned, the normal case

  2  DETECTED FORMAT       a generic parser for the detected format,
                           when no binding exists

  3  AUTO-PARSER           the L0–L5 pipeline (../autoparser/) proposes
                           a parser from observed structure

  4  QUARANTINE            none of the above succeeded — §6
```

**A mismatch between binding and detection is a signal, not an error.** If
`CON-fortigate-dc-01` is bound to a FortiOS 7 parser and detection starts
reporting JSON where it reported syslog, the firewall was upgraded. That is
worth an alert before it is worth a parse failure.

### 3.3 Parse

The parser itself is **content, not code** — see §4. At runtime it is a compiled
matcher applied to the record.

```
  PER-RECORD BUDGET AT 10,000 EPS ACROSS 8 WORKERS

    8 workers × 1,250 records/sec each
    = 800 µs per record, wall clock, per worker
    of which parsing must fit in ~300 µs

  That budget is generous for a grok pattern and tight for a
  chain of unanchored regexes. It is the reason for §7's rule
  about anchoring.
```

### 3.4 Validate and normalize

```
  VALIDATE     required fields present · types coercible ·
               timestamp parseable and within a sane window ·
               no field exceeding its size cap

  NORMALIZE    source field  →  common schema field

               FortiGate  srcip      →  source.ip
                          dstip      →  destination.ip
                          srcintf    →  source.interface
                          action     →  event.action   (mapped values)
                          devname    →  observer.name

               CrowdStrike LocalIP    →  source.ip
                           ComputerName → host.name
                           UserName   →  user.name

  Two sources, one schema. Everything downstream reads
  source.ip and neither knows nor cares which product produced it.
```

**The common schema is the collector's most load-bearing contract.** It is what
lets the Fact Engine extract a relationship without a per-source branch. It
belongs in `contracts/` as a specification file rather than in prose, and it is
the second thing to write after the Security Fact schema.

---

## 4. Parsers are content, not code

This is the design decision that determines whether the collector can carry a
hundred connectors.

```
  IF A NEW SOURCE MEANS NEW CODE
    new source → code change → rebuild → re-run the handoff §13
    gates → redeploy the fleet.
    Connector #40 costs what connector #1 cost.

  IF A NEW SOURCE MEANS NEW CONTENT
    new source → a parser definition in a signed bundle → loaded
    at runtime → no rebuild, no re-qualification of the binary.
    The binary qualified at Phase 12 is the binary at connector #100.
```

### 4.1 The bundle

```
  parsers-2026.08.18.bundle          signed, versioned, immutable
    ├── fortinet.fortios.v7.yaml
    ├── crowdstrike.falcon.v3.yaml
    ├── scalefusion.audit.v1.yaml
    ├── generic.cef.yaml
    ├── generic.rfc5424.yaml
    └── manifest.json                versions, hashes, schema version

  LOADING
    · fetched from SaaS or side-loaded for air-gapped sites
    · signature verified before load
    · loaded into a NEW parser table; the old table serves traffic
      until the new one is ready, then an atomic swap
    · ⚠ NO PROCESS RESTART. A restart drains the JetStream consumer
      and, per 01 §9.2, restarts are what cause STREAM loss.
```

**Hot reload is not a convenience feature here.** Because the collector's only
uncontrolled loss class is STREAM, and because STREAM loss happens whenever the
parser stops consuming, every avoided restart is avoided data loss. Parser
updates will be frequent. They must not cost packets.

### 4.2 Where the auto-parser fits

```
  A source with no binding and no matching generic parser goes to
  the auto-parser rather than straight to quarantine.

    L0–L5 (../autoparser/) observes structure across many records,
    clusters them with Drain, and PROPOSES a parser definition.

    PROPOSE  →  CONFIRM  →  FREEZE

    proposed   used, flagged, facts carry reduced confidence
    confirmed  a human accepted it in the Controller
    frozen     promoted into the next content bundle

  ⚠ PROPOSED — the auto-parser is escalation E9; it is not named in
    the handoff. It is what makes the MSSP catalog tractable, since
    every customer has three sources nobody has ever seen.
```

---

## 5. Parsing makes the data bigger

A sizing fact that is easy to get backwards.

```
  raw syslog line                              1,104 bytes
  parsed, normalized, JSON, named fields       1,330 bytes    +20%

  IT NEVER PERSISTS IN THIS FORM.

  The parsed record exists only in flight, between the parser and
  the Fact Engine. Nothing writes it to disk. If it ever needs to
  be persisted — for debugging, for a replay buffer, for analytics
  — the sizing in 00 §4.3 is wrong and must be redone.

  Local Analytics receives a REDUCED projection (08 §4), not this.
```

---

## 6. Quarantine — never drop, always account

```
  A RECORD REACHES QUARANTINE WHEN
    · detection returned PLAIN and no parser matched
    · a bound parser threw or produced no required field
    · validation failed — unparseable timestamp, type mismatch
    · JetStream max-deliver was exhausted (02 §4.2)

  WHAT HAPPENS
    · published to  quarantine.<connector_id>
    · retained 7 days, capped at 2 GB per collector
    · sampled — the first 1,000 distinct shapes per connector per
      day, not every record. A source emitting 40,000 EPS of an
      unparseable format must not fill the disk with evidence
      of a single fact.
    · counted:  parse_failed_total{connector,reason}
    · surfaced in the Controller as a per-connector parse rate

  WHAT NEVER HAPPENS
    · silent discard
    · a parse failure shortening the coverage window ← see below
```

```
  PARSE FAILURE IS NOT A COVERAGE GAP.

  A gap means WE DID NOT SEE THE DATA. A parse failure means we saw
  it and could not read it. The data is in quarantine and will be
  reprocessed when the parser is fixed (02 §6.2).

  Conflating the two makes a parser bug look like a collection
  outage, which sends the investigation to the wrong team.
```

---

## 7. Considerations

**Anchor every pattern.** An unanchored regex over a 4 KB log line backtracks,
and a pathological input turns 300 µs into 300 ms. At 1,250 records/sec/worker
that is a stalled worker, a growing buffer, and eventually STREAM drops at the
gateway. Anchoring, possessive quantifiers, and a per-record parse timeout with
the record routed to quarantine on expiry.

**The parser table swap must be atomic and the old table must drain.**
In-flight records were matched against the old table; completing them under the
new one produces records that are half one schema and half another.

**Do not parse what will be filtered.** If the noise policy (`05 §5`) drops 70%
of firewall traffic logs, evaluate the drop predicate on the raw line where the
predicate allows it. Parsing then discarding is the single largest waste
available in the collector, and it is spending the most expensive budget in the
box.

**One worker pool per stream, not one shared pool.** A slow parser on a PULL
source with 2 MB API responses must not occupy workers that STREAM needs. The
consumer split in `02 §6.1` exists for this; honour it with separate pools.

**Version parsers, never mutate them.** `fortinet.fortios.v7` and
`fortinet.fortios.v7.1` coexist. Facts carry the parser version that produced
them, which is what makes the replay diff in `02 §9.2` meaningful — you can tell
which facts came from the defective version.

**Timestamp extraction is where most silent wrongness lives.** Two-digit years,
missing timezones, local time from a device in another region, and the
RFC3164 format that omits the year entirely. Every parser declares its timestamp
strategy explicitly, and a record whose extracted `event_time` is more than 24 h
from `received_at` is flagged rather than accepted quietly.

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Unanchored regex, pathological input | Worker stalls, buffer fills, STREAM drops at the gateway | Anchoring + per-record timeout → quarantine |
| Parser update requires restart | Consumer drains, STREAM data lost on every content update | Atomic hot swap, §4.1 |
| Parsers compiled into the binary | Every connector is a rebuild and a re-qualification | Content bundles, §4 |
| Parse-then-filter | Largest budget spent on discarded data | Evaluate drop predicates pre-parse where possible |
| Shared worker pool | A slow PULL source starves STREAM | Pool per stream |
| Silent discard of unparsed | Data loss looks like a quiet source | Quarantine, counted and sampled |
| Parse failure shortens coverage window | Parser bug reads as a collection outage | Distinct counters, §6 |
| Non-atomic table swap | Mixed-schema records | Swap with drain |
| Timestamp misparsed | Facts land at the wrong time; ordering wrong | Explicit strategy per parser + 24 h skew flag |
| Quarantine unsampled | 2 GB filled by one source in minutes | 1,000 distinct shapes/connector/day |

---

## 9. Example: Meridian

### 9.1 The parse profile of COL-mer-01

```
  CONNECTOR              FORMAT        PARSER                  RATE
  ─────────────────────────────────────────────────────────────────
  CON-fortigate-dc-01    RFC3164 KV    fortinet.fortios.v7    6,200
  CON-fortigate-dc-02    RFC3164 KV    fortinet.fortios.v7    3,100
  CON-paloalto-perim     CSV           paloalto.panos.v11     1,400
  CON-nsx-dfw            JSON          vmware.nsx.v4            180
  CON-fortimanager       JSON          fortinet.fortimanager     ~0

  PARSE SUCCESS RATE                                        99.97%
  QUARANTINED                                          ~3.3 EPS

  Of the quarantined, 91% is a single shape: FortiGate
  `type=event subtype=wad` records emitted by a WAN-optimisation
  module nobody enabled deliberately. It has been the same 3 EPS
  for six weeks.

  → the right response is a noise-policy drop rule, not a parser.
    Writing a parser for it would produce 285,000 correctly parsed
    records per day describing a feature that is not in use.
```

### 9.2 A format change, caught by detection

```
  2026-08-18 02:10   FortiOS upgraded 7.2 → 7.4 on dc-02 during a
                     maintenance window.

  02:11  detection on CON-fortigate-dc-02 begins reporting JSON
         where it has reported RFC3164 for eleven months

  02:11  BINDING/DETECTION MISMATCH raised
           bound     fortinet.fortios.v7   (expects RFC3164 KV)
           detected  JSON
           → the bound parser starts failing; records quarantine

  02:11  Controller alarm:
           "CON-fortigate-dc-02 · parse rate 99.9% → 4.1% ·
            format changed · likely device upgrade"

  02:14  the collector falls back to generic.json, which extracts
         enough for entity and relationship extraction to continue
         at REDUCED CONFIDENCE. Facts still flow.

  09:30  fortinet.fortios.v74 bundled, hot-loaded, no restart
  09:31  parse rate back to 99.96%
  09:40  the 02:11–09:31 window replayed from RAW_STREAM
         ⚠ 4 h retention — only 05:31 onward survived.
           3 h 20 m of full-fidelity data was lost to the buffer
           window, though the reduced-confidence facts from the
           JSON fallback remain.

  THE LESSON THAT WENT INTO THE RUNBOOK
    a binding/detection mismatch is a PAGE, not a ticket. The clock
    that matters is RAW_STREAM retention, and it is four hours.
```

### 9.3 What the fallback saved

```
  WITHOUT generic.json FALLBACK
    7 h 20 m of dc-02 entirely unparsed
    → coverage window for CON-fortigate-dc-02 collapses
    → SaaS correctly refuses to assert anything about that
      segment for the period
    → 14 attack paths through the dc-02 segment become
      UNKNOWN rather than absent — which is right, but useless

  WITH IT
    entity and relationship extraction continued from JSON
    field names that generic.json could map by convention
    (src_ip, dst_ip, action were recognisable)
    → 84% of the relationships still extracted
    → confidence 0.94 → 0.61, visible on every fact
    → the paths stayed visible, marked lower confidence

  A degraded parser that says so beats no parser. It also beats a
  confident parser that is wrong, which is what would have happened
  had the v7 parser silently coerced JSON into its expected fields.
```

---

*Next: [Enrichment Engine](04-enrichment-engine.md)*

# 3 — The Parser Engine

**Series:** [The Edge Collector](00-index.md) · **LLD:** §18, §19, §20, §83

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. Divergences are
> recorded in [13-escalations.md](13-escalations.md).
> Budget: **3.5 vCPU · 8 GB RAM** — the largest CPU allocation in the
> collector, and correctly so (`00 §4.1`).

---

## 1. Purpose

Turn bytes into structured fields, or admit that it could not and say so loudly.

This is the hot path. It is the only stage that touches every byte of every
event; everything downstream operates on progressively less data. It is also
where the collector's genericness is won or lost — if a new source needs new
code here, the catalog cannot grow.

---

## 2. Position

```
  INPUTS
    enveloped raw records from OVERLOOK_RAW (02)
    the parser registry (LLD §7 internal/parser/registry.go)

  OUTPUTS
    parsed events → normalization (04)
    unparseable   → overlook.deadletter (LLD §20)
    parse telemetry → overlook_events_parsed_total,
                      overlook_parse_failures_total (LLD §50)

  CONSUMED BY
    04 normalization engine
    the auto-parser feedback loop (../autoparser/) — LLD §83
```

---

## 3. The interface

LLD §18 defines it:

```go
type Parser interface {
    Name() string
    Supports(source Source, sample []byte) bool
    Parse(ctx context.Context, raw RawEvent) ([]ParsedEvent, error)
}
```

Three properties of this interface worth naming, because they constrain
everything below:

```
  Supports() TAKES A SAMPLE, NOT A FORMAT NAME
    → detection and selection are the same decision, and a parser
      can decline a record it does not recognise even when it is
      bound to the connector. That is what makes §5's format-change
      detection possible.

  Parse() RETURNS A SLICE
    → one raw record may contain many events. A CloudTrail page, a
      batched webhook, a multi-record file. The fan-out happens
      HERE, which is why parsed volume exceeds raw volume (§6).

  Parse() TAKES A CONTEXT
    → a per-record timeout is expressible. Use it (§8).
```

---

## 4. Parser selection

LLD §19:

```
  Incoming Event
       │
       ├── connector has a configured parser  →  use it
       │
       └── no parser  →  format detector
                           JSON · CEF · Syslog
                                 ↓
                          Generic Parser
```

### 4.1 Detection

Cheap, structural, content-independent. First match wins, all decided on the
first few bytes.

```
  1  JSON            leading { or [ , balanced
  2  CEF             "CEF:0|" prefix
  3  LEEF            "LEEF:1.0|" or "LEEF:2.0|"
  4  syslog RFC5424  "<PRI>1 " structured header
  5  syslog RFC3164  "<PRI>" then BSD timestamp
  6  key=value       ≥3 unquoted k=v pairs
  7  CSV/TSV         consistent delimiter count across a sample
  8  XML             leading <
  9  PLAIN           none of the above
```

### 4.2 A binding/detection mismatch is a signal, not an error

**PROPOSED — the LLD's flow treats detection as a fallback only.**

```
  If con-fortigate-dc-02 is bound to a FortiOS 7 parser and detection
  starts reporting JSON where it reported syslog for eleven months,
  THE FIREWALL WAS UPGRADED.

  That is worth an alarm before it is worth 3.1 million dead-letter
  records. Running detection continuously on a sampled 1-in-1000 of
  bound connectors costs almost nothing and turns a silent
  catastrophe into a page. Worked example in §10.2.
```

---

## 5. Parsers are content, not code

The decision that determines whether the collector can carry a hundred
connectors. LLD §7 puts `parsers/` beside `schemas/` at the repository root
rather than inside `internal/`, which implies this — it is worth making explicit.

```
  IF A NEW SOURCE MEANS NEW CODE
    new source → code change → rebuild → re-run every gate →
    redeploy the fleet. Connector #40 costs what connector #1 cost.

  IF A NEW SOURCE MEANS NEW CONTENT
    new source → a parser definition in a signed bundle → loaded at
    runtime → no rebuild. The binary qualified in the performance
    gate is the binary at connector #100.
```

### 5.1 The bundle

```
  parsers-2026.08.18.bundle              signed (LLD §74, §75)
    ├── fortinet.fortios.v7.yaml
    ├── crowdstrike.falcon.v3.yaml
    ├── scalefusion.audit.v1.yaml
    ├── generic.cef.yaml
    ├── generic.rfc5424.yaml
    └── manifest.json

  LOADING
    · signature verified before load (LLD §75 signed updates)
    · loaded into a NEW registry; the old one serves traffic until
      the new one is ready, then an ATOMIC SWAP with drain
    · ⚠ NO PROCESS RESTART
```

```
  HOT RELOAD IS NOT A CONVENIENCE HERE.

  In LLD §5's monolith, a restart stops ingestion, not just parsing.
  The collector's only uncontrolled loss class is STREAM (01 §4.3),
  and STREAM loss happens on every restart. Parser updates will be
  frequent. They must not cost packets.

  Managed by the Parser Manager in the control plane (LLD §4.2,
  doc 10).
```

### 5.2 Where the auto-parser fits

LLD §83 places the auto-parser in Phase 2. Until then, unknown sources fall to
generic parsers and dead-letter.

```
  WHEN IT ARRIVES (../autoparser/)
    L0–L5 observes structure across many records, clusters with
    Drain, and PROPOSES a parser definition.

    PROPOSE  →  CONFIRM  →  FREEZE

    proposed   used, flagged, facts carry reduced confidence (06 §7)
    confirmed  a human accepted it in the UI (LLD §63 "Parsing")
    frozen     promoted into the next signed bundle

  It is what makes an MSSP catalog tractable, because every customer
  has three sources nobody has ever seen.
```

---

## 6. Parsing makes the data bigger

A sizing fact that is easy to get backwards, and one of the inputs to ESC-1.

```
  raw syslog line                          1,104 bytes
  parsed, named fields, JSON               1,330 bytes    +20%

  AND Parse() RETURNS A SLICE — one CloudTrail page of 50 records
  becomes 50 parsed events from one raw record. For API sources the
  multiplier is on COUNT as well as size.

  UNDER LLD §15 THIS IS PERSISTED TO DISK. Under ESC-1's proposed
  resolution it exists only in flight, between worker pools, and
  never touches storage.
```

---

## 7. Dead-letter — never drop, always account

LLD §20 is explicit that parser failures must not drop events.

```
  A RECORD DEAD-LETTERS WHEN
    · detection returned PLAIN and no parser matched
    · a bound parser threw or returned no usable fields
    · JetStream max-deliver was exhausted (02 §4.2)
    · the per-record parse timeout expired (§8)

  RECORD (LLD §20)
    { event_id, connector, parser, reason, attempts,
      first_seen, last_attempt }

  PROPOSED ADDITIONS
    · SAMPLING. The first 1,000 distinct shapes per connector per
      day, not every record. A source emitting 3 EPS of an
      unparseable format must not fill 30 GB with evidence of one
      fact. LLD §70 gives dead-letter 7 days; without sampling that
      budget is consumed in hours.
    · a payload EXCERPT (200 bytes) rather than the full payload,
      so the dead-letter store is not a shadow copy of raw.
```

```
  ⚠ PARSE FAILURE IS NOT A COVERAGE GAP.

  A gap means WE DID NOT SEE THE DATA. A parse failure means we saw
  it and could not read it — the record is in dead-letter and will
  be reprocessed when the parser is fixed (02 §7.2).

  Conflating them makes a parser bug look like a collection outage
  and sends the investigation to the wrong team. Separate counters,
  separate UI treatment.
```

---

## 8. Considerations

**Anchor every pattern, and use the context timeout.** An unanchored regex over a
4 KB log line backtracks, and a pathological input turns 300 µs into 300 ms. At
LLD §17's 16 parser workers that is a stalled pool, a growing buffer, and
eventually STREAM drops at the gateway. Anchoring, possessive quantifiers, and a
per-record deadline on the `ctx` that LLD §18 already passes.

**Per-record CPU budget.** At 10,000 EPS across 8 active workers that is 1,250
records/sec/worker — 800 µs wall clock, of which parsing must fit in ~300 µs.
Generous for a grok pattern, tight for a chain of unanchored regexes.

**The registry swap must be atomic and the old registry must drain.** In-flight
records were matched against the old registry; completing them under the new one
produces half-old, half-new field sets.

**Do not parse what will be discarded.** If the noise policy (`06 §5`) drops 70%
of firewall traffic logs, evaluate the drop predicate on the raw line where it
can be. Parsing and then discarding spends the collector's most expensive budget
on data that was never going to be used.

**Recover at the worker boundary.** LLD §5's monolith means a parser panic on
malformed input kills the collector. `recover()` per worker, dead-letter the
record, continue. This is not optional.

**Version parsers, never mutate them.** `fortinet.fortios.v7` and
`fortinet.fortios.v71` coexist, and parsed events carry the parser version. That
is what makes the replay diff in `02 §10.2` meaningful — you can tell which facts
came from the defective version.

**Timestamp extraction is where most silent wrongness lives.** Two-digit years,
missing timezones, local time from a device in another region, and RFC3164 which
omits the year entirely. Every parser declares its timestamp strategy explicitly,
and a record whose extracted time is more than 24 h from `received_at` is flagged
rather than accepted quietly.

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Unanchored regex, pathological input | Worker pool stalls, buffer fills, STREAM drops upstream | Anchoring + per-record `ctx` deadline |
| Panic on malformed input | Whole collector dies (monolith) | `recover()` per worker → dead-letter |
| Parser update requires restart | STREAM data lost on every content update | Atomic hot swap, §5.1 |
| Parsers compiled into the binary | Every connector is a rebuild and a re-qualification | Signed content bundles, §5 |
| Parse-then-discard | Largest budget spent on data that is thrown away | Pre-parse drop predicates |
| Dead-letter unsampled | 30 GB filled in hours by one source | 1,000 shapes/connector/day + excerpts |
| Parse failure counted as a coverage gap | Parser bug reads as a collection outage | Separate counters, §7 |
| Non-atomic registry swap | Mixed field sets in one batch | Swap with drain |
| Detection never run on bound connectors | A device upgrade silently dead-letters everything | Sampled continuous detection, §4.2 |
| Timestamp misparsed | Facts land at the wrong time; ordering wrong | Explicit strategy + 24 h skew flag |

---

## 10. Example: Meridian

### 10.1 The parse profile of COL-mer-01

```
  CONNECTOR              FORMAT        PARSER                  RATE
  ─────────────────────────────────────────────────────────────────
  con-fortigate-dc-01    RFC3164 KV    fortinet.fortios.v7    6,200
  con-fortigate-dc-02    RFC3164 KV    fortinet.fortios.v7    3,100
  con-paloalto-perim     CSV           paloalto.panos.v11     1,400
  con-nsx-dfw            JSON          vmware.nsx.v4            180
  con-fortimanager       JSON          fortinet.fortimanager     ~0

  overlook_events_parsed_total       99.97%
  overlook_parse_failures_total       ~3.3 EPS

  91% of dead-lettered records are ONE shape: FortiGate
  `type=event subtype=wad` from a WAN-optimisation module nobody
  enabled deliberately. Same 3 EPS for six weeks.

  → the right response is a noise drop rule, not a parser. Writing
    a parser would produce 285,000 correctly parsed records per day
    describing a feature that is not in use.
```

### 10.2 A format change, caught by sampled detection

```
  02:10  FortiOS upgraded 7.2 → 7.4 on dc-02 in a maintenance window.

  02:11  sampled detection on con-fortigate-dc-02 reports JSON where
         it has reported RFC3164 for eleven months

  02:11  BINDING/DETECTION MISMATCH raised
           bound     fortinet.fortios.v7   expects RFC3164 KV
           detected  JSON
         UI: "con-fortigate-dc-02 · parse rate 99.9% → 4.1% ·
              format changed · likely device upgrade"

  02:14  fallback to generic.json. It maps src_ip, dst_ip and action
         by convention, so entity and relationship extraction
         continues AT REDUCED CONFIDENCE. Facts still flow.

  09:30  fortinet.fortios.v74 bundled, hot-loaded, no restart
  09:31  parse rate back to 99.96%
  09:40  replay from OVERLOOK_RAW

  ⚠ COL-mer-01's window is ~10.6 hours, so the full 7 h 20 m was
    recoverable. On COL-mer-03 (4.4 h — see 02 §10.1) it would not
    have been.

  WITHOUT THE FALLBACK
    7 h 20 m entirely unparsed → 14 attack paths through the dc-02
    segment become UNKNOWN rather than absent. Correct, but useless.

  WITH IT
    84% of relationships still extracted, confidence 0.94 → 0.61,
    visible on every fact.

  A degraded parser that says so beats no parser. It also beats a
  confident parser that is wrong — which is what would have happened
  had the v7 parser silently coerced JSON into its expected fields.
```

---

*Next: [Normalization Engine](04-normalization-engine.md)*

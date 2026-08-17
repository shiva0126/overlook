# L0/L1 — Format Detection and Structure Extraction

**Series:** [Auto-parser](00-index.md)

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

Answer *what shape is this?* and, for anything structured, extract the fields for free.

This is the cheapest and most reliable layer, and it disposes of the majority of real sources. Everything harder — template mining, typing, semantic mapping — only applies to what survives it.

---

## 2. Position

```
  INPUT    a framed record from the receiver, plus the source
           registry's current belief about (src_ip, port)

  OUTPUT   format verdict with confidence
           for structured formats: a field map, extracted
           for unstructured: the raw payload, handed to L2

  FEEDS    L2 template mining (unstructured only)
           L3 field typing (both)
           the source registry (binding a source to a format)
```

---

## 3. The detection ladder

Ordered by decreasing certainty. First confident match wins.

```
  1  JSON
     parse attempt. Succeeds or does not. No ambiguity.
     → fields free. Nested objects flattened with dotted paths.
     ⚠ a JSON payload WRAPPED IN SYSLOG is common. Detect the
       syslog envelope first, then the JSON body.

  2  XML
     parse attempt. Same certainty as JSON.
     → fields free. Attributes and elements both extracted.

  3  CEF          ArcSight Common Event Format
     signature: "CEF:" + version + 6 pipe-delimited headers,
     then key=value extensions.
     → headers positional, extensions key-value. Fully specified.

  4  LEEF         QRadar Log Event Extended Format
     signature: "LEEF:" + version + 4 tab or pipe-delimited
     headers, then key=value.
     → same treatment as CEF.

  5  RFC 5424     structured syslog
     signature: "<PRI>1 " + ISO timestamp + hostname + app-name +
     procid + msgid + structured-data.
     → header fields free. STRUCTURED-DATA elements are key-value
       and also free. The MSG body is usually unstructured → L2.

  6  RFC 3164     BSD syslog
     signature: "<PRI>" + "Mmm dd hh:mm:ss" + host + tag.
     → header fields free, loosely. The body is free text → L2.

  7  CSV / TSV
     consistent delimiter count across a sample window, plus a
     plausible header row or a stable column count.
     → fields positional. Header inference is a separate problem.

  8  KEY-VALUE
     a high proportion of tokens matching key=value or key: value,
     with a consistent separator.
     → this is what Splunk auto-extracts, and it covers a large
       share of network device output.

  9  W3C / NCSA    web and proxy access logs
     positional, space-delimited, with a known column order that
     may be declared in a header directive.

 10  UNSTRUCTURED
     none of the above. Free text. → L2 template mining.
```

---

## 4. Detection needs a sample, not a line

```
  ONE LINE IS NOT ENOUGH.

  A single line of key-value output can look like CSV. A single
  JSON line inside an otherwise free-text stream proves nothing
  about the stream.

  METHOD
    buffer the first N records from an unseen (src_ip, port)
      N = 200, or 60 seconds, whichever first
    score each format against the WHOLE sample
    require agreement across ≥ 90% of the sample
    below that threshold → MIXED, which is its own verdict (§6)
```

---

## 5. Layered formats are the normal case

The single most common real-world shape, and the one naive detectors get wrong.

```
  <134>1 2026-08-17T09:14:22Z fw-dc1 FortiGate - - - {"srcip":"10.4.2.17", ...}
  └────────── RFC 5424 envelope ─────────────────────┘ └──── JSON body ────┘

  DETECTION MUST PEEL, NOT GUESS:
    1  detect the transport envelope (syslog)
    2  extract its header fields
    3  RE-RUN DETECTION on the MSG body
    4  repeat until the innermost layer is reached

  COMMON STACKS
    syslog → JSON              modern network devices
    syslog → CEF               security appliances
    syslog → key-value         FortiGate, many firewalls
    syslog → free text         legacy Unix, older devices
    JSON → base64 → JSON       cloud event wrappers
    JSON → embedded XML        Windows event forwarding

  Depth limit 4. Beyond that, treat the innermost as opaque.
```

---

## 6. The MIXED verdict

```
  A source that emits multiple formats on one channel is normal,
  not an error. A FortiGate sends traffic logs in key-value and
  system events in a different shape on the same port.

  MIXED handling
    · sub-classify by a discriminator token found in the sample
      (FortiOS: type=traffic vs type=event)
    · each sub-class gets its own detection verdict and its own
      downstream parser
    · the source registry records a source as having N sub-formats

  A detector that forces one verdict onto a mixed source produces
  a parser that works for the majority class and silently fails
  the rest — which shows up later as a field-presence anomaly,
  far from its cause.
```

---

## 7. What structure extraction gives, and what it does not

```
  GIVES
    field names and values, for free, with no inference
    nesting preserved as dotted paths
    types where the format carries them (JSON numbers, booleans)

  DOES NOT GIVE
    ✕ meaning. "srcip" is a name, not a semantic.
    ✕ canonical keys. A username field is not yet an IDENTITY.
    ✕ relationships. A parsed line is not yet an observation.

  L1 output is a BAG OF NAMED VALUES. Everything that makes it
  useful happens in L3 and L4.
```

---

## 8. Considerations

**Confidence must be recorded and must decay.** A source bound to a format at onboarding may change after a firmware upgrade. The binding carries a confidence and a TTL, and a sustained parse-rate drop invalidates it and forces re-detection (`../engines/03 §3.1`).

**Never let detection be silent.** A source that cannot be classified is a first-class Controller item — *"4,200 records/hour, unidentified"* — with sample viewing and manual override. Silently discarding unclassifiable input is the failure mode that produces a graph with a hole nobody knows about.

**Structured does not mean parsed.** JSON gives fields for free and tells you nothing about which of them matters. The temptation is to declare victory at L1 because the output looks tidy.

**Manual override must be first-class.** Some sources are genuinely unguessable — a proprietary device emitting a bespoke format. An operator who knows the answer should be able to say so, and that override should be shareable (`07`).

---

## 9. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Detection on one line | Wrong verdict from an unrepresentative sample | 200-record or 60-second sample window, 90% agreement |
| Envelope not peeled | Body treated as opaque text; all fields lost | Layered detection, depth 4 |
| Mixed source forced to one verdict | Minority class silently unparsed | MIXED verdict with sub-classification |
| Binding never re-evaluated | Firmware upgrade breaks parsing silently | Confidence with TTL; parse-rate collapse forces re-detection |
| Unclassified source discarded | Graph hole nobody knows about | First-class Controller item with samples |
| CSV header inferred wrongly | Every field shifted by one | Require a plausible header or a declared column order; otherwise positional with generated names |

---

## 10. Example: Meridian, four sources detected

```
  10.4.0.9:6514 — Palo Alto
    sample 200 records, 60 seconds
    layer 1: RFC 5424 envelope        confidence 0.99
    layer 2: CSV within MSG           confidence 0.96
             (PAN-OS emits comma-separated fields in the body)
    verdict: syslog5424 → csv
    → 74 positional fields extracted, named from the PAN-OS
      field-order definition in the fingerprint content

  10.4.7.2:6514 — FortiGate
    layer 1: RFC 5424 envelope        confidence 0.99
    layer 2: KEY-VALUE                confidence 0.98
    verdict: syslog5424 → kv
    → fields free: srcip, dstip, dstport, action, service, ...
    MIXED detected: type=traffic (94%) and type=event (6%)
    → two sub-classes registered, each parsed separately

  10.4.2.40:514 — Linux rsyslog
    layer 1: RFC 3164 envelope        confidence 0.97
    layer 2: UNSTRUCTURED             free text
    verdict: syslog3164 → unstructured
    → header fields free; body handed to L2 template mining

  10.4.9.22:514 — the unknown device
    layer 1: RFC 3164 envelope        confidence 0.88
    layer 2: no format above threshold
             kv scored 0.41 · csv 0.33 · unstructured 0.51
    verdict: UNIDENTIFIED
    → 4,200 records/hour quarantined
    → Controller: "unidentified source · view samples ·
      identify manually · ignore"
    → the operator recognises a legacy Juniper SRX and binds it
      manually. 14 hours of quarantined records replayed.
```

**Three of four sources were fully resolved at L1 with no inference at all.** Only the Linux host reaches template mining, and only the unknown device needed a human. That ratio is what makes the layer worth building first.

---

*Next: [Template mining](02-template-mining.md)*

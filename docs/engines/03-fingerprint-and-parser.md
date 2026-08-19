# E2 Fingerprint and E3 Parser

**Series:** [Engine documentation](00-index.md) · **v1:** not required — every v1 source returns JSON

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1. Purpose

**E2, Fingerprint**, answers *what is this?* For a connector the answer is declared. For a syslog line arriving from an unknown IP, it must be inferred.

**E3, Parser**, turns bytes into a typed record using a grammar selected by E2's answer. It is the most open-ended component in the collector and carries a permanent content cost.

Both exist only for stream and file sources. Neither is needed while every source is an authenticated JSON API — which is why they are the largest single saving in the v1 scope cut (`../10 §4.2`).

---

## 2. Position

```
  INPUTS
    journaled record + provenance envelope

  OUTPUTS
    E2 → record annotated with (vendor, product, format, version)
    E3 → typed record with extracted fields
       → quarantine bucket on failure
       → evidence store (raw, hashed, encrypted, TTL'd)

  DEPENDS ON
    the content pipeline — fingerprint signatures and parser
    grammars are shipped content, not code
```

---

## 3. Mechanics

### 3.1 E2 — fingerprinting

```
  TRIVIAL CASE (PULL, AGENT)
    the connector or agent declares its schema. No inference needed.

  THE REAL CASE (STREAM)
    a syslog line arrives from 10.4.9.22:514 and nothing says whether
    it is a FortiGate, a Palo Alto or a Linux host.

  METHOD
    1  buffer the first N samples from an unseen (src_ip, port)
    2  score them against a signature set — structural markers,
       field ordering, vendor tokens, version strings
    3  above a confidence threshold, bind the source in the registry
       with a TTL
    4  below it, quarantine and surface as UNIDENTIFIED SOURCE

  RE-IDENTIFICATION TRIGGER
    a sustained parse-rate drop invalidates the binding and forces
    re-fingerprinting. This is how a firmware upgrade is caught.
```

The source registry is a cache keyed on `(src_ip, port)`, and it must be manually overridable — some sources are simply unguessable, and an operator who knows the answer should be able to say so.

### 3.2 E3 — parsing

**Declarative, not per-vendor code.** A grammar runtime executing shipped definitions, not 400 hand-written Go parsers.

```
  HANDLER FAMILIES
    CEF / LEEF          structured, well-specified
    RFC5424 / RFC3164   syslog framing plus a free-text payload
    JSON                trivial
    key=value           common in firewall logs
    regex-grammar       the fallback, and the one that rots

  ONE PARSER PER SOURCE TYPE — strictly 1:1
    borrowed from Chronicle (../06 §2.2). It makes parser health
    measurable per source, ownership unambiguous, and prevents the
    "one clever parser handles four vendors" mess.
```

### 3.3 Never silently drop

The most damaging failure available to this engine.

```
  parse failure → QUARANTINE, with a retained sample
                → grouped by failure signature
                → counted, surfaced, replayable

  A parser that silently discards 40% of records produces a graph
  that is 40% wrong and looks perfectly healthy on every dashboard.
```

### 3.4 Two health signals, not one

```
  PARSE RATE        parsed / received, against a per-source baseline
                    catches: format changed, wrong parser selected

  FIELD PRESENCE    per-field population rate, against a baseline
                    catches: the subtler failure where the record
                    still parses but a field was renamed

  Example: src_user populated in 94% of records last week, 3% today.
  Parse rate is still 100%. Nothing looks broken. Only field-presence
  monitoring catches it.
```

### 3.5 The evidence store

E3 is where the raw form is last available in a retrievable shape. It is hashed, encrypted and written with a TTL; the hash becomes `evidence_ref` on every fact derived from it.

```
  content-addressed        sha256 of the raw record
  encrypted at rest        customer KMS-wrapped key
  TTL                      default 90 days, operator-configurable
                           with the disk cost shown live
  access                   only through the Controller, only with
                           RBAC, every retrieval audited
```

---

## 4. Considerations

**Grammars are permanent content, not a project.** Google maintains hundreds of parsers and adds continuously (`../08 T9`). Every vendor log format we support is an ongoing maintenance liability. This is the strongest argument for depth over breadth: six deep API connectors carry no parser burden at all.

**Version drift is the normal case.** Vendors change formats in patch releases without notice. The system must assume the parser is wrong rather than assume the data is wrong.

**Partial parses are useful.** A record where 12 of 15 fields extract cleanly should emit those 12 with an incompleteness flag, not fail wholesale.

**Timestamps are the hardest field.** Timezone-less local times, epoch seconds versus milliseconds, and vendor-specific formats. Half of any correlation is temporal, so a mis-parsed timestamp is worse than a missing one.

**Debugging without the data.** Support engineers cannot receive customer logs. Everything needed to fix a grammar — samples, failure signatures, replay, diff — must work locally, operated by the customer.

**Sampling policy for quarantine.** Retaining every failed record during a total parse failure would fill the disk. Retain N per failure signature, plus counts.

---

## 5. Failure modes

| Failure | Behaviour |
|---|---|
| Unknown source | Quarantine, surface as "4,200 records/hour, unidentified", offer manual identification |
| Misidentification | Wrong grammar → low parse rate → automatic re-fingerprint trigger |
| Format change after upgrade | Parse rate collapse → attention item → content update → journal replay |
| Field renamed, format unchanged | **Only field-presence monitoring catches this.** Parse rate stays 100% |
| Grammar regression from a content update | Automatic rollback if parse rate drops >5% post-update |
| Quarantine fills the disk | Sample-and-count rather than retain-all |

---

## 6. Contracts

```
  MUST GUARANTEE
    no record is ever silently discarded
    every parse failure is counted, sampled and replayable
    evidence is written before any derived fact references it
    parser and source type are strictly 1:1
    grammars are versioned content, independently rollback-able
```

---

## 7. Scope

```
  BUILD IN V1
    nothing

    Every v1 source — AWS IAM, Entra Graph, GitHub, Okta/AD LDAP,
    the agent, AI org APIs — returns structured JSON or LDAP objects.
    E4 Normalizer consumes them directly.

  BUILD WHEN THE FIRST STREAM SOURCE ARRIVES
    fingerprint engine + source registry
    grammar runtime with CEF/LEEF/RFC5424/kv handlers
    quarantine, parse-rate and field-presence monitors
    journal replay integration

  NOTE
    the EVIDENCE STORE is needed in v1 even without E3 — connector
    responses (policy documents, trust policies) are evidence and
    must be hashed and retained. Build it beside E4 instead.
```

---

## 8. Example: Meridian, the FortiGate that changed overnight

Meridian is a future-state deployment for this engine — it has four firewalls, so E2 and E3 exist here even though they do not in v1.

```
  ESTABLISHED STATE

    source registry, COL-DC1
      10.4.0.9:6514   → paloalto / PAN-OS 11.1     parse 99.1%
      10.4.0.11:6514  → paloalto / PAN-OS 11.1     parse 98.7%
      10.4.7.2:6514   → fortinet / FortiOS 7.4     parse 98.4%
      10.4.7.3:6514   → fortinet / FortiOS 7.4     parse 98.9%
      10.4.2.40:514   → linux / rsyslog            parse 97.2%
      10.4.9.22:514   → UNIDENTIFIED  4,200 rec/h  ← see below

  THE UNIDENTIFIED SOURCE
    A device nobody documented is sending syslog. E2 buffered samples,
    scored them against the signature set, and got no match above
    threshold. Rather than guessing, it quarantined and raised:

      ⚠ UNIDENTIFIED SOURCE  10.4.9.22:514 · 4,200 rec/h
        [ view samples ] [ identify manually ] [ ignore this source ]

    The operator looks at the samples, recognises a legacy Juniper
    SRX, and binds it manually. Parsing begins. 14 hours of
    quarantined records are replayed from the journal.

  THE OVERNIGHT UPGRADE — 03:40

    Meridian's network team upgrades fw-branch-02 (10.4.7.3) from
    FortiOS 7.4 to 7.6 during a change window. The log format changes:
    several fields are reordered and one is renamed.

  03:41  E3 parse rate for that source drops 98.9% → 4%
  03:44  sustained drop crosses the threshold
         → binding invalidated, E2 re-fingerprints
         → best match is still "fortinet", but version scoring is
           ambiguous; confidence sits below threshold
         → source marked DEGRADED rather than silently mis-parsed
  03:45  quarantine begins accumulating, grouped by failure signature
         (3 distinct signatures, 40 samples retained per signature)

  06:12  Controller attention inbox, seen by the operator:

         ● PARSE RATE COLLAPSE                             fw-branch-02
           98.9% → 4% since 03:41
           Likely a format change after the overnight firmware upgrade.
           41,000 records quarantined · 3 failure signatures
           Graph consequence: reachability edges from this device
           are STALE. Nothing tombstoned — no coverage window emitted.
           [ view samples ] [ report to Overlook ] [ re-identify ]

  THE FIX
    The operator clicks "report to Overlook", which generates a
    REDACTED bundle: failure signatures, field-shape summaries and
    structurally-anonymised samples. No customer data leaves.

    Overlook ships content v2026.08.15 with a FortiOS 7.6 grammar.
    Staged rollout: canary → 10% → 100%.

    The operator replays: source fw-branch-02, last 72h, dry run.
    Diff shows 41,000 records now parse, yielding 340 aggregate
    records and 12 changed reachability edges.
    Applied. The gap closes.

  THE QUIETER FAILURE, TWO WEEKS LATER

    A PAN-OS policy change stops populating the source-user field.
    Parse rate stays at 99.1% — every record parses perfectly.

    FIELD PRESENCE monitoring catches it:
      src_user populated 94% last week → 2% today

    Without that second signal, Meridian's firewall data would have
    silently stopped contributing identity attribution to the graph,
    and every path through those edges would have quietly lost a hop.
```

---

*Next: [Normalizer and Enrichment](04-normalizer-and-enrichment.md)*

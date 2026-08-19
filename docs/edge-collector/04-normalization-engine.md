# 4 — The Normalization Engine

**Series:** [The Edge Collector](00-index.md) · **LLD:** §21

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `LLD-edge-collector-v1.0` is the implementation boundary. Divergences are
> recorded in [13-escalations.md](13-escalations.md).
> Budget: **1.0 vCPU · 4 GB RAM** (`00 §4`).

---

## 1. Purpose

Map vendor field names onto one common schema, so that nothing downstream needs
to know which product produced a record.

The LLD separates this from parsing (§7 has `internal/parser/` and
`internal/normalize/` as distinct packages), and that separation is right. They
answer different questions:

```
  PARSING          "what are the fields in these bytes?"
                   vendor-specific · format-specific · fragile ·
                   changes when a device is upgraded

  NORMALIZATION    "what do those fields MEAN?"
                   vendor-neutral · stable · changes only when the
                   common schema changes
```

Collapsing them means every parser has to know the common schema, and a schema
change means editing a hundred parsers.

---

## 2. Position

```
  INPUTS
    parsed events (03), with vendor field names

  OUTPUTS
    normalized events on the LLD §21 schema → enrichment (05)
    unmappable fields → the extension namespace (§5)
    normalization telemetry → overlook_events_normalized_total

  CONSUMED BY
    05 enrichment engine — which looks up on source.ip, user.name,
       asset.hostname and would need a per-vendor branch without this
```

---

## 3. The common schema

LLD §21 defines it. It is ECS-shaped, which is the right lineage — Elastic Common
Schema is the closest thing the industry has to a standard, and adopting its
field names means every parser author already knows them.

```
  event.id · event.kind · event.type · event.category
  event.action · event.outcome · event.severity
  timestamp
  source.ip · source.port
  destination.ip · destination.port
  network.protocol · network.transport
  user.id · user.name
  asset.id · asset.name · asset.hostname · asset.type
  process.pid · process.name · process.hash
  file.name · file.path · file.hash
  cloud.provider · cloud.account · cloud.region · cloud.resource_id
  application.id · application.name
  database.name
  security.rule_id · security.rule_name
```

### 3.1 Mapping is a table, not code

```
  FortiGate                    →  common schema
    srcip                      →  source.ip
    srcport                    →  source.port
    dstip                      →  destination.ip
    dstport                    →  destination.port
    action    (accept|deny)    →  event.action + event.outcome
    devname                    →  asset.hostname       (the observer)
    user                       →  user.name
    proto     (6|17)           →  network.transport

  CrowdStrike                  →  common schema
    LocalIP                    →  source.ip
    RemoteIP                   →  destination.ip
    ComputerName               →  asset.hostname
    UserName                   →  user.name
    SHA256HashData             →  process.hash
    FileName                   →  process.name

  Two products. One schema. Enrichment reads source.ip and neither
  knows nor cares which produced it.
```

**The mapping ships in the same signed bundle as the parsers** (`03 §5.1`), for
the same reason: a new source must not require a rebuild.

### 3.2 Value normalization, not only field normalization

The part most easily missed, and the source of most downstream wrongness.

```
  FIELD NAMES ARE THE EASY HALF. VALUES DIVERGE TOO:

    event.outcome     FortiGate "accept"/"deny"
                      Palo Alto "allow"/"drop"/"reset-both"
                      Windows   "4624"/"4625"
                      AWS       "Success"/"Failure"
                      → ONE enum: success · failure · unknown

    network.transport 6 · "6" · "tcp" · "TCP" · "Tcp"
                      → lowercase IANA name

    timestamp         epoch seconds · epoch millis · RFC3339 ·
                      RFC3164 (no year!) · local time, no zone
                      → RFC3339 UTC, always

    severity          0–10 · 1–5 · "critical" · "P1" · "high"
                      → ONE ordinal scale, and record the original
                        in the extension namespace

  A rule that maps the field but not the value produces
  `event.outcome: "reset-both"` in a schema whose consumers test
  for "failure". The test silently never matches.
```

---

## 4. Required fields and what happens without them

```
  ALWAYS REQUIRED
    event.id · timestamp · event.category · event.action

  A record missing any of these cannot be reasoned about at all
  and goes to overlook.deadletter with reason
  `normalization_missing_required`.

  ⚠ THIS IS A DIFFERENT FAILURE FROM A PARSE FAILURE.
    Parse failure    = we could not read the bytes
    Normalize failure = we read them, and they do not describe
                        a security event we understand

    Same destination, different reason code, different fix. A parse
    failure needs a parser; a normalization failure usually needs a
    mapping rule, and sometimes means the source is genuinely not
    security-relevant and should be filtered at the connector.
```

---

## 5. The extension namespace

```
  NOT EVERY USEFUL FIELD FITS LLD §21, and dropping the remainder
  loses information the Fact Engine needs.

    FortiGate  policyid, vdom, sessionid
    AWS        requestParameters, userIdentity.sessionContext
    GitHub     workflow_run.head_branch, actor_location
    Entra      conditionalAccessStatus, riskLevelDuringSignIn

  → carried under a vendor-namespaced extension:
      ext.fortinet.policyid
      ext.aws.userIdentity.sessionContext.attributes.mfaAuthenticated

  RULES
    · extension fields are NEVER read by generic downstream logic
    · they ARE read by fact extraction rules, which are per-source
      anyway (06 §4)
    · they are subject to privacy classification like any other
      field (07 §4) — an extension namespace is not a privacy
      loophole
    · anything read by more than three sources' rules is a
      candidate for promotion into the common schema
```

`ext.aws...mfaAuthenticated` is a good illustration: it looks like vendor trivia
and it is what determines whether an assumed-role relationship is protected by
MFA, which changes an attack path's weight.

---

## 6. Considerations

**Normalization is pure and should be trivially testable.** In → out, no I/O, no
network, no cache, no clock. That makes it the one stage where a golden-file
corpus per source is cheap and complete: one file of raw records, one of expected
normalized output, run on every bundle change. This should be the strictest
tested module in the collector because it is the one where errors are silent.

**Timezone handling belongs here, not in the parser.** The parser extracts the
timestamp as written; normalization applies the connector's configured timezone
and converts to UTC. Putting it in the parser means the timezone becomes parser
configuration, and the same parser then cannot serve two devices in different
regions.

**Do not enrich here.** The temptation is constant — the asset lookup would fit
neatly. It must not, because enrichment needs caches and I/O, and this stage's
purity is what makes it testable. LLD §7's package boundary already says so.

**Preserve the original where you transform.** `event.severity` becomes an
ordinal; `ext.<vendor>.original_severity` keeps what the device said. When an
engineer disputes a finding, the first question is "what did the firewall
actually log?", and the answer must not require a replay.

**One schema version, carried on the record.** `schema_version` on every
normalized event. Without it a bundle upgrade that changes a mapping is
indistinguishable from a source behaving differently, and there is no way to
tell which facts predate the change.

---

## 7. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Field mapped, value not normalized | Downstream tests silently never match | Value enums, §3.2 |
| Timestamp not converted to UTC | Facts land at the wrong time; ordering wrong across sources | UTC always; timezone is connector config |
| Timezone handled in the parser | One parser cannot serve two regions | Handle it here |
| Unmappable fields dropped | Fact extraction loses the fields it needs most | Extension namespace, §5 |
| Extension namespace exempt from privacy | Plaintext identifiers ship unnoticed | Classified like any field, §5 |
| Normalization failure counted as parse failure | Wrong team investigates | Distinct reason codes, §4 |
| Enrichment creeping into this stage | Loses purity, loses testability | Package boundary, LLD §7 |
| No `schema_version` on records | A mapping change is indistinguishable from source drift | Version every record |
| Original values discarded | Disputes require a replay to answer | Keep originals in `ext.` |

---

## 8. Example: Meridian

### 8.1 One FortiGate record, normalized

```
  PARSED (vendor fields)
    date=2026-08-18 time=09:14:22 devname="FGT-DC-01"
    srcip=10.4.22.81 srcport=51422 dstip=10.9.1.40 dstport=443
    proto=6 action=accept policyid=42 vdom="root"
    user="jsmith" sessionid=8827411

  NORMALIZED
    event.id            evt-019289
    timestamp           2026-08-18T09:14:22Z    ← +05:30 applied
                                                  from connector config
    event.kind          event
    event.category      network
    event.type          connection
    event.action        network_flow
    event.outcome       success                 ← from "accept"
    event.severity      3                       ← FortiGate has none;
                                                  policy default
    source.ip           10.4.22.81
    source.port         51422
    destination.ip      10.9.1.40
    destination.port    443
    network.transport   tcp                     ← from proto=6
    user.name           jsmith
    asset.hostname      FGT-DC-01               ← the OBSERVER
    schema_version      1.0

    ext.fortinet.policyid    42
    ext.fortinet.vdom        root
    ext.fortinet.sessionid   8827411
    ext.fortinet.original_action  accept
```

**`asset.hostname` holding the observer rather than the subject is a real
ambiguity in LLD §21.** A firewall record has three assets — the source host, the
destination host, and the firewall that saw it. One `asset.*` group cannot hold
all three. The convention above (`asset.*` = observer, endpoints in `source.*`
and `destination.*`) works, but it must be *written down*, because the opposite
convention is equally natural and two parsers written by two people will
disagree.

### 8.2 A value-normalization defect, and how it hid

```
  A Palo Alto mapping shipped with

      action → event.outcome, passthrough

  Palo Alto emits allow · deny · drop · reset-both · reset-client.
  The FortiGate mapping emits success · failure.

  WHAT BROKE
    the Fact Engine's rule for blocked connections tests
      event.outcome == "failure"
    Palo Alto blocks emit "drop" and "reset-both".
    → NO blocked-connection facts from the perimeter firewall
    → the perimeter looked permissive: 1,400 EPS of traffic and
      zero denies recorded

  WHY IT WENT UNNOTICED FOR NINE DAYS
    parse rate 99.99%. Normalization rate 100%. Every counter in
    LLD §50 was green. Nothing failed — the values were simply
    wrong, and no metric measures meaning.

  HOW IT WAS FOUND
    an analyst asked why the perimeter firewall had never denied
    anything.

  THE FIX, AND THE REAL LESSON
    map the enum, and add a golden-file test per source that asserts
    on VALUES rather than on success rates. §6's testability
    argument is not theoretical; this is the class of bug it exists
    to catch, and it is invisible to every metric the LLD specifies.
```

---

*Next: [Enrichment Engine](05-enrichment-engine.md)*

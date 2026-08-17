# L3 — Field Typing

**Series:** [Auto-parser](00-index.md)

---

## 1. Purpose

Answer *what is this token?* for every variable position that L1 extracted or L2 mined.

```
  "10.4.2.17"                    → IPv4
  "priya.s@meridian.com"         → email
  "CORP\priyas"                  → domain-qualified account
  "arn:aws:iam::123:role/Deploy" → AWS ARN
  "2026-08-17T09:14:22Z"         → timestamp, RFC3339
  "4122"                         → integer, probably a port
  "a3f9c2..."                    → hex, 64 chars, probably SHA-256
  "/Users/priya/work"            → filesystem path, POSIX
```

Typing is what turns a bag of strings into candidates for canonical keys, and it is the last purely mechanical layer. L4 needs types to do anything at all.

---

## 2. Position

```
  INPUT    named fields (L1) or variable positions with sample
           values (L2)
  OUTPUT   a type verdict per field, with confidence, plus derived
           facts (is it a canonical key candidate? which scheme?)
  FEEDS    L4 semantic mapping
```

---

## 3. Three signals, combined

No single signal is sufficient. Typing works by agreement.

```
  SIGNAL 1 — SHAPE
    regex and format validators against the value itself
    → strong for structured types (IP, UUID, ARN, timestamp)
    → weak for anything that looks like a word

  SIGNAL 2 — DISTRIBUTION
    the statistical behaviour of the field ACROSS SAMPLES
    → cardinality, uniqueness ratio, value-length distribution,
      character-class distribution, stability over time
    → this is what distinguishes a username field from a
      free-text field, when both are just words

  SIGNAL 3 — CONTEXT
    the field NAME, if it has one, and the surrounding template
    → "srcip" is a strong prior. "user" is a strong prior.
    → in mined templates there is no name, but the surrounding
      constant tokens are: "for invalid user <*> from <*>"
      tells you a great deal about positions 6 and 8
```

**Shape alone is the classic mistake.** A regex that matches `\d+` types every integer identically, and a port, a byte count, a UID and a process ID are entirely different things downstream.

---

## 4. The type lattice

Types are hierarchical. A verdict may sit at any level, and confidence generally falls as specificity rises.

```
  SCALAR
    integer · float · boolean · string · timestamp · duration

  NETWORK
    ipv4 · ipv6 · cidr · mac · port · fqdn · dns_name · url · uri

  IDENTITY
    email · upn · sam_account · dn · uuid · sid · principal_arn
    username_bare        ← the hard one, §5

  CLOUD
    aws_arn · azure_resource_id · gcp_resource_name ·
    account_id · region · availability_zone

  SECURITY
    hash_md5 · hash_sha1 · hash_sha256 · cve_id · certificate_serial
    credential_pattern   ← never the value, only the shape (§7)

  PATH
    posix_path · windows_path · unc_path · s3_uri · registry_key

  ENUM
    a closed set of values observed — action, severity, protocol
    → detected by low cardinality and high stability
```

---

## 5. The hard cases

Where typing genuinely fails, and what to do about it.

```
  BARE USERNAMES
    "priyas" is a username. So is "backup". So is "svc-deploy".
    None of them is distinguishable from an English word by shape.

    SOLUTION — distribution plus context:
      · cardinality in the hundreds-to-thousands, not millions
      · stable set membership over time (the same values recur)
      · high overlap with values seen in a KNOWN identity field
        from another source
      → that last signal is the strong one. If 84% of this field's
        distinct values also appear as sAMAccountName in the AD
        connector's output, it is an account field.

  INTEGERS
    port · uid · pid · byte count · session id · error code
    → distribution: ports cluster at known values and cap at 65535;
      uids cluster low; byte counts have a long tail
    → context: the constant token before it in the template

  HOSTNAMES vs FREE TEXT
    "fw-branch-02" vs "connection reset"
    → distribution: hostnames are stable, recur, and overlap with
      known ASSET names from other sources

  OPAQUE IDENTIFIERS
    "8f14e45f-ceea-467a" is a UUID by shape and could be a session,
    a request, an object or a correlation id.
    → shape gives the type; only L4 gives the meaning, and often
      the answer is "none, ignore it"
```

**The cross-source overlap signal is our advantage.** A SIEM types fields in isolation. We already hold an entity store populated from authoritative sources, so we can ask *"do these values look like things we already know?"* — which is a far stronger signal than any regex.

---

## 6. Canonical key candidacy

The output L4 actually consumes.

```
  For each typed field, decide: could this be a CANONICAL KEY,
  and under which scheme? (../13-contracts.md Part IV)

  email          → email:<lowercased>              PRIORITY 1
  upn            → upn:<lowercased>                PRIORITY 4
  sam_account    → sam:<domain>\<name>             PRIORITY 5
  principal_arn  → arn:<normalised>                PRIORITY 1
  uuid + context → adguid: or idp:                 PRIORITY 2-3
  fqdn           → fqdn:<lowercased, dot-stripped> PRIORITY 3
  mac            → mac:<normalised>                PRIORITY 4
  ipv4 + time    → ip:<addr>@<date>                PRIORITY 5, WEAK

  CANDIDACY IS NOT ASSIGNMENT.
  L3 says "this field could be an identity canonical key at
  priority 5." L4 decides whether this template actually asserts
  something about that identity, and E6 does the resolution.
```

---

## 7. Credential shapes — detect, never capture

```
  Some values are recognisable as credentials by shape:
    AKIA[0-9A-Z]{16}          AWS access key id
    ghp_[A-Za-z0-9]{36}       GitHub PAT
    sk-[A-Za-z0-9]{48}        OpenAI-style key
    eyJ...                    JWT
    high-entropy strings of a plausible length

  WHEN ONE IS DETECTED
    · record the TYPE and the FIELD
    · record that a credential-shaped value appeared in this
      source's output
    · NEVER retain the value. Not in sample_values, not in the
      template store, not in the journal beyond its retention.
    · raise a FINDING: a source is logging credential material

  A device logging its own API tokens is a real and common
  problem, and the auto-parser is where it is discovered. But
  discovering it must not mean storing it.
```

---

## 8. Failure modes

| Failure | Consequence | Mitigation |
|---|---|---|
| Shape-only typing | Ports, UIDs and byte counts all become "integer" | Combine shape, distribution and context |
| Bare usernames typed as free text | Identity fields missed entirely | Cross-source overlap against the entity store |
| Free text typed as usernames | Junk canonical key candidates flood L4 | Cardinality and stability thresholds |
| Credential value retained | We become the breach we are reporting | Detect shape, record type, discard value |
| Type verdict never revisited | A field changes meaning after an upgrade | Verdicts carry confidence and are re-evaluated on drift |
| Over-specific verdict | "SHA-256" when it is any 64-char hex | Report the lattice level actually supported by evidence |

---

## 9. Considerations

**Typing runs on samples, offline, never at ingest.** Like mining, this is a learning activity in the scanner pool. Once a parser is frozen, field types are baked into it and no inference runs per record.

**Cross-source overlap is the strongest signal available and requires the graph to already exist.** Which means auto-parsing gets *better* as more authoritative sources are connected — typing a Linux syslog username field is far easier once AD is connected. Worth stating in onboarding: connect identity authorities first, and everything else parses better.

**Report confidence at the level the evidence supports.** Claiming `hash_sha256` for any 64-character hex string is over-specific; `hex_64` is honest and equally useful downstream.

**Enum detection is quietly valuable.** A field with 6 distinct values across 4 million records is an enum, and enums frequently carry the semantics L4 needs — `action=accept|deny`, `outcome=success|failure`. Detecting them cheaply gives L4 a strong prior.

---

## 10. Example: Meridian, typing three fields

### 10.1 A FortiGate key-value field — easy

```
  FIELD  srcip     (named, from L1 key-value extraction)
  SAMPLES  10.4.2.17 · 10.9.1.3 · 10.4.8.2 · ... (200)

  SHAPE         100% match IPv4                          → 0.99
  DISTRIBUTION  cardinality 1,204 / 4.1M · stable set    → consistent
  CONTEXT       field name "srcip" · known FortiOS field → 0.99

  VERDICT  ipv4, confidence 0.99
  CANONICAL KEY CANDIDACY
    ip:<addr>@<date> — PRIORITY 5, TIME-BOUNDED, WEAK
    → and L4 will almost certainly prefer to resolve through the
      DHCP lease data rather than use this directly
```

### 10.2 A mined variable position — the hard one

```
  TEMPLATE  "Failed password for invalid user <*> from <*> port <*> ssh2"
  POSITION  6
  SAMPLES   admin · root · test · oracle · postgres · priyas · ...

  SHAPE         all lowercase alphanumeric, 3-12 chars
                matches: username_bare, hostname_short, free word
                → ambiguous, 0.4 each

  DISTRIBUTION  cardinality 341 across 12,408 observations
                uniqueness ratio 0.027 — heavy repetition
                stable set: 88% of values recur across all 7 days
                → strongly suggests a bounded identifier set,
                  not free text                          → 0.75

  CONTEXT       preceding constant tokens "for invalid user"
                → 0.95 for username_bare

  CROSS-SOURCE OVERLAP                         ← THE DECIDING SIGNAL
                of 341 distinct values, 284 (83%) also appear as
                sAMAccountName in the AD connector's output
                → 0.97

  VERDICT  username_bare, confidence 0.94
  CANONICAL KEY CANDIDACY
    sam:<domain>\<value> — PRIORITY 5
    ⚠ the DOMAIN is not in this log line. L4 must supply it from
      source context (this host's domain membership), or the key
      is unresolvable.
```

**Without the AD connector, that field types at 0.75 and is probably rejected.** With it, 0.94 and accepted. Auto-parsing quality is a function of what else is already connected.

### 10.3 A credential, correctly handled

```
  SOURCE  a customer-built application forwarding to syslog
  TEMPLATE  "API request to <*> with token <*> returned <*>"
  POSITION  4
  SAMPLES   ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx · ...

  SHAPE     matches github_pat pattern                   → 0.98
  ENTROPY   high, consistent length                      → confirms

  ACTIONS TAKEN
    ✓ type recorded: credential_pattern / github_pat
    ✓ FINDING raised: "source app-billing is logging GitHub
      personal access tokens to syslog"  severity HIGH
    ✓ the template's position 4 is marked NEVER_RETAIN
    ✓ sample_values for that position purged and future capture
      suppressed
    ✓ journal retention for this source flagged for review

    ✕ NO VALUE STORED. Not in samples, not in the template, not
      in any finding. The finding says a token appeared, its type,
      and where — nothing more.

  Meridian's application team was logging tokens in production
  and did not know. The auto-parser found it while trying to
  work out what a field was.
```

---

*Next: [Semantic mapping](04-semantic-mapping.md)*

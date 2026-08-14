# E4 Normalizer and E5 Enrichment

**Series:** [Engine documentation](00-index.md) · **v1:** both required

---

## 1. Purpose

**E4, Normalizer**, maps vendor-specific fields onto the Overlook schema so that no downstream engine ever sees a vendor difference. After E4, an AWS IAM role, an LDAP group and an Okta app assignment are the same *kind* of thing described in the same vocabulary.

**E5, Enrichment**, attaches context the record does not carry — which asset an IP belongs to, who owns a resource, what business criticality applies.

They are documented together because they are the two stages that turn *data about records* into *data about things*, and because E5 depends on state that E4's output eventually creates.

---

## 2. Position

```
  INPUT   typed record in vendor vocabulary
          (from E3 for stream sources, or directly from the
           connector for JSON/LDAP sources)

  E4 OUT  normalized record in Overlook vocabulary
  E5 OUT  normalized record + attached context

  DEPENDS ON
    field mapping definitions   (shipped content, per source)
    the local entity store      (E5 reads it; E6 writes it)
    business context connectors (CMDB, HR) for ownership
```

---

## 3. Mechanics

### 3.1 E4 — what normalization actually does

Five distinct transformations, each with its own failure mode:

```
  1  FIELD MAPPING
     vendor field name → Overlook schema field
     declarative, per source type, shipped as content

  2  TIMESTAMP NORMALIZATION
     every time value → UTC, RFC3339
     handles: epoch seconds vs milliseconds, timezone-less local
     times, vendor formats, and the distinction between
     event-time and observation-time

  3  IDENTIFIER NORMALIZATION
     ARN, DN, UPN, SPN, resource ID, URI — each has a canonical form
     lowercase where case-insensitive, trim, strip display formatting,
     expand abbreviations, resolve relative forms

  4  ENUM CANONICALIZATION
     allow | permit | accept | ALLOWED  →  ALLOW
     the vocabulary is closed; unmapped values pass through flagged

  5  TYPE COERCION
     ports to int, IPs to normalized v4/v6, booleans from
     "true"/"yes"/"1"/"enabled"
```

### 3.2 The `extra` bag

Unmappable fields are **retained, not discarded**:

```
  A field we do not understand today may be the field we need
  next quarter. Keeping it costs bytes locally and nothing
  upstream — the Privacy Gate's allow-list means an unmapped
  field cannot leak (../11-privacy-gate.md §3.2).
```

### 3.3 E5 — enrichment lookups

```
  ASSET           IP or hostname → known asset
                  MUST be time-bounded: DHCP churn means an IP
                  maps to different assets over time
  IDENTITY        account string → known identity
  GEO / ASN       external IPs only
  THREAT INTEL    external indicators
  BUSINESS        owner, criticality, environment, cost centre
                  from CMDB and HR connectors
  TAGS / LABELS   cloud tags, Kubernetes labels
```

### 3.4 The circularity, and how to live with it

E5 reads the entity store. The entity store is populated by E6, which runs *after* E5. That is a genuine circular dependency.

```
  RESOLUTION: enrichment is BEST-EFFORT and EVENTUALLY CONSISTENT.

  The first record from a new asset arrives unenriched.
  By the tenth, the asset exists and enrichment succeeds.

  Nothing blocks. Nothing retries in-line. Facts built from
  unenriched records carry lower confidence and are naturally
  superseded as later observations merge in.
```

**Do not attempt synchronous enrichment with blocking lookups.** It converts a streaming pipeline into a request-response system and will not hold at volume.

---

## 4. Considerations

**Timestamps deserve disproportionate care.** A source reporting local time without an offset will silently misalign every temporal comparison against it. Record both source-reported time and receive time, and record which one downstream is trusting.

**Normalization is lossy on purpose, but log what you lose.** If an enum value is unmapped or a field is dropped, that must be counted and visible — otherwise vendor drift is invisible.

**Identifier normalization is where entity resolution succeeds or fails.** `CORP\priyas` and `corp\PriyaS` must produce the same normalized form, or E6 sees two people. Case folding rules are per-field-class and must be declared, not guessed.

**Enrichment must never be authoritative.** It attaches context; it does not assert identity. If enrichment guesses that IP 10.4.2.17 is `AST-oracle-01` and the binding is stale, E6 must be able to override it with a stronger signal.

**Time-bounded IP bindings are mandatory.** An IP→asset mapping without a validity window will attribute one host's activity to another after a DHCP lease change. Every binding carries a `valid_from`/`valid_to`.

**Business context is the highest-value enrichment and the most often missing.** Ownership is what converts a finding into an assigned ticket. A CMDB or HR connector is worth more per byte than almost anything else (`../03 §2.8`).

---

## 5. Failure modes

| Failure | Behaviour |
|---|---|
| Ambiguous timezone | Policy per source: source-declared → appliance TZ → flag. Never silently assume UTC |
| Unmapped field | Retained in `extra`, counted, surfaced when the rate changes |
| Unmapped enum value | Passed through flagged; monitored — indicates vendor drift |
| Enrichment lookup miss | Proceed unenriched, flag it. **Never block** |
| Stale IP→asset binding | Time-bounded bindings prevent misattribution; expired bindings simply do not match |
| Entity store unavailable | Enrichment degrades to none; pipeline continues with reduced confidence |
| Mapping content regression | Field-presence monitoring detects it; content rollback |

---

## 6. Contracts

```
  E4 MUST GUARANTEE
    every timestamp is UTC and unambiguous
    identifier normalization is deterministic and declared
    no field is discarded without being counted
    the predicate/enum vocabulary is closed and versioned

  E5 MUST GUARANTEE
    it never blocks the pipeline
    every attached context carries its source and confidence
    IP-derived bindings are time-bounded
    it never overrides a stronger identity signal
```

---

## 7. Scope

```
  BUILD IN V1
    field mapping runtime + mappings for the 6 v1 sources
    timestamp, identifier and enum normalization
    the extra bag
    asset and identity lookups
    tag/label enrichment (cloud tags matter for ABAC)

  DEFER
    geo / ASN            no external IP analysis in v1
    threat intel         not an exposure input
    business context     until a CMDB connector exists — but note
                         this is the highest-value deferral to revisit
```

---

## 8. Example: Meridian, one identity through E4 and E5

Priya S appears in eight sources. Here is what E4 and E5 do to three of them, before E6 ever sees them.

```
  SOURCE 1 — ACTIVE DIRECTORY  (LDAP, EDGE-DC1)

  RAW
    distinguishedName: CN=Priya S,OU=Engineering,OU=Users,DC=corp,DC=meridian,DC=com
    sAMAccountName:    priyas
    userPrincipalName: Priya.S@corp.meridian.com
    mail:              Priya.S@Meridian.com
    objectGUID:        {8F14E45F-CEEA-467A-9D29-2B1E4C4E1D3F}
    whenCreated:       20240314091422.0Z
    memberOf:          [CN=Developers,OU=Groups,...], ... 11 more

  E4 NORMALIZE
    dn        → "cn=priya s,ou=engineering,ou=users,dc=corp,dc=meridian,dc=com"
                (lowercased — DNs are case-insensitive)
    sam       → "corp\priyas"          (domain-qualified, lowercased)
    upn       → "priya.s@corp.meridian.com"
    mail      → "priya.s@meridian.com"  ← lowercased. CRITICAL:
                without this, "Priya.S@Meridian.com" from AD and
                "priya.s@meridian.com" from Entra are different
                strings and E6 sees two people.
    objectGUID→ "8f14e45f-ceea-467a-9d29-2b1e4c4e1d3f"
                (braces stripped, lowercased)
    whenCreated → "2024-03-14T09:14:22Z"
                (AD generalised time → RFC3339 UTC)
    memberOf  → 12 normalized DNs

  E5 ENRICH
    OU "Engineering" → department, from the OU-to-department map
    no CMDB connector at Meridian yet → owner unknown
    → flagged: "business context unavailable"


  SOURCE 2 — AWS IAM  (EDGE-CLD)

  RAW
    UserName: priya.s
    Arn:      arn:aws:iam::123456789012:user/priya.s
    CreateDate: 2024-04-02T11:03:00+00:00
    Tags: [{Key: "email", Value: "Priya.S@meridian.com"},
           {Key: "team",  Value: "payments"}]

  E4 NORMALIZE
    Arn       → "arn:aws:iam::123456789012:user/priya.s"
                (already canonical; account ID preserved as a
                 distinct field, not just embedded in the string)
    CreateDate→ "2024-04-02T11:03:00Z"
    tag.email → "priya.s@meridian.com"   ← same lowercase rule,
                same result as AD. This is the join.
    tag.team  → retained — it is load-bearing, because Meridian
                uses ABAC keyed on the team tag

  E5 ENRICH
    account 123456789012 → "Production" (from AWS Organizations)
    → environment: production, criticality inherited


  SOURCE 3 — THE AGENT ON LT-4471  (EDGE-DC1)

  RAW
    host: LT-4471
    user: priyas
    scan: mcp_config
    path: /Users/priyas/.claude/claude_desktop_config.json
    servers: [{name: "mcp-filesystem", args: ["/Users/priyas/work"],
               env_keys: ["GITHUB_TOKEN"]}]     ← keys only, never values

  E4 NORMALIZE
    host      → "lt-4471"
    user      → "priyas"   — NOT domain-qualified. The agent sees a
                local username. E4 cannot invent the domain.
    path      → normalized, separators canonical
    env_keys  → retained as a list of NAMES. The values were never
                collected and never enter the pipeline.

  E5 ENRICH
    host lt-4471 → AST-77c2, via the CrowdStrike-derived asset index
    ← this is the enrichment that matters. Without it, the agent's
      observation is about a hostname string rather than an asset.

    logged-on user, from the EDR host record + domain membership
      → "corp\priyas"
    ← E5 does NOT assert that this is Priya. It attaches a
      domain-qualified account string with confidence 0.9 and
      leaves the identity assertion to E6.


  WHAT E6 RECEIVES

    from AD     mail = priya.s@meridian.com          (priority 1 key)
    from AWS    tag.email = priya.s@meridian.com     (priority 1 key)
    from agent  corp\priyas                          (priority 5 key)

  Two deterministic matches on a priority-1 key. One weak key that
  will need the Resolution Directory.

  AND THE LOWERCASE RULE IN E4 IS WHY THE FIRST TWO MATCH AT ALL.
```

### 8.1 The enrichment circularity, live

```
  00:02  AD sweep begins. First record: a computer object, LT-4471.
         E5 tries to enrich → entity store is empty → no match.
         Record proceeds unenriched, confidence reduced.

  00:04  E6 creates AST-lt-4471 in the entity store.

  01:31  CrowdStrike overlay (band 5) reports LT-4471 with its
         current IP, OS and patch level.
         E5 now enriches successfully → attaches to AST-lt-4471.

  09:14  The agent on LT-4471 reports its MCP config.
         E5 enriches immediately — the asset has existed for 9 hours.

  Nothing blocked. Nothing retried. The first record was poorer than
  the last, and the graph converged without any coordination.
```

---

*Next: [Entity Resolution](05-entity-resolution.md)*

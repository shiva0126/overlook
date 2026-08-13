# Overlook — The Controller: Local Appliance UI and Connector Control Plane

**Version:** 0.1 (design draft)
**Date:** 2026-08-12
**Companion to:** `01-system-design.md`, `03-connectors.md`, `04-data-flow-to-security-facts.md`
**Status:** Brainstorming / architecture. No implementation.

---

## What this document is

The **Controller** is the appliance-local management plane: the UI and API that run on the Edge Node itself, inside the customer's environment.

`01-system-design.md` §16 gave it eight bullet points. That undersold it. The Controller is where three things happen that can happen nowhere else — anything requiring **plaintext customer data**, anything requiring **customer-held keys**, and anything the customer must be able to do **when Overlook SaaS is unreachable**.

| Part | Chapters | Covers |
|---|---|---|
| I | 1–3 | Who it is for, what it is, design principles |
| II | 4 | Information architecture |
| III | 5–14 | **The connector control plane** — the bulk of this document |
| IV | 15–17 | Ingestion, resolution and entity surfaces |
| V | 18–20 | The three proof screens |
| VI | 21 | Response policy and the local veto |
| VII | 22–25 | Content, diagnostics, agents, administration |
| VIII | 26–27 | Identity, RBAC, and the two-identity problem |
| IX | 28–29 | Non-functional reality and degraded mode |
| X | 30 | Open questions |

---

# PART I — FOUNDATIONS

---

## 1. Who uses the Controller

Getting this wrong produces the wrong product, so state it plainly.

```
   THE OVERLOOK OPERATOR
     platform engineer, security engineer, or cloud infrastructure lead
     1-2 people per customer
     logs in WEEKLY, not daily
     goals: is it collecting? what's broken? what am I not covering?
     does NOT want a dashboard. Wants a task list and a way to act.

   NOT the SOC analyst
     analysts live in the SaaS console
     they touch the appliance only through the resolve API, invisibly,
     via their browser

   NOT the CISO
     sees the SaaS dashboard
     BUT will be shown three Controller screens during procurement
     and during audit (Part V) — those must be presentable to an
     executive and an auditor, not just an engineer
```

### 1.1 The consequence

The Controller is a **low-frequency, high-consequence** interface. Every screen should answer "what do I need to do?" rather than "here is some data." Nobody watches it. When they open it, something needs attention or something needs changing.

Design implication: **an inbox, not a dashboard.** The landing surface is a prioritised list of things requiring an operator decision. Metrics are available but subordinate.

---

## 2. What the Controller is — and is not

```
   IS                                    IS NOT
   ────────────────────────────────       ────────────────────────────────
   appliance operation and config         an investigation console
   connector control plane                an attack path viewer
   plaintext-requiring workflows          a findings triage queue
   customer-held key management           a reporting tool
   the privacy proof surface              a second product
   the local veto on response
   the offline fallback
```

### 2.1 The line, drawn explicitly

> The Controller operates the appliance and does the work that requires plaintext. The SaaS console does investigation and cross-domain correlation. When SaaS is unreachable, the Controller degrades to a **read-only local view** — enough to triage, not enough to replace.

The temptation to grow the Controller into a full local console must be resisted: it doubles the UI engineering and splits the product. The one exception is degraded mode (Chapter 29), and it should feel like a lifeboat, not a second home.

---

## 3. Design principles

```
  P1  ALLOW-LIST THINKING
      Every list of what may happen is explicit. Nothing is enabled
      because it wasn't forbidden. Mirrors the Privacy Gate (04 §17.1).

  P2  SHOW THE CONSEQUENCE BEFORE THE ACTION
      Every toggle, every policy change, every disable shows what it
      will do to the graph and the findings BEFORE it is applied.
      This is the Controller's differentiating idea (Chapter 9).

  P3  NEVER CONFLATE "OFF" WITH "BROKEN"
      Chosen-off, credential-failed, rate-limited, scheduled-pause and
      crash-quarantined are five different states with five different
      actions. One "disabled" badge for all of them is a bug.

  P4  HONEST GAPS
      Coverage is always displayed as a percentage with what's missing.
      Never imply completeness that doesn't exist.

  P5  THE CUSTOMER'S POLICY WINS
      Any local policy beats any SaaS instruction, always, visibly.

  P6  WORKS OFFLINE
      Every operational function works with no internet connection.
```

---

# PART II — INFORMATION ARCHITECTURE

---

## 4. Top-level structure

```
  OVERLOOK CONTROLLER · edge-ap-south-1-a

  ┌────────────────────────────────────────────────────────────────┐
  │  ⚠ ATTENTION (4)          ← the landing surface. An inbox.      │
  ├────────────────────────────────────────────────────────────────┤
  │                                                                │
  │  COLLECTION                                                    │
  │    Catalog          browse & add — what Overlook supports       │
  │    Connections      manage what's configured — the ops list     │
  │    Coverage         what we see and what we don't               │
  │    Budget           API quota and cost governance               │
  │    Ingestion        syslog / flow / webhook / agents            │
  │                                                                │
  │  DATA                                                          │
  │    Resolution       review queue, entity browser                │
  │    Evidence         hash lookup, retention                      │
  │    Local data       retained dataset, retention, query          │
  │                                                                │
  │  PRIVACY            ← the proof screens                         │
  │    Outbound         exactly what left, in final form            │
  │    Policy           field allow-lists, tokenization, bucketing   │
  │    Resolve log      who de-tokenized what, when                 │
  │                                                                │
  │  CONTROL                                                       │
  │    Response         action classes, protected assets, kill switch│
  │    Sync             SaaS connection, queue, offline state        │
  │                                                                │
  │  PLATFORM                                                      │
  │    Content          parsers, primitives, fingerprints, versions  │
  │    Diagnostics      journal replay, support bundle, self-test    │
  │    Administration   users, RBAC, audit, backup, network, certs   │
  └────────────────────────────────────────────────────────────────┘
```

Four collection views instead of one list, because with 118 connector types and potentially hundreds of instances, a single list cannot serve browsing, operating, gap analysis and cost governance at once.

### 4.1 The Attention inbox

```
  ⚠ ATTENTION                                                    4 items

  ● CREDENTIAL EXPIRED                                        2 hours ago
    aws / account 445566778899 — collection stopped
    12 collectors affected · 1,204 entities going stale
    [ Update credential ]  [ Disable this instance ]  [ Snooze 24h ]

  ● PARSE RATE COLLAPSE                                          today
    fortigate / fw-dc1-01 — 98% → 3%
    Likely format change after a firmware update.
    Sample retained. 41,000 records quarantined.
    [ View samples ]  [ Report to Overlook ]  [ Re-identify source ]

  ● RESOLUTION REVIEW                                        12 pending
    Ambiguous identity matches need adjudication.
    Oldest: 6 days. Affects 340 potential edges.
    [ Review now ]

  ● UNIDENTIFIED SOURCE                                        ongoing
    10.4.9.22:514 — 4,200 records/hour, no fingerprint match
    [ Identify manually ]  [ View samples ]  [ Ignore this source ]
```

Every item: what happened, what it costs, what to do about it. No item without an action.

---

# PART III — THE CONNECTOR CONTROL PLANE

---

## 5. The object model — why my first sketch was wrong

The naive model is "a list of connectors with on/off switches." That collapses four distinct levels and is unusable in a real environment.

```
   CONNECTOR TYPE          aws
     what Overlook supports. One manifest. 28 collectors.
     A customer does not "have" this — they have instances of it.
        │
        ├── INSTANCE       aws / org o-a1b2c3 / account 123456789012
        │     one authenticated connection, one credential,
        │     one rate-limit domain, one health state.
        │     A customer may have 42 of these.
        │        │
        │        ├── COLLECTOR    iam.roles
        │        │     one data-gathering routine. Produces declared
        │        │     entities/relationships. Independently toggleable.
        │        │        │
        │        │        └── SCOPE   region ap-south-1
        │        │              a slice within the collector's reach:
        │        │              region, OU, namespace, subscription,
        │        │              share path, index
        │        └── COLLECTOR    ec2.instances
        └── INSTANCE       aws / account 445566778899
```

**Four levels, four independent on/off controls.** Real requests that need each level:

```
   TYPE      "we don't use GCP at all — hide it"
   INSTANCE  "disable the account we're decommissioning; keep the other 41"
   COLLECTOR "collect AWS IAM but NOT S3 object-level inventory — cost"
             "collect AD but NOT the full ACL sweep — our SOC alerts on it"
   SCOPE     "don't collect from our EU subsidiary's subscriptions"
             "don't scan the HR file share"
```

Collector-level control is the one most often missing from products like this, and the one enterprises ask for first — for cost, for source-system load, for data sensitivity, and for compliance.

### 6. State: never one "disabled" badge

```
   NOT_CONFIGURED     supported, never set up
   CONFIGURED         credentials present, not yet validated
   VALIDATING         test collection running
   ENABLED / HEALTHY  collecting, meeting success criteria
   ENABLED / DEGRADED collecting partially — some collectors failing,
                      partial enumeration, or reduced confidence
   ENABLED / STALE    last success outside the freshness threshold
   PAUSED_BY_OPERATOR deliberate. A human chose this.
   PAUSED_BY_SCHEDULE inside a configured blackout window
   PAUSED_BY_BUDGET   quota ceiling reached; will resume next period
   PAUSED_BY_GOVERNOR rate-limited by the source; backing off
   BLOCKED_CREDENTIAL auth failing; circuit open; NOT retrying
   QUARANTINED        crashed repeatedly; requires manual reset
   FAILED             persistent error, not credential-related
   ARCHIVED           removed but history retained
```

Thirteen states, five of which look like "off" to a naive UI. The operator's next action is completely different for each, so the UI must distinguish them — with distinct colour, distinct wording, and distinct available actions. Conflating "I turned this off" with "this broke" is how an operator loses trust in the whole panel.

---

## 7. Catalog — browse and add

The discovery-first surface. **Never an alphabetical list of 118 items.**

```
  CATALOG                                          118 connectors

  ┌──────────────────────────────────────────────────────────────┐
  │  ✦ DETECTED IN YOUR ENVIRONMENT                    14 found   │
  │                                                              │
  │  We found these by looking at what you've already connected: │
  │  cloud org APIs, DNS traffic, flow data, and federated apps  │
  │  in Entra.                                                   │
  │                                                              │
  │   ▸ CrowdStrike Falcon    seen in DNS + Entra app registry   │
  │       would add: host runtime context, response actions      │
  │       est. impact: +12,000 nodes, +4 finding types           │
  │       [ Connect ]  [ Not interested ]                        │
  │                                                              │
  │   ▸ Palo Alto Panorama    seen in flow data (3 devices)      │
  │       would add: network reachability, segmentation gaps      │
  │       est. impact: +14 attack paths currently invisible       │
  │       [ Connect ]  [ Not interested ]                        │
  │                                                              │
  │   ▸ Snowflake             seen in Okta app assignments        │
  │   ▸ GitLab                seen in DNS                         │
  │   ... 10 more                                                 │
  └──────────────────────────────────────────────────────────────┘

  BROWSE BY DOMAIN
    Identity & access    16    ●●●●●●●●○○○○○○○○   6 connected
    Cloud & infra        10    ●●●○○○○○○○         3 connected
    Code & build         12    ●●○○○○○○○○○○       2 connected
    Data platforms       18    ●○○○○○○○○○○○○○○○○○ 1 connected
    Security tooling     17    ○○○○○○○○○○○○○○○○○  0 connected
    Network & edge       15    ○○○○○○○○○○○○○○○    0 connected
    AI platforms         15    ●●○○○○○○○○○○○○○    2 connected
    Business context      8    ○○○○○○○○           0 connected
    Generic ingestion     7    ●●●○○○○            3 connected

  [ Search ]  [ Filter: produces ▾ ]  [ Filter: not connected ]
```

### 7.1 Filter by what it produces

An operator's real question is not "do you support Tenable?" but "what will fill my vulnerability gap?" So the catalog is filterable by **output**, driven by the manifest's `produces` block:

```
  FILTER: produces →  CAN_ASSUME edges
                      identity entities
                      network reachability
                      data classification
                      AI agent inventory
                      vulnerability properties
                      ownership data
```

### 7.2 The connector detail page, before connecting

```
  CATALOG › Identity & access › Okta

  WHAT IT ADDS TO YOUR GRAPH
    entities        IDENTITY (user, service account), GROUP, APPLICATION
    relationships   MEMBER_OF, CAN_ASSUME, AUTHENTICATES_TO
    properties      MFA state, last activity, lifecycle status
    findings        3 types — see list

  WHAT IT UNLOCKS THAT YOU DON'T HAVE TODAY
    • Federated CAN_ASSUME edges into your 3 AWS orgs
      → est. 41 attack paths currently invisible
    • MFA state on 4,200 identities
      → improves confidence on 1,100 existing edges
    • Authoritative canonical keys for identity resolution
      → would resolve 340 of your 12 pending review-queue items

  WHAT IT READS   (plain language, per collector)
    users            profile, status, MFA factors, last login
    groups           membership
    applications     assignments, sign-on modes
    system log       authentication events (metadata only)
    NOT read:        passwords, session tokens, MFA secrets

  WHAT IT NEEDS
    Okta API token with read-only admin, or an OAuth service app
    [ Copy least-privilege setup ]  ·  Terraform / manual / CLI

  COST ESTIMATE
    ~2,400 API calls/day · ~6% of your Okta rate limit
    ~340 MB/month journaled · ~0.4 MB/day of facts

  [ Connect Okta ]        [ Add to plan ]        [ Back ]
```

The "what it unlocks that you don't have today" block is computable — the manifest declares outputs, and the graph knows what's missing. That turns a catalog entry into a business case.

---

## 8. Connections — the operational list

```
  CONNECTIONS                                    18 types · 71 instances

  [ All ] [ Needs attention 4 ] [ Paused 3 ] [ Healthy 64 ]     [ ⏻ Stop all ]

  ▼ aws                                    42 instances    ●39 ◐2 ✕1
    ┌──────────────────────────────────────────────────────────────┐
    │ TYPE-LEVEL                                        [ ⏻ ON  ]  │
    │ 28 collectors · 24 enabled · 4 disabled by operator          │
    │ org auto-discovery: ON — new accounts adopt "prod" profile   │
    │ [ Manage collectors ]  [ Manage profiles ]  [ Bulk edit ]    │
    ├──────────────────────────────────────────────────────────────┤
    │ ● 123456789012  Production        healthy   4m ago  [ ⏻ ON ] │
    │   profile: prod-full · 28/28 collectors · quota 22%          │
    │ ● 234567890123  Staging           healthy   6m ago  [ ⏻ ON ] │
    │   profile: nonprod-lite · 12/28 collectors · quota 4%        │
    │ ✕ 445566778899  Sandbox      BLOCKED_CREDENTIAL     [ ⏻ ON ] │
    │   auth failing 2h · circuit open · NOT retrying              │
    │   → 1,204 entities going stale · [ Fix credential ]          │
    │ ◐ 556677889900  EU-Prod          degraded  11m ago  [ ⏻ ON ] │
    │   iam.access_analyzer failing · confidence reduced on        │
    │   3,400 edges · [ Details ]                                  │
    │ ⏸ 667788990011  Decommissioning  PAUSED_BY_OPERATOR [ ⏻ OFF]│
    │   paused 14 Jul by shiva · "account being closed"            │
    │   → graph retained, marked stale · [ Resume ] [ Archive ]    │
    │   ... 37 more                              [ Show all 42 ]   │
    └──────────────────────────────────────────────────────────────┘

  ▶ entra                                   1 instance     ●1
  ▶ active-directory                        2 instances    ●2
  ▶ okta                                    1 instance     ●1
  ▶ github                                  3 instances    ●2 ⏸1
  ▶ kubernetes                              5 instances    ●4 ◐1
  ▶ fortigate                               6 instances    ●5 ✕1
```

Notes on this design:

- **Grouped by type, expandable to instances.** 71 flat rows is unreadable; 18 collapsed groups is scannable.
- **The paused instance shows who paused it, when, and why.** A free-text reason is mandatory on pause. Six months later somebody will ask.
- **Every failure row states the graph consequence** — "1,204 entities going stale" — not just "error."
- **`⏻ Stop all`** is deliberately top-right and always present. During an incident or a change freeze, an operator needs one button that halts every outbound API call. It should require a typed confirmation and log loudly.

---

## 9. The impact preview — the Controller's differentiating feature

This is the idea that makes the connector plane more than a config screen.

Because every collector's manifest **declares what it produces**, and the graph knows what depends on those outputs, the Controller can compute the consequence of any change *before it is applied*. No competing product can do this, because none has a manifest-declared dependency chain from collector → entity/edge type → finding.

### 9.1 Disabling a collector

```
  DISABLE COLLECTOR:  aws / iam.access_analyzer
  ────────────────────────────────────────────────────────────────

  WHAT YOU LOSE

    ✕ FINDING TYPES DISABLED                                     2
        IAM-002  Cross-account trust without ExternalId
        IAM-041  Unused escalation primitive held
        → 14 currently-open findings will be withdrawn

    ◐ CONFIDENCE REDUCED                              12,400 edges
        CAN_ASSUME edges lose provider verification
        0.99 → 0.93 (static analysis only)
        → 3 attack paths drop below the display threshold
        → 1 CRITICAL path becomes MEDIUM

    ○ CAPABILITY LOST
        External-access ground truth for 42 accounts
        Unused-permission data feeding CIEM rightsizing
        → "Rightsize" recommendations unavailable for 340 roles

  WHAT YOU GAIN
        ~1,100 fewer API calls/day  (−3% of IAM quota)
        ~12 MB/day less journaled

  ALTERNATIVES
        ▸ Reduce frequency 4h → 24h — keeps all findings,
          saves 83% of the calls        [ Do this instead ]
        ▸ Scope to production accounts only — keeps critical
          coverage, saves 61%           [ Do this instead ]

  ────────────────────────────────────────────────────────────────
  Reason for disabling (required):
  [                                                            ]
  [ Cancel ]                              [ Disable collector ]
```

The **Alternatives** block is what turns this from a warning into help. Most "turn it off" requests are really cost or noise complaints, and a frequency or scope change usually solves them without losing coverage.

### 9.2 The same machinery, inverted, for enabling

```
  ENABLE COLLECTOR:  github / oidc_trusts
  ────────────────────────────────────────────────────────────────
  WHAT YOU GAIN
    + 340 attack paths currently INVISIBLE become visible
        of which 4 are estimated CRITICAL
    + 2 finding types enabled
        IAM-004  OIDC trust with insufficient subject condition
        ASPM-011 Pipeline with unscoped cloud role assumption
    + Supply-chain path completion: your GitHub repos connect to
      your AWS roles. Today that hop is missing, so every
      repo → cloud path stops at the boundary.

  COST
    ~180 API calls/day · <1% of quota · negligible storage

  [ Cancel ]                                    [ Enable ]
```

"340 paths currently invisible" is honest, specific, and unarguable. It is the growth loop from `03-connectors.md` §11 rendered as a UI.

### 9.3 The multiplicative property, made visible

`03-connectors.md` §23.2 argued that connector value is multiplicative — a path needs *every* hop, so one missing connector makes paths invisible rather than shorter. The Controller should show that directly:

```
  PATH COMPLETION ANALYSIS

  Your graph has 41 path fragments that DEAD-END at a missing hop.

    ▸ 340 fragments stop at:  repo → ??? → cloud role
        missing hop: GitHub OIDC trust collection
        [ Enable github/oidc_trusts ]

    ▸ 128 fragments stop at:  on-prem identity → ??? → cloud
        missing hop: AD FS / federation trust collection
        [ Configure federation collection ]

    ▸  74 fragments stop at:  cloud workload → ??? → on-prem DB
        missing connector: FortiGate policy collection on fw-dc2
        [ Connect fw-dc2 ]
```

This is the single most persuasive screen in the product for driving connector adoption, and it costs nothing extra — the fragments are a byproduct of path computation.

---

## 10. Collector management

```
  aws  ›  COLLECTORS                              28 total · 24 enabled

  [ All ] [ Enabled ] [ Disabled ] [ By cost ] [ By graph value ]

  ┌──────────────────────────────────────────────────────────────────┐
  │ COLLECTOR              PRODUCES          FREQ    COST   STATE    │
  ├──────────────────────────────────────────────────────────────────┤
  │ organizations          accounts, OUs      24h    ▁      ⏻ ON     │
  │ iam.users              IDENTITY           4h     ▂      ⏻ ON     │
  │ iam.roles              ROLE, CAN_ASSUME   4h     ▄      ⏻ ON     │
  │ iam.policies           grants             4h     ▅      ⏻ ON     │
  │ iam.access_analyzer    verification       4h     ▂      ⏻ ON     │
  │ sts.assume_chains      CAN_ASSUME         12h    ▃      ⏻ ON     │
  │ cloudtrail             usage, EVENT_SUM   15m    ▇      ⏻ ON     │
  │ ec2.instances          ASSET              4h     ▃      ⏻ ON     │
  │ s3.buckets             DATASTORE          4h     ▂      ⏻ ON     │
  │ s3.object_inventory    classification     7d     █      ⏻ OFF    │
  │   └ disabled 2 Aug by shiva — "cost: $4k/mo in LIST calls"       │
  │ rds.instances          DATASTORE          4h     ▂      ⏻ ON     │
  │ kms.keys               CAN_READ (crypto)  12h    ▁      ⏻ ON     │
  │ secretsmanager         SECRET             12h    ▂      ⏻ ON     │
  │ ...                                                              │
  └──────────────────────────────────────────────────────────────────┘

  ⚠ 4 disabled collectors are reducing coverage:
      s3.object_inventory   → data classification 34% (of S3)
      inspector             → vulnerability properties absent
      macie                 → PII detection in S3 absent
      config                → configuration history absent
    [ Review impact of all 4 ]
```

**Sort by graph value** is the important affordance. An operator wants to know which of the 28 collectors actually matter. Graph value is computable: how many edges, how many findings, how many crown-jewel paths depend on this collector's output.

### 10.1 Cost sparkline, not a number

The `COST` column uses a relative bar rather than a currency figure, because we cannot know the customer's contract pricing. It ranks collectors against each other within the connector, which is what the decision actually needs. Absolute API-call counts are available on the detail view.

---

## 11. Profiles

Nobody hand-tunes 695 collectors across 71 instances. Profiles make bulk configuration tractable.

```
  PROFILES for aws                                        4 defined

  ● prod-full          28/28 collectors · 15m-24h cadence
                       applied to 12 instances
                       "Everything, highest frequency"

  ● nonprod-lite       12/28 collectors · 24h cadence
                       applied to 26 instances
                       "IAM + inventory only. No audit trail,
                        no classification, no flow."

  ● sandbox-minimal     6/28 collectors · 7d cadence
                       applied to 3 instances
                       "Existence and IAM only."

  ● eu-restricted      24/28 collectors · 24h cadence
                       applied to 1 instance
                       "Excludes classification collectors —
                        GDPR review pending."

  AUTO-ADOPTION
    New accounts discovered under org o-a1b2c3 adopt:
      account name matches  *prod*      → prod-full
      account name matches  *stag*|*dev*→ nonprod-lite
      OU = Sandbox                      → sandbox-minimal
      otherwise                         → nonprod-lite + notify

  [ New profile ]  [ Edit ]  [ Where is this applied? ]
```

Auto-adoption matters: cloud estates grow continuously, and an operator who must manually configure every new account will simply stop, leaving silent gaps.

### 11.1 Overrides must be visible

```
  Instance 556677889900 (EU-Prod)
    profile: eu-restricted
    ⚠ 2 local overrides deviate from the profile:
        cloudtrail  frequency 15m → 1h    (set 4 Aug, reason: cost)
        kms.keys    disabled              (set 2 Aug, reason: perms)
    [ Revert to profile ]  [ Promote overrides into profile ]
```

Silent drift between profile and instance is how a config plane becomes untrustworthy. Every deviation is shown, attributed, and reversible.

---

## 12. Budget and quota governance

```
  BUDGET

  ┌────────────────────────────────────────────────────────────────┐
  │ AWS       ████████░░░░░░░░░░░░  22% of published quota         │
  │           ceiling 30% · 2.4M calls today · 11M/day allowed     │
  │           top consumers: cloudtrail 41% · iam.policies 22%     │
  ├────────────────────────────────────────────────────────────────┤
  │ Entra     ███░░░░░░░░░░░░░░░░░   9%   ceiling 30%              │
  │ Okta      ██░░░░░░░░░░░░░░░░░░   6%   ceiling 25%              │
  │ GitHub    ████████████░░░░░░░░  38%   ceiling 30%  ⚠ OVER      │
  │           → github/secret_scanning throttled by governor       │
  │           [ Raise ceiling ]  [ Reduce frequency ]              │
  └────────────────────────────────────────────────────────────────┘

  POLICY
    Default ceiling: 30% of published quota per provider
    ↳ 70% is reserved for the customer's own workloads. Always.

    Blackout windows (no collection):
      Active Directory   02:00-04:00 daily    "backup window"
      Oracle prod        Sun 01:00-06:00      "maintenance"
      [ Add window ]

    On ceiling breach:  throttle  ▾   (throttle | pause | alert only)
```

Two design commitments worth stating in the UI itself: the default is 30% of the customer's quota, and the other 70% is theirs. An operator who reads that sentence stops worrying about the appliance breaking their automation.

---

## 13. Adding an instance

```
  CONNECT: aws                                      step 2 of 4

  ● SCOPE                                              ✓ complete
      Organization o-a1b2c3 detected · 42 accounts found
      ☑ include all current accounts
      ☑ auto-adopt new accounts (rules in Profiles)
      ☐ exclude: [ 667788990011 (Decommissioning) ]

  ● CREDENTIALS                                        ← you are here
      method:  ● IRSA (recommended — no stored secret)
               ○ AssumeRole from a hub account
               ○ Access key (least preferred)

      Apply this in each account:
      ┌──────────────────────────────────────────────────┐
      │ # Terraform · least privilege · read-only        │
      │ module "overlook_reader" { ... }                  │
      └──────────────────────────────────────────────────┘
      [ Copy Terraform ]  [ Copy CloudFormation ]  [ Copy CLI ]

      ⓘ This role grants READ ONLY. Response actions require a
        separate role you can choose never to create.

  ○ PROFILE            choose collectors and cadence
  ○ VALIDATE           scoped test run against 1 account

  [ Back ]                                         [ Continue ]
```

### 13.1 Validation must be a real, scoped run

```
  VALIDATE                                          step 4 of 4

  Running a scoped test against account 123456789012 (ap-south-1)...

    ✓ credentials valid, 12 collectors authorised
    ✓ organizations       42 accounts enumerated
    ✓ iam.roles           412 roles, 3 pages, 41s
    ✓ iam.policies        1,204 policies parsed, 0 failures
    ⚠ iam.access_analyzer not enabled in this account
        → 2 finding types unavailable · [ how to enable ]
    ✓ ec2.instances       1,847 instances

  RESULT
    1,204 identities · 412 roles · 38 with admin privilege
    First facts in 4 minutes. First finding in 6 minutes.

    ⚡ PREVIEW: 3 findings already visible in this one account
       IAM-001  escalation to admin via PassRole + Lambda   CRITICAL
       IAM-026  admin identity without MFA                  CRITICAL
       IAM-022  access key 641 days old                      MEDIUM

  [ Adjust configuration ]              [ Enable all 42 accounts ]
```

Showing three real findings from one account before the operator commits is the strongest possible validation, and it costs one scoped run.

---

## 14. Coverage

```
  COVERAGE

  ┌───────────────────────────────────────────────────────────────┐
  │ Identity     ████████████████████░  97%                       │
  │   ✓ Entra ✓ AD ✓ Okta            ○ AD CS not connected        │
  │ Cloud        ███████████████████░░  94%                       │
  │   ✓ AWS (39/42 accounts) ✓ Azure  ○ GCP not connected         │
  │ Code         ██████████████░░░░░░░  71%                       │
  │   ✓ GitHub (2/3 orgs) ✓ Jenkins   ○ Terraform Cloud           │
  │ Data         ██████████████████░░░  88%                       │
  │   ⚠ classification 34% — s3.object_inventory disabled         │
  │ AI           ███████████████░░░░░░  78%                       │
  │   ✓ Bedrock ✓ OpenAI              ○ vector DBs, ○ MCP registry│
  │ Network      ████████████░░░░░░░░░  62%   ← weakest           │
  │   ✓ FortiGate (5/6)               ○ Panorama ○ Zscaler        │
  │ Runtime      █████████░░░░░░░░░░░░  45%   ← weakest           │
  │   ○ no EDR connected  ○ no agents deployed                    │
  └───────────────────────────────────────────────────────────────┘

  WHAT THE GAPS COST YOU
    Network 62%  → 74 path fragments dead-end at a missing
                   firewall policy hop
    Runtime 45%  → no process context; Shadow AI detection limited
                   to DNS-level inference
    Data class.  → 4,100 datastores unclassified; crown-jewel
                   designation incomplete

  [ Close the biggest gap ]
```

Coverage is deliberately honest, and honesty is the growth mechanism. "Network 62%" is both an admission and a next step.

---

# PART IV — INGESTION, RESOLUTION, ENTITIES

---

## 15. Ingestion

Push and stream sources are not connectors and need their own surface.

```
  INGESTION

  LISTENERS
    syslog/udp:514    ● 41,200 rec/s   ⚠ est. 0.4% loss (UDP)
                      [ recommend TCP/TLS — how to migrate ]
    syslog/tcp:6514   ● 8,100 rec/s    0% loss
    netflow/udp:2055  ● 47,000 rec/s   → 180k aggregates/day
    webhooks/443      ● 1,204 today    3 sources
    agents/mtls:8443  ● 1,847 agents   1,844 reporting

  SOURCE REGISTRY                                      41 sources
    10.4.0.9:514    fortigate/FortiOS 7.4    ✓ parse 99.1%
    10.4.0.11:514   paloalto/PAN-OS 11.1     ✓ parse 98.7%
    10.4.2.40:514   linux/rsyslog            ✓ parse 97.2%
    10.4.9.22:514   ⚠ UNIDENTIFIED           4,200 rec/h
                    [ view samples ] [ identify ] [ ignore ]
    fw-dc1-01       ✕ fortigate PARSE 3%     ← was 98% yesterday
                    likely firmware format change
                    [ samples ] [ report ] [ re-identify ]

  QUARANTINE                                    41,204 records
    grouped by failure signature, samples retained
    [ Browse ]  [ Replay after content update ]
```

Parse rate per source, displayed permanently, is the defence against the silent-degradation failure mode in `04 §11.3`.

---

## 16. Resolution review

The ongoing human task the architecture creates. Not a debug page — a queue somebody works.

```
  RESOLUTION REVIEW                                    12 pending

  ┌──────────────────────────────────────────────────────────────┐
  │ ARE THESE THE SAME IDENTITY?                    score 0.78   │
  │                                                              │
  │   A  priya.s@corp.com          B  p.sharma@corp.com          │
  │      Okta · Engineering           AD · Engineering            │
  │      created 2024-03-14           created 2024-03-14          │
  │      manager: r.kumar             manager: r.kumar            │
  │      12 group memberships         11 group memberships        │
  │                                   (10 shared)                 │
  │                                                              │
  │   EVIDENCE FOR      same manager, same dept, same creation    │
  │                     date, 10 shared groups                    │
  │   EVIDENCE AGAINST  different email local-part                 │
  │                                                              │
  │   IF MERGED         +14 edges · 1 new path to a crown jewel   │
  │   IF SEPARATE       both remain with reduced confidence        │
  │                                                              │
  │   [ Same identity ]  [ Different ]  [ Skip ]                 │
  │   ☑ remember this rule for future matches                    │
  └──────────────────────────────────────────────────────────────┘

  QUALITY
    deterministic 71% · probabilistic 24% · queued 5%
    over-merges reported: 0   under-merges suspected: 3
```

`IF MERGED` showing the graph consequence is the same impact-preview principle applied here — it tells the operator whether this decision matters.

## 17. Entity browser and evidence

```
  ENTITY  priya.s@corp.com                    IDN-9f3a7c21e845b0d6

  ALIASES         okta:00u1a2b3 · adguid:8f14e45f · sam:corp\priyas
                  upn:priya.s@corp.local · arn:...:user/priya.s
  SOURCES         okta, active-directory, aws, crowdstrike, agent
  RESOLUTION      deterministic · confidence 1.0
  [ Force split ]  [ View all 5 source records ]

  EVIDENCE LOOKUP
    hash: [ sha256:8a1f...c4d2                    ]  [ Retrieve ]
    → iam_policy_document · 4.2 KB · captured 04:00:11
      retained until 2026-11-10
    ⓘ every retrieval is logged. 14 retrievals in 30 days.
```

`Force split` matters: over-merges are the unforgivable error, so unmerging must be one click and must be reversible.

---

# PART V — THE THREE PROOF SCREENS

These are shown to CISOs and auditors. They are the privacy architecture made visible, and they should be built early — they are the reason a customer chooses this product.

## 18. Outbound inspector

```
  OUTBOUND                                    last 24h · 11.8 MB

  1,204 facts sent · 1,891 heartbeat refreshes · 3 quarantined

  ┌──────────────────────────────────────────────────────────────┐
  │ 04:00:11  RELATIONSHIP  CAN_ASSUME                            │
  │   IDN-9f3a7c21e845b0d6 → ROL-2b8e4f19a70c5d33                 │
  │   [ show what this was before the Privacy Gate ▾ ]            │
  │   ┌────────────────────────────────────────────────────────┐  │
  │   │ BEFORE (never left this appliance)                     │  │
  │   │   subject  email:priya.s@corp.com                      │  │
  │   │   object   arn:aws:iam::123456789012:role/DevOpsAdmin   │  │
  │   │   evidence <4.2 KB IAM policy document>                 │  │
  │   │ AFTER (what actually crossed the boundary)             │  │
  │   │   subject  IDN-9f3a7c21e845b0d6                        │  │
  │   │   object   ROL-2b8e4f19a70c5d33                        │  │
  │   │   evidence sha256:8a1f...c4d2  (hash only)             │  │
  │   │ STRIPPED   email, ARN, account id, role name,           │  │
  │   │            policy document                              │  │
  │   └────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘

  QUARANTINED — never transmitted                               3
    ✕ PROPERTY on DST-1c4b — field `db_hostname` not in allow-list
      [ view ] [ add field to policy ] [ discard ]

  [ Export for audit ]  [ Filter ]  [ Verify a batch signature ]
```

The before/after diff is the whole point. A customer can see, per fact, exactly what was withheld.

## 19. Privacy policy

```
  PRIVACY POLICY                          v7 · edited 2 Aug by shiva

  TOKENIZATION
    ☑ usernames, emails    ☑ hostnames      ☑ internal IPs
    ☑ ARNs, resource ids   ☑ repo names     ☑ file paths
    ☑ group names          ☐ department      ← customer choice
    ☐ region names         ☐ CVE ids        ☐ model names
                                            (these carry no PII)

  BUCKETING
    record counts   → 1M-10M     ▾
    data volumes    → order of magnitude ▾
    durations       → 15-min buckets ▾

  FIELD ALLOW-LIST                              per fact type ▾
    RELATIONSHIP: subject, predicate, object, mechanism,
      conditions, privilege_level, confidence, first_seen,
      last_seen, observation_count, evidence_ref, sources
    ⓘ Anything not listed is dropped. Additions require an edit here.

  PROMPT HANDLING
    ● metadata only (default)   ○ local inspection   ○ full capture

  ┌──────────────────────────────────────────────────────────────┐
  │ PREVIEW THIS POLICY                                          │
  │ Applying the proposed change to the last 24h would:          │
  │   • additionally tokenize `department` on 4,201 facts         │
  │   • drop `region` from 1,100 facts                            │
  │   • reduce outbound volume 11.8 MB → 11.2 MB                  │
  │   • SaaS effect: department-based grouping unavailable        │
  │ [ Apply ]  [ Discard ]                                        │
  └──────────────────────────────────────────────────────────────┘

  [ Version history ]  [ Export as document ]
```

`Export as document` produces the human-readable statement of exactly what leaves — a procurement and audit artifact generated from live configuration rather than written by marketing.

## 20. Resolve log

```
  RESOLVE LOG                             de-tokenization requests

  09:14:02  analyst@corp.com    47 tokens   IDENTITY, ASSET, DATASTORE
            path view · SaaS session s_8f14 · granted
  09:16:41  analyst@corp.com     1 token    evidence retrieval
            sha256:8a1f...c4d2 · granted
  09:22:10  contractor@ext.com   4 tokens   IDENTITY
            ✕ DENIED — no resolve:identity scope

  30-day summary: 1,204 requests · 6 users · 3 denied
  [ Export for audit ]  [ Manage resolve RBAC ]
```

An unexpected gift of the architecture: the customer gets a complete record of who looked at whose data, which they would otherwise have to build themselves.

---

# PART VI — RESPONSE

## 21. Local policy and the veto

```
  RESPONSE POLICY

  MASTER              ○ enabled   ● DISABLED (default)
    ⓘ While disabled, this appliance refuses every response
      command from Overlook SaaS. SaaS cannot override this.

  ACTION CLASSES               (each independently controlled)
    Block connection      ☐ enabled   ☑ requires local approval
    Terminate process     ☐ enabled   ☑ requires local approval
    Quarantine host       ☐ enabled   ☑ requires local approval
    Lock account/session  ☐ enabled   ☑ requires local approval

  PROTECTED ASSETS — never actioned, no exceptions
    ☑ domain controllers (auto-detected: 4)
    ☑ OT/ICS network segments (10.90.0.0/16)
    ☑ certificate authorities (2)
    + [ add asset or tag ]

  MAINTENANCE WINDOWS — no automated action
    Sat 22:00 - Sun 06:00 weekly · [ add ]

  TTL CEILING           4h  ▾   (max any quarantine may last)
  AUTO-RELEASE          ☑ agent releases on TTL even if this
                          appliance is unreachable

  [ Command audit log ]                        [ ⏻ KILL SWITCH ]
```

Master-disabled by default is the right posture. A customer can deploy the entire platform with provably zero write capability, which removes the largest procurement objection, and enable response later as a deliberate act.

---

# PART VII — PLATFORM

## 22. Content

```
  CONTENT

  parsers                v2026.08.09   ● current    412 grammars
  escalation-primitives  v2026.08.02   ◐ update available
                                         +6 primitives (GCP)
  ai-fingerprints        v2026.08.11   ● current    1,847 signatures
  iam-semantics          v2026.07.28   ◐ update available
                                         AWS added 3 services
  mcp-reputation         v2026.08.10   ● current
  classification         v2026.06.14   ● current

  UPDATE MODE   ○ automatic  ● staged (canary → 10% → 100%)
                ○ manual approval   ○ pinned
  ☑ auto-rollback if parse rate drops >5% after update

  AIR-GAPPED    [ Upload signed bundle ]
  [ Version history ]  [ Rollback ]
```

## 23. Diagnostics

```
  DIAGNOSTICS

  JOURNAL REPLAY                          ← primary debugging tool
    source [ fortigate/fw-dc1-01 ▾ ]
    window [ 2026-08-11 00:00 → 2026-08-12 00:00 ]
    with content version [ v2026.08.09 ▾ ]
    [ Dry run — diff against original output ]

    ⓘ Because raw data never leaves your environment, Overlook
      support cannot reproduce parsing issues. Replay lets you
      verify a fix locally before applying it.

  PIPELINE                             throughput per stage
    receive ▇▇▇▇▇▇▇  parse ▇▇▇▇▇▇  normalize ▇▇▇▇▇
    resolve ▇▇▇▇     derive ▇▇▇    fact build ▇▇   gate ▇▇
    ⚠ resolve is the bottleneck — 41% of pipeline latency

  SUPPORT BUNDLE
    ☑ config  ☑ health  ☑ metrics  ☑ error samples (redacted)
    ☐ raw samples  ← off by default; contains customer data
    [ Generate ]  [ Preview exactly what's included ]
```

## 24. Agents

```
  AGENTS                                1,847 enrolled · 1,844 reporting

  fleet     Windows 1,204 · macOS 512 · Linux 131
  versions  2.1.0 (1,801) · 2.0.4 (46)   [ stage upgrade ]
  health    ● 1,844 reporting   ⚠ 3 silent >48h

  COLLECTION POLICY
    AI config scan     4h ▾      process snapshot   60s ▾
    ☑ MCP configs  ☑ local models  ☑ IDE extensions
    ☐ full process tree  ← use your EDR connector instead

  RESOURCE LIMITS       CPU <1% avg · RAM <150 MB · net <50 MB/day
    [ view compliance report ]

  RESPONSE EXECUTOR     ● NOT INSTALLED (separate package)
```

## 25. Administration

```
  ADMINISTRATION

  LOCAL USERS                     ← independent of Overlook SaaS
    ● shiva@corp.com          Operator      2FA ✓   2 min ago
    ● r.kumar@corp.com        Auditor       2FA ✓   3 days ago
    AUTH  ○ local accounts  ● corporate IdP (OIDC)
          ⓘ Local accounts remain as break-glass if the IdP
            is unreachable.

  ROLES
    Operator   configure connectors, manage credentials, resolve
    Auditor    read-only + outbound inspector + all audit logs
    Responder  approve response actions
    Custodian  privacy policy, retention, tokenization key

  AUDIT LOG              hash-chained, tamper-evident
    [ Export ]  [ Stream to SIEM ]  [ Verify chain ]

  APPLIANCE
    version 2.1.0 · uptime 41d · [ upgrade ]
    enrollment: deployment DPL-acme-bank · cert expires 62d (auto-renew)
    tokenization key: customer KMS ● reachable · [ rotate ]
    backup: nightly to [ configured target ] · [ restore ]
```

---

# PART VIII — IDENTITY AND ACCESS

## 26. Two identity systems, unavoidably

```
   SaaS IDENTITY                    CONTROLLER IDENTITY
   analysts, CISO                   the Overlook operator, auditors
   Overlook-managed SSO             the CUSTOMER's IdP, or local accounts
   works only when online           MUST work offline
   grants resolve scopes            grants appliance control
```

They cannot be the same system, because the Controller must function when SaaS is unreachable — that is a core requirement, not an edge case. Consequences:

- The Controller needs its own authentication (OIDC/SAML to the customer's IdP, with local break-glass accounts).
- The **resolve endpoint** is the bridge: it accepts a SaaS-issued, short-lived, user-scoped JWT and validates it against a key pinned at enrollment, then applies **local** RBAC on top. A SaaS admin cannot grant themselves resolve rights the Controller has not authorised.
- Two audit logs, both exportable, correlatable by session id.

## 27. Separation of duties

```
  Operator   configures collection, manages credentials
             CANNOT change the privacy policy or the tokenization key
  Custodian  privacy policy, retention, tokenization key, resolve RBAC
             CANNOT configure connectors or run response
  Responder  approves response actions
             CANNOT change protected-asset lists
  Auditor    reads everything, changes nothing
```

Splitting Operator from Custodian is what lets a customer honestly claim that the person running the appliance cannot widen what leaves it.

---

# PART IX — NON-FUNCTIONAL REALITY

## 28. The dual-SLA problem

The Controller process hosts two things with completely different requirements:

```
   ADMIN UI                    RESOLVE / EVIDENCE API
   a few sessions/week         every analyst's browser, constantly
   seconds are fine            p95 < 150 ms or the SaaS UI feels broken
   downtime tolerable          downtime blocks all investigation
```

Design consequences:

- The resolve/evidence API gets **reserved capacity** (pool D in `04 §26`) that the admin UI can never consume.
- Admin operations that could be expensive — impact previews, coverage computation, replay dry-runs — run as background jobs with progress, never synchronously in a request.
- The admin UI is server-rendered and boring. The resolve API is a tight, cached, indexed lookup path. They share a process but not a code path.

## 29. Degraded mode

```
  ⚠ NOT CONNECTED TO OVERLOOK SaaS — 4h 12m

  Still working:
    ✓ collection, parsing, resolution, fact building
    ✓ facts buffering locally (queue 18% of 7-day capacity)
    ✓ evidence and token resolution for local users
    ✓ all appliance configuration
    ✓ local entity and finding view (read-only)

  Unavailable:
    ✕ cross-appliance correlation and global attack paths
    ✕ response actions
    ✕ content updates

  [ Test connection ]  [ View queue ]  [ Local findings ]
```

The banner states specifically what does and does not work. A generic "connection error" would leave an operator unsure whether data is being lost — and the answer, which they need to know, is that it is not.

---

# PART X — OPEN QUESTIONS

```
  Q1  Does the Controller ship a local read-only findings/graph view at
      all, or is degraded mode limited to inventory and health? (04 Q5)

  Q2  Impact preview requires a collector → edge-type → finding dependency
      graph. Is that derived automatically from manifests plus finding
      definitions, or maintained by hand? Automatic is better and harder.

  Q3  "Graph value" ranking for collectors — computed from live graph
      contribution, or declared statically in the manifest?

  Q4  Profiles: appliance-local, or defined centrally in SaaS and
      distributed? Central is nicer to operate, but it means SaaS can
      change what gets collected — which cuts against P5.

  Q5  Path-fragment analysis (§9.3) needs the path engine, which lives in
      SaaS. Does the Controller show it by fetching from SaaS (breaking
      offline capability for that screen), or does the appliance compute
      local fragments itself?

  Q6  Is `⏻ Stop all` a soft pause (finish in-flight, stop scheduling) or
      a hard abort? Recommend soft, with a separate hard abort behind
      a confirmation.

  Q7  Server-rendered or SPA? Server-rendered is far less work solo and
      suits a low-frequency admin tool; the resolve API is separate
      regardless.

  Q8  Does the Controller expose its own API for customers to automate
      configuration (infrastructure-as-code for connector setup)?
      Enterprises will ask. It also risks becoming a second surface
      to maintain.
```

---

*End of document.*

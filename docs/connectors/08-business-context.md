# Domain 08 — Business Context

**8 connectors · 38 collectors** · [Index](00-index.md)

Band 5, alongside the security overlays. This is the most undervalued domain in the catalog: it produces almost no edges and yet determines whether findings ever get fixed.

⚠ **The reason it matters.** A finding without an owner is work for the security team. A finding with an owner is work for the team that can actually fix it. Everything else in this catalog answers *what is wrong*; this domain answers *whose problem is it* and *how much does it matter*.

---

## 1 · ServiceNow

```
  api_surface   configuration
  functions     collect · respond (separate — ticket creation)
  auth          OAuth or a dedicated integration user
  least priv    read on CMDB tables, change and incident tables;
                write ONLY in the response manifest
  ⚠             CMDB quality varies enormously. Ingest it as a
                CORROBORATING source with declared confidence, never
                as authoritative over what we directly observed.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `cmdb_ci` | Configuration items across classes | `ASSET` corroboration, business names | 12h | ★ |
| `cmdb_relationships` | CI dependency relationships | application↔infrastructure mapping | 12h | ★ |
| `ownership` | Assigned-to, owned-by, support group per CI | **ownership on every entity** | 12h | ★ |
| `business_services` | Business services and their criticality tiers | **crown-jewel designation input** | 12h | ★ |
| `change_requests` | Changes, windows, approvals | maintenance-window awareness, change attribution | 4h | ★ |
| `incidents` | Incidents linked to CIs | operational context | 4h | |
| `cmdb_health` | CMDB completeness and staleness metrics | how much to trust the above | 24h | |
| — `create_ticket` | Ticket creation for remediation | **RESPOND** | — | |

**7 collect + 1 respond.**

`business_services` deserves attention: it is often the only place a customer has already written down which systems are business-critical. Importing it means crown jewels are designated from the customer's own definitions rather than from our guesses.

---

## 2 · Jira

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Projects, leads, components | ownership mapping | 12h | ★ |
| `users_groups` | Users, groups, permission schemes | `IDENTITY` corroboration | 12h | |
| `issues` | Issues linked to assets or findings | remediation tracking, SLA state | 4h | ★ |
| `components` | Components and their owners | fine-grained ownership | 12h | |
| — `create_issue` | Issue creation for remediation | **RESPOND** | — | |

**4 collect + 1 respond.**

The `issues` collector closes the loop: a finding exported to Jira, then tracked back, gives **time-to-remediate** — which is the metric that turns a posture product into a managed service deliverable.

---

## 3 · Workday

```
  ⚠ HR data is the most sensitive non-security data in the catalog.
    Collect the minimum: employment status, department, manager,
    location, start/end dates. Never compensation, never performance,
    never personal contact details.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `workers` | Workers, employment status, start/end dates | `IDENTITY` enrichment, **leaver detection** | 4h | ★ |
| `org_structure` | Supervisory organisations, cost centres | department, org hierarchy | 12h | ★ |
| `managers` | Manager relationships | ownership escalation path, resolution input | 12h | ★ |
| `locations` | Work locations and regions | jurisdiction and residency context | 24h | |
| `contingent_workers` | Contractors and their end dates | **contractor access findings** | 4h | ★ |

**5 collectors.**

`workers` and `contingent_workers` produce one of the highest-signal findings in the whole product: **an account that is still active for a person whose employment ended.** No identity system reliably knows this; HR does.

---

## 4 · BambooHR / HiBob and similar

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `employees` | Employees, status, dates | `IDENTITY` enrichment, leaver detection | 4h | ★ |
| `departments` | Department structure | org context | 12h | ★ |
| `reporting` | Reporting lines | ownership escalation | 12h | |

**3 collectors.**

---

## 5 · Slack

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Workspace members, guests, bots | `IDENTITY`, external user findings | 4h | ★ |
| `channels` | Channels, visibility, shared/connect channels | **external sharing exposure** | 4h | ★ |
| `apps` | Installed apps and their OAuth scopes | `CAN_ACCESS`, third-party privilege | 4h | ★ |
| `bots` | Bot users and their tokens | `IDENTITY` (non-human), `SECRET` | 4h | ★ |
| `ai_usage` | AI assistant and agent app activity | `AI_APPLICATION`, `USES` | 4h | ★ |
| `admin_audit` | Admin and access events | `EVENT_SUMMARY` | 4h | |

**6 collectors.**

`apps` and `bots` matter more than they look: a Slack app with broad OAuth scopes is a non-human identity with access to years of conversation, and Slack Connect channels are external sharing that no DLP sees.

---

## 6 · Microsoft Teams

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `teams_channels` | Teams, channels, membership, external access | `GROUP`, sharing exposure | 4h | ★ |
| `apps` | Installed apps, permission policies | `CAN_ACCESS` | 4h | ★ |
| `bots` | Bot registrations and their identities | `IDENTITY` (non-human) | 4h | ★ |
| `guests` | Guest users with access | external `CAN_READ` | 4h | ★ |
| `copilot_usage` | Copilot activity and agent usage | `AI_APPLICATION`, `USES` | 4h | ★ |

**5 collectors.**

---

## 7 · Confluence

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `spaces` | Spaces, ownership, visibility | `DATASTORE` | 12h | ★ |
| `permissions` | Space and page permissions, anonymous access | `CAN_READ`, exposure findings | 4h | ★ |
| `content_classification` | Sampled page content and labels | `DATA_CLASS` | 7d | |
| `external_shares` | Public links and guest access | exposure findings | 4h | ★ |

**4 collectors.**

Wikis are a chronic and under-examined location for credentials and architecture documents. The `content_classification` collector frequently surfaces secrets pasted into runbooks.

---

## 8 · PagerDuty

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `services` | Services and their technical owners | **ownership on applications** | 12h | ★ |
| `teams` | Teams and membership | ownership groups | 12h | ★ |
| `on_call` | Current and future on-call schedules | who to route a critical finding to, now | 4h | ★ |
| `escalation_policies` | Escalation chains | ownership escalation path | 12h | |

**4 collectors.**

`on_call` is a small collector with a large effect: it converts *"this critical path affects the payments service"* into *"page the person who is on call for payments right now."*

---

## Domain summary

| Connector | Collect | Respond |
|---|---|---|
| ServiceNow | 7 | 1 |
| Jira | 4 | 1 |
| Workday | 5 | — |
| BambooHR / HiBob | 3 | — |
| Slack | 6 | — |
| Microsoft Teams | 5 | — |
| Confluence | 4 | — |
| PagerDuty | 4 | — |
| **Total** | **38** | **2** |

### The four things only this domain provides

```
  1  OWNERSHIP
     ServiceNow ownership · Jira project leads · PagerDuty services ·
     Workday managers
     → the difference between a finding that gets fixed and a finding
       that sits in a backlog. No technical connector knows who owns
       an asset.

  2  CRITICALITY FROM THE CUSTOMER'S OWN DEFINITIONS
     ServiceNow business services and their tiers
     → crown jewels designated from what the business already
       declared critical, not from our inference. Far more
       defensible in a review.

  3  LEAVER AND CONTRACTOR SIGNAL
     Workday workers and contingent workers
     → an active account for someone who left is invisible to every
       identity provider, because the IdP was never told. HR knows.

  4  MAINTENANCE WINDOWS AND CHANGE ATTRIBUTION
     ServiceNow change requests
     → response actions must respect change freezes (../05 §21), and
       a graph change three hours after an approved change request
       is routine, while the same change with no ticket is a finding.
```

### The collaboration connectors are a second AI discovery surface

Slack `ai_usage`, Teams `copilot_usage` and their `apps`/`bots` collectors reveal AI adoption inside collaboration tools — assistants, agent apps, bots with OAuth scopes. This is AI that is neither in a cloud provider's console nor on an endpoint, and it is where a large share of real enterprise AI usage actually happens.

### Why this domain is Band 5

Every collector here attaches to entities other domains created. Ownership attaches to an asset; criticality attaches to a datastore; a leaver flag attaches to an identity. Run these before the entities exist and you get orphaned properties with nothing to bind to.

---

*Next: [Generic ingestion](09-generic-ingestion.md)*

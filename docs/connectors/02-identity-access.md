# Domain 02 — Identity and Access

**16 connectors · 118 collectors** · [Index](00-index.md)

The most important domain. ~73% of TrustGraph edges originate from identity and permission data (`../02-iam-deep-dive.md`). Everything here runs in **Band 1** — identity authorities must land before anything that references them, because canonical keys come from here.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · Microsoft Entra ID

```
  api_surface   configuration
  functions     collect · respond (separate — user disable, session revoke)
  auth          app registration with certificate → client secret
  least priv    Directory.Read.All, Application.Read.All,
                RoleManagement.Read.Directory, Policy.Read.All,
                AuditLog.Read.All, IdentityRiskyUser.Read.All
  coverage      tenant-wide; delta queries where available
  ⚠             the privilege here lives in APPLICATIONS, not users.
                An app with RoleManagement.ReadWrite.Directory is
                Global Admin by another name, and nobody reviews app
                permissions with the rigour they review admin groups.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, UPN, mail, immutable ID, MFA state | `IDENTITY`, **canonical keys** | 1h delta | ★ |
| `groups` | Groups, membership, **dynamic membership rules** | `GROUP`, `MEMBER_OF`, escalation input | 1h delta | ★ |
| `directory_roles` | Role definitions + assignments | `ROLE`, `CAN_ASSUME` | 1h | ★ |
| `pim` | Eligible vs active assignments, activation policy | standing vs eligible privilege | 4h | ★ |
| `app_registrations` | Applications, redirect URIs, **federated credentials** | `APPLICATION`, external trust | 4h | ★ |
| `service_principals` | SPs, credentials metadata, **owners** | `IDENTITY`, ownership escalation | 4h | ★ |
| `app_role_assignments` | Graph API permissions granted to apps | escalation input — the big one | 4h | ★ |
| `oauth_grants` | Delegated permission consent grants | `CAN_ACCESS`, illicit consent findings | 4h | ★ |
| `conditional_access` | CA policies, conditions, **exclusions** | `PROTECTS` + gap findings | 12h | ★ |
| `authentication_methods` | Registered methods per user | MFA posture properties | 4h | |
| `administrative_units` | AUs and scoped role assignments | scoped privilege | 12h | |
| `devices` | Registered/joined devices, compliance | `ASSET`, `ASSIGNED_TO` | 4h | |
| `sign_in_logs` | Interactive + non-interactive sign-ins | `EVENT_SUMMARY`, USED state, dormancy | 15m | |
| `audit_logs` | Directory changes | change detection | 15m | |
| `identity_protection` | Risky users, risk detections | `FINDING` overlay | 1h | |
| `cross_tenant` | B2B/B2C settings, partner (GDAP) relationships | external privilege into the tenant | 12h | ★ |
| `connect_sync` | AAD Connect config, sync account, PHS/PTA state | **on-prem↔cloud pivot** | 12h | ★ |

**17 collectors.**

---

## 2 · Active Directory (LDAP)

```
  api_surface   configuration
  functions     collect · respond (separate — account disable)
  auth          dedicated read-only service account, LDAPS
  least priv    domain user + explicit read on the AdminSDHolder and
                deleted-objects containers. No write, anywhere.
  coverage      per domain; full sweep emits, uSNChanged delta does not
  ⚠             a 720,000-ACE enumeration looks exactly like
                reconnaissance. Ship a collection profile the customer
                hands to their own SOC to allowlist, and schedule the
                full sweep outside backup windows.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | User objects, UAC flags, SPNs, pwdLastSet | `IDENTITY`, kerberoast candidates | 1h delta | ★ |
| `computers` | Computer objects, OS, **delegation attributes** | `ASSET`, delegation escalation | 1h delta | ★ |
| `groups` | Groups, membership, **nesting** | `GROUP`, `MEMBER_OF` transitively | 1h delta | ★ |
| `ous` | Organisational units, structure | scope, department enrichment | 12h | |
| `acls` | **All object ACEs** | ACL-derived edges — the big one | 24h | ★ |
| `gpos` | GPOs, links, **file system permissions** | `CAN_EXECUTE` on linked OUs | 12h | ★ |
| `trusts` | Domain and forest trusts, SID filtering | cross-domain `TRUSTS` | 12h | ★ |
| `delegation` | Unconstrained, constrained, RBCD attributes | `CAN_ASSUME` escalation | 4h | ★ |
| `domain_policy` | Password policy, **MachineAccountQuota**, lockout | RBCD precondition, posture | 12h | ★ |
| `laps` | LAPS password read permissions (never values) | `CAN_EXECUTE` on hosts | 12h | ★ |
| `service_accounts` | gMSAs, MSAs, and their read permissions | `IDENTITY`, `CAN_ASSUME` | 12h | |
| `sites_subnets` | AD sites and subnets | network↔identity correlation | 24h | |
| `deleted_objects` | Tombstoned objects | leaver detection, retraction support | 24h | |
| `security_events` | 4624/4768/4769/4662 from DCs | `EVENT_SUMMARY`, USED state | continuous | |
| `bloodhound_import` | SharpHound JSON, if the customer already runs it | full graph import, alternative to `acls` | on-demand | |

**15 collectors.**

---

## 3 · AD Certificate Services

```
  api_surface   configuration
  auth          read access to the CA and the AD configuration container
  ⚠             an entire escalation surface in a separate service
                that almost nobody monitors.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `cas` | Enterprise CAs, their ACLs, EKU policies | `CAN_ASSUME` via ManageCA | 12h | ★ |
| `templates` | Certificate templates, **enrollment rights, SAN flags** | escalation edges (ESC1-ESC8 class) | 12h | ★ |
| `enrollment_services` | Enrollment endpoints, web enrollment, NTLM relay exposure | `EXPOSES`, relay findings | 12h | ★ |
| `issued_certs` | Issued certificate metadata, templates used | anomalous issuance findings | 24h | |

**4 collectors.**

---

## 4 · Okta

```
  api_surface   configuration
  functions     collect · respond (separate — suspend user, clear sessions)
  auth          OAuth service app with private key JWT → API token
  least priv    read-only admin scopes: okta.users.read, okta.groups.read,
                okta.apps.read, okta.policies.read, okta.logs.read
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, profile, status, **immutable ID** | `IDENTITY`, canonical keys | 1h delta | ★ |
| `groups` | Groups, membership, group rules | `GROUP`, `MEMBER_OF` | 1h | ★ |
| `applications` | App integrations, sign-on modes, provisioning | `APPLICATION`, federation config | 4h | ★ |
| `app_assignments` | User/group → app assignments | `CAN_ACCESS` | 4h | ★ |
| `policies` | Sign-on, MFA, password policies + rules | `PROTECTS` + gap findings | 12h | ★ |
| `admin_roles` | Admin role assignments and scopes | `ROLE`, privileged identities | 4h | ★ |
| `factors` | Enrolled MFA factors per user | MFA posture | 4h | |
| `system_log` | Authentication and admin events | `EVENT_SUMMARY`, USED state | 15m | |
| `identity_providers` | Inbound federation, social IdPs | external trust | 12h | ★ |

**9 collectors.**

---

## 5 · Ping Identity

```
  api_surface   configuration
  auth          worker app OAuth credentials
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, population, attributes | `IDENTITY`, canonical keys | 1h | ★ |
| `groups` | Groups, membership | `GROUP`, `MEMBER_OF` | 4h | ★ |
| `applications` | Applications, OIDC/SAML config | `APPLICATION`, federation | 4h | ★ |
| `roles` | Admin roles and scopes | privileged identities | 4h | ★ |
| `policies` | Authentication and MFA policies | `PROTECTS` + gaps | 12h | |
| `audit` | Authentication and admin activity | `EVENT_SUMMARY` | 15m | |

**6 collectors.**

---

## 6 · OneLogin

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, status, custom attributes | `IDENTITY` | 1h | ★ |
| `roles` | Roles + membership | `ROLE`, `MEMBER_OF` | 4h | ★ |
| `apps` | Applications, connectors, provisioning | `APPLICATION` | 4h | ★ |
| `mappings` | User mappings and rules | rule-based membership | 12h | |
| `events` | Authentication and admin events | `EVENT_SUMMARY` | 15m | |

**5 collectors.**

---

## 7 · JumpCloud

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, attributes, MFA | `IDENTITY` | 1h | ★ |
| `user_groups` | Groups, membership | `GROUP`, `MEMBER_OF` | 4h | ★ |
| `systems` | Managed systems | `ASSET`, `ASSIGNED_TO` | 4h | ★ |
| `system_groups` | System groups, bindings | grouping, policy scope | 4h | |
| `applications` | SSO applications and assignments | `APPLICATION`, `CAN_ACCESS` | 4h | |
| `policies` | Device and security policies | `PROTECTS` | 12h | |
| `directory_insights` | Authentication and admin events | `EVENT_SUMMARY` | 15m | |

**7 collectors.**

---

## 8 · Google Workspace

```
  api_surface   configuration
  auth          service account with domain-wide delegation
  ⚠             domain-wide delegation is itself a major escalation
                surface. The `dwd_grants` collector is load-bearing.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, aliases, 2SV enrolment, suspension | `IDENTITY`, canonical keys | 1h | ★ |
| `groups` | Groups, membership, settings | `GROUP`, `MEMBER_OF` | 4h | ★ |
| `org_units` | OU tree | scope, department enrichment | 12h | |
| `admin_roles` | Admin role assignments, privileges | privileged identities | 4h | ★ |
| `oauth_apps` | Third-party OAuth apps + granted scopes | `CAN_ACCESS`, consent findings | 4h | ★ |
| `dwd_grants` | Domain-wide delegation client IDs + scopes | **impersonation escalation** | 12h | ★ |
| `devices` | Managed mobile and Chrome devices | `ASSET` | 4h | |
| `admin_audit` | Admin and login events | `EVENT_SUMMARY`, USED state | 15m | |

**8 collectors.**

---

## 9 · Scalefusion (UEM + OneIdP)

```
  api_surface   configuration
  functions     collect · respond (separate — device lock/wipe)
  auth          API key
  ⚠             FORK: if Scalefusion is your authoritative directory,
                its user IDs are priority-1 canonical keys. If it
                syncs FROM AD/Entra/Workspace, they are aliases only
                and the upstream source is the authority. This
                changes the canonical key priority list.
  ⚠             covers MANAGED devices only. Identity coverage here
                is not workforce coverage — report it honestly.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Directory users, email, immutable ID | `IDENTITY`, canonical keys or aliases | 1h | ★ |
| `user_groups` | Groups and membership | `GROUP`, `MEMBER_OF` | 4h | |
| `devices` | Enrolled devices **with assigned user** | `ASSET` + `ASSIGNED_TO` — authoritative | 4h | ★ |
| `applications` | Installed app inventory per device | software properties, AI tooling detection | 12h | |
| `policies` | Device profiles and policies | `SECURITY_CONTROL`, `PROTECTS` | 12h | |
| `compliance` | Per-device compliance state | asset properties | 4h | |
| `sso_assignments` | OneIdP application assignments | `CAN_ACCESS` | 4h | |
| `conditional_access` | OneIdP access rules and conditions | `PROTECTS` + gap findings | 12h | |

**8 collectors.**

---

## 10 · AWS IAM Identity Center

```
  api_surface   configuration
  auth          delegated administrator account, read-only
  ⚠             dep: aws.organizations. Permission sets are
                meaningless without the account tree.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `instances` | SSO instances, identity store config | scope | 24h | |
| `permission_sets` | Permission sets, inline + managed policies | grants across accounts | 4h | ★ |
| `assignments` | User/group → permission set → account | `CAN_ASSUME` across the estate | 4h | ★ |
| `identity_store` | Users and groups in the SSO store | `IDENTITY`, `GROUP` | 4h | ★ |
| `external_idp` | External IdP federation config | trust edges | 12h | ★ |

**5 collectors.**

---

## 11 · CyberArk

```
  api_surface   configuration
  auth          CCP/AAM application identity, read-only
  ⚠             NEVER retrieve credential values. Metadata only.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `safes` | Safes, ownership | scope | 12h | ★ |
| `safe_members` | Safe permissions per identity | `CAN_READ` on credentials | 4h | ★ |
| `accounts` | Privileged account metadata, platform, target | `SECRET` → identity bridge | 4h | ★ |
| `platforms` | Platform definitions, rotation policy | credential hygiene properties | 12h | |
| `sessions` | PSM session records | `EVENT_SUMMARY`, USED state | 1h | |
| `applications` | AAM/CCP application identities | `IDENTITY` (non-human) | 12h | |

**6 collectors.**

---

## 12 · HashiCorp Vault

```
  api_surface   configuration
  auth          token or AppRole with a read-only policy
  ⚠             metadata only — never read secret values.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `auth_methods` | Enabled auth backends, roles, bindings | `IDENTITY`, `AUTHENTICATES_TO` | 4h | ★ |
| `policies` | ACL policies | grants on secret paths | 4h | ★ |
| `secret_engines` | Mounted engines, paths, config | `SECRET` namespaces | 12h | ★ |
| `entities_aliases` | Identity entities and aliases | identity resolution input | 4h | ★ |
| `namespaces` | Vault namespaces (Enterprise) | scope | 12h | |
| `audit_devices` | Audit device config | posture | 24h | |

**6 collectors.**

---

## 13 · Delinea / Thycotic

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `folders` | Secret Server folder tree | scope | 12h | ★ |
| `permissions` | Folder and secret permissions | `CAN_READ` on credentials | 4h | ★ |
| `secrets` | Secret metadata, templates, target systems | `SECRET` → identity bridge | 4h | ★ |
| `users_groups` | Users, groups, roles | `IDENTITY`, `GROUP` | 4h | |
| `audit` | Access and check-out events | `EVENT_SUMMARY` | 1h | |

**5 collectors.**

---

## 14 · BeyondTrust

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `managed_systems` | Managed systems and accounts | `ASSET`, `SECRET` | 4h | ★ |
| `permissions` | Smart rules, access policies | `CAN_READ` on credentials | 4h | ★ |
| `users_groups` | Users, groups, roles | `IDENTITY` | 4h | |
| `sessions` | Session recordings metadata | `EVENT_SUMMARY` | 1h | |
| `password_policies` | Rotation and complexity policy | credential hygiene | 12h | |

**5 collectors.**

---

## 15 · SailPoint

```
  api_surface   configuration
  ⚠             SailPoint is the customer's own entitlement view.
                Ingesting it gives ownership and certification data
                we cannot derive — but its entitlement model is
                NOT a substitute for our own permission closure.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `identities` | Identity cube, attributes, manager | `IDENTITY`, **ownership**, org hierarchy | 4h | ★ |
| `accounts` | Correlated accounts per identity across sources | **identity resolution input** | 4h | ★ |
| `entitlements` | Entitlement catalog per source | grants as the customer models them | 12h | |
| `roles` | Business and IT roles, membership | `ROLE`, `MEMBER_OF` | 12h | |
| `certifications` | Access review campaigns and decisions | governance posture, stale-access findings | 24h | |
| `sources` | Connected source systems | coverage comparison against ours | 24h | |

**6 collectors.**

---

## 16 · Saviynt

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users and attributes | `IDENTITY`, ownership | 4h | ★ |
| `accounts` | Account correlation across endpoints | identity resolution input | 4h | ★ |
| `entitlements` | Entitlement catalog | grants as modelled | 12h | |
| `roles` | Roles and membership | `ROLE`, `MEMBER_OF` | 12h | |
| `campaigns` | Certification campaigns | governance posture | 24h | |

**5 collectors.**

---

## 17 · Duo

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `users` | Users, enrolment status, **bypass status** | MFA posture, bypass findings | 4h | ★ |
| `phones_tokens` | Registered authentication devices | MFA method properties | 12h | |
| `policies` | Authentication policies and application bindings | `PROTECTS` + gaps | 12h | ★ |
| `applications` | Protected applications | `PROTECTS` scope | 12h | |
| `auth_logs` | Authentication events | `EVENT_SUMMARY` | 15m | |

**5 collectors.**

---

## Domain summary

| Connector | Collectors | Band |
|---|---|---|
| Microsoft Entra ID | 17 | 1 |
| Active Directory | 15 | 1 |
| AD Certificate Services | 4 | 1 |
| Okta | 9 | 1 |
| Ping Identity | 6 | 1 |
| OneLogin | 5 | 1 |
| JumpCloud | 7 | 1 |
| Google Workspace | 8 | 1 |
| Scalefusion | 8 | 1 |
| AWS IAM Identity Center | 5 | 1 |
| CyberArk | 6 | 2 |
| HashiCorp Vault | 6 | 2 |
| Delinea / Thycotic | 5 | 2 |
| BeyondTrust | 5 | 2 |
| SailPoint | 6 | 2 |
| Saviynt | 5 | 2 |
| Duo | 5 | 1 |
| **Total** | **122** | |

### The rule that governs this whole domain

```
  Band 1 exists because of this domain. Canonical keys come from
  identity authorities, and every other connector's records must
  resolve against them.

  Collect AWS IAM before Entra and AWS identities resolve on weak
  keys — ARNs and tag values instead of verified email — and may
  never merge with their AD/Entra counterparts.

  The graph does not break loudly when this happens. It fragments
  silently, and the hybrid attack paths simply do not exist.
```

---

*Next: [Code, build, artifacts](03-code-build-artifacts.md)*

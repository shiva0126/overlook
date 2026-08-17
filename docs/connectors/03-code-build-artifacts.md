# Domain 03 — Code, Build and Artifacts

**12 connectors · 84 collectors** · [Index](00-index.md)

Band 3 — workload and supply chain. These connectors are where the software supply chain meets cloud identity, and the single highest-value thing in the domain is the **OIDC trust**: a repository that can assume a cloud role.

⚠ **Domain-wide dependency:** OIDC trust collectors have `dep: aws.iam.roles / azure.rbac.assignments / gcp.iam.service_accounts`. The trust exists on the cloud side and the subject on the repo side. Collect one without the other and the edge is invisible.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · GitHub

```
  api_surface   configuration (+ push for webhooks)
  functions     collect only
  auth          GitHub App (preferred) → fine-grained PAT
  least priv    read:org, repo:read, actions:read, security_events:read,
                members:read. NEVER write scopes.
  coverage      per organisation
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organizations` | Orgs, settings, 2FA enforcement, base permissions | scope, posture | 12h | ★ |
| `repositories` | Repos, visibility, default branch, archived state | `REPOSITORY` | 4h | ★ |
| `members` | Org members, roles, outside collaborators | `IDENTITY`, `MEMBER_OF` | 4h | ★ |
| `teams` | Teams, membership, repo permissions | `GROUP`, `CAN_WRITE` on repos | 4h | ★ |
| `collaborators` | Per-repo direct collaborators | `CAN_WRITE` | 4h | ★ |
| `branch_protection` | Protection rules, required reviews, rulesets | `PROTECTS` on `CAN_DEPLOY` | 12h | ★ |
| `oidc_trusts` | Actions OIDC subject claims + cloud trust config | `CAN_ASSUME` **cross-boundary** | 12h | ★ |
| `actions_workflows` | Workflow definitions, triggers, permissions | `PIPELINE`, `CAN_DEPLOY` | 4h | ★ |
| `actions_secrets` | Secret and variable **names** per repo/org/env | `SECRET` presence | 4h | ★ |
| `actions_runners` | Self-hosted runners, labels, scope | `ASSET`, escalation surface | 12h | |
| `deploy_keys` | Deploy keys, read/write flag | `SECRET`, `CAN_WRITE` | 12h | |
| `apps_installations` | Installed GitHub Apps and their permissions | `CAN_ACCESS`, third-party privilege | 12h | ★ |
| `secret_scanning` | Secret scanning alerts | `SECRET` → identity bridge | 4h | ★ |
| `code_scanning` | Code scanning alerts | `VULNERABILITY` properties | 12h | |
| `dependabot` | Dependency alerts, SBOM | `VULNERABILITY`, `ARTIFACT` | 12h | |
| `webhooks` | Repo and org webhook events | change detection | continuous | |

**16 collectors.**

---

## 2 · GitLab

```
  auth          group access token or OAuth app, read_api scope
  least priv    read_api, read_repository. No write.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `groups` | Group tree, settings | scope | 12h | ★ |
| `projects` | Projects, visibility, default branch | `REPOSITORY` | 4h | ★ |
| `members` | Group and project members with access levels | `IDENTITY`, `CAN_WRITE` | 4h | ★ |
| `protected_branches` | Protection rules, approval requirements | `PROTECTS` | 12h | ★ |
| `ci_variables` | CI/CD variable **names**, masked/protected flags | `SECRET` presence | 4h | ★ |
| `pipelines` | Pipeline definitions, schedules | `PIPELINE`, `CAN_DEPLOY` | 4h | ★ |
| `id_tokens` | JWT/OIDC config for cloud auth | `CAN_ASSUME` cross-boundary | 12h | ★ |
| `runners` | Shared and specific runners | `ASSET`, escalation surface | 12h | |
| `deploy_tokens` | Deploy tokens and scopes | `SECRET` | 12h | |
| `integrations` | Project integrations and webhooks | third-party access | 12h | |
| `security_findings` | SAST/DAST/dependency/secret scan results | `VULNERABILITY`, `SECRET` | 12h | |

**11 collectors.**

---

## 3 · Bitbucket

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `workspaces` | Workspaces, settings | scope | 12h | ★ |
| `repositories` | Repos, visibility | `REPOSITORY` | 4h | ★ |
| `permissions` | Workspace, project and repo permissions | `IDENTITY`, `CAN_WRITE` | 4h | ★ |
| `branch_restrictions` | Branch permissions and merge checks | `PROTECTS` | 12h | |
| `pipelines` | Pipeline config, deployment environments | `PIPELINE`, `CAN_DEPLOY` | 4h | ★ |
| `pipeline_variables` | Variable **names**, secured flags | `SECRET` presence | 4h | ★ |
| `access_keys` | SSH access keys | `SECRET` | 12h | |
| `oauth_consumers` | OAuth consumers and scopes | third-party access | 12h | |

**8 collectors.**

---

## 4 · Azure DevOps

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organizations` | Org settings, policies | scope | 12h | ★ |
| `projects` | Projects and visibility | scope | 12h | ★ |
| `repositories` | Git repos, default branch | `REPOSITORY` | 4h | ★ |
| `permissions` | Security namespaces, ACLs, group membership | `IDENTITY`, `CAN_WRITE` | 4h | ★ |
| `branch_policies` | Branch policies and approvers | `PROTECTS` | 12h | ★ |
| `pipelines` | Build and release pipelines | `PIPELINE`, `CAN_DEPLOY` | 4h | ★ |
| `service_connections` | **Service connections to cloud subscriptions** | `CAN_ASSUME` cross-boundary | 4h | ★ |
| `variable_groups` | Variable group names, linked vaults | `SECRET` presence | 4h | ★ |
| `agent_pools` | Agent pools, self-hosted agents | `ASSET`, escalation surface | 12h | |
| `pats` | Personal access token metadata, scopes, expiry | `SECRET`, credential hygiene | 12h | ★ |
| `extensions` | Installed extensions and permissions | third-party privilege | 12h | |

**11 collectors.**

---

## 5 · Jenkins

```
  ⚠ Jenkins is chronically over-privileged and rarely reviewed.
    Its credential store is frequently the single most valuable
    target in a customer's build estate.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `jobs` | Jobs, pipelines, SCM config | `PIPELINE`, `CAN_DEPLOY` | 4h | ★ |
| `credentials` | Credential **IDs, types, scopes** (never values) | `SECRET` → identity bridge | 4h | ★ |
| `users_permissions` | Users, matrix/role-based authorisation | `IDENTITY`, `CAN_MODIFY` | 4h | ★ |
| `nodes` | Build agents, labels, executors | `ASSET` | 12h | |
| `plugins` | Installed plugins and versions | `VULNERABILITY` surface | 24h | |
| `build_history` | Recent builds, triggering identity | `EVENT_SUMMARY`, USED state | 4h | |
| `global_config` | Security realm, CSRF, script approval | posture findings | 12h | ★ |

**7 collectors.**

---

## 6 · CircleCI

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Projects, VCS linkage | `REPOSITORY` link | 4h | ★ |
| `contexts` | Contexts, restrictions, variable names | `SECRET` presence, scope | 4h | ★ |
| `pipelines` | Pipeline and workflow config | `PIPELINE` | 4h | ★ |
| `oidc` | OIDC token config for cloud auth | `CAN_ASSUME` cross-boundary | 12h | ★ |
| `env_vars` | Project env var names | `SECRET` presence | 4h | |
| `runners` | Self-hosted runners | `ASSET` | 12h | |

**6 collectors.**

---

## 7 · Terraform Cloud / Enterprise

```
  ⚠ state files routinely contain database passwords, private keys
    and API tokens in plaintext. The `state_metadata` collector
    reads METADATA only — never state contents.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organizations` | Orgs, teams, membership | `IDENTITY`, `GROUP` | 12h | ★ |
| `workspaces` | Workspaces, linked repos, execution mode | `PIPELINE` | 4h | ★ |
| `variables` | Variable **names**, sensitive flags | `SECRET` presence | 4h | ★ |
| `dynamic_credentials` | OIDC/workload identity config to clouds | `CAN_ASSUME` cross-boundary | 12h | ★ |
| `state_metadata` | State version metadata, resource counts | `CAN_DEPLOY` scope | 12h | ★ |
| `run_history` | Runs, who applied, drift detection | `EVENT_SUMMARY`, change attribution | 4h | |
| `policy_sets` | Sentinel/OPA policies | `PROTECTS` | 12h | |

**7 collectors.**

---

## 8 · Harness

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `pipelines` | Pipelines and stages | `PIPELINE`, `CAN_DEPLOY` | 4h | ★ |
| `connectors` | Cloud/repo connectors and their credentials | `CAN_ASSUME` cross-boundary | 4h | ★ |
| `secrets` | Secret references and managers | `SECRET` presence | 4h | ★ |
| `rbac` | Users, user groups, role assignments | `IDENTITY`, `CAN_MODIFY` | 4h | ★ |
| `delegates` | Delegates and their scope | `ASSET`, escalation surface | 12h | |

**5 collectors.**

---

## 9 · ArgoCD

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `applications` | Applications, source repos, destinations | `CAN_DEPLOY` into clusters | 4h | ★ |
| `projects` | AppProjects, allowed sources/destinations | scope constraints | 12h | ★ |
| `rbac` | RBAC policy, local accounts, SSO config | `IDENTITY`, `CAN_MODIFY` | 4h | ★ |
| `repositories` | Configured repos and credential refs | `SECRET`, repo linkage | 12h | |
| `clusters` | Registered clusters and their credentials | K8s bridge | 12h | ★ |

**5 collectors.**

---

## 10 · JFrog Artifactory

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `repositories` | Local, remote, virtual repos | `ARTIFACT` namespaces | 12h | ★ |
| `permissions` | Permission targets, users, groups | `CAN_READ`/`CAN_WRITE` on artifacts | 4h | ★ |
| `users_groups` | Users, groups, API key presence | `IDENTITY` | 4h | |
| `access_tokens` | Token metadata, scopes, expiry | `SECRET` | 12h | |
| `xray_findings` | Vulnerability and licence findings | `VULNERABILITY` | 12h | |

**5 collectors.**

---

## 11 · Sonatype Nexus

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `repositories` | Repositories and formats | `ARTIFACT` namespaces | 12h | ★ |
| `privileges_roles` | Privileges, roles, user assignments | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `users` | Users and sources | `IDENTITY` | 4h | |
| `iq_findings` | Lifecycle policy violations | `VULNERABILITY` | 12h | |

**4 collectors.**

---

## 12 · Container registries (Docker Hub, Quay, GHCR, GAR/ACR/ECR-standalone)

```
  ⚠ cloud-native registries (ECR, ACR, GAR) are collectors INSIDE
    their cloud connector, not here. This connector covers
    standalone and third-party registries.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `repositories` | Image repositories, visibility | `ARTIFACT` | 12h | ★ |
| `permissions` | Teams, robot accounts, access levels | `CAN_WRITE` on images | 4h | ★ |
| `images` | Tags, digests, manifests, base images | `ARTIFACT`, `CONTAINS` | 12h | |
| `scan_results` | Vulnerability scan output | `VULNERABILITY` | 12h | |
| `robot_accounts` | Machine accounts and tokens | `IDENTITY` (non-human), `SECRET` | 12h | ★ |

**5 collectors.**

---

## Domain summary

| Connector | Collectors |
|---|---|
| GitHub | 16 |
| GitLab | 11 |
| Bitbucket | 8 |
| Azure DevOps | 11 |
| Jenkins | 7 |
| CircleCI | 6 |
| Terraform Cloud | 7 |
| Harness | 5 |
| ArgoCD | 5 |
| JFrog Artifactory | 5 |
| Sonatype Nexus | 4 |
| Container registries | 5 |
| **Total** | **90** |

### The pattern across every connector here

Each has four collector classes, and the value ranks in this order:

```
  1  CROSS-BOUNDARY TRUST     oidc_trusts · service_connections ·
                              dynamic_credentials · id_tokens
                              → the only source of repo→cloud edges.
                                Nothing else can produce them.

  2  WHO CAN WRITE            members · teams · collaborators ·
                              permissions
                              → who inherits everything downstream

  3  SECRET PRESENCE          actions_secrets · ci_variables ·
                              credentials · variable_groups
                              → names and types only, never values.
                                A secret is an identity edge.

  4  CONTROLS                 branch_protection · protected_branches ·
                              branch_policies
                              → PROTECTS edges that reduce the weight
                                of CAN_DEPLOY
```

If a customer enables only one collector per connector in this domain, it should be the cross-boundary trust collector. It is the hop that turns a repository from an inventory item into an attack path.

---

*Next: [Security tooling](04-security-tooling.md)*

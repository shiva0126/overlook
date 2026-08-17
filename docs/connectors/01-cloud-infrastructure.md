# Domain 01 — Cloud and Infrastructure

**10 connectors · 138 collectors** · [Index](00-index.md)

The densest domain by collector count and the one that feeds permission closure. Everything here is `api_surface: configuration` except the audit-trail collectors.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · AWS

```
  api_surface   configuration (+ log_stream for cloudtrail)
  functions     collect · respond (separate manifest)
  auth          IRSA → AssumeRole from a hub account → access key
  least priv    ReadOnlyAccess is too broad. Ship a scoped policy:
                iam:Get*/List*, organizations:Describe*/List*,
                ec2:Describe*, s3:GetBucket*/ListBucket, rds:Describe*,
                lambda:Get*/List*, kms:Describe*/List*, eks:Describe*/List*,
                access-analyzer:List*/Get*, cloudtrail:LookupEvents
  coverage      all enumeration collectors emit windows, per account+region
  ⚠             every collector is scoped per (account, region). A
                42-account estate is 42 instances, not one.
```

### Identity and governance

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organizations` | Accounts, OUs, SCPs, RCPs | account tree, org-level constraints | 24h | ★ |
| `iam.users` | IAM users, access keys, MFA devices | `IDENTITY`, credential age properties | 4h | ★ |
| `iam.roles` | Roles + **trust policies** | `ROLE`, `CAN_ASSUME` preconditions | 4h | ★ |
| `iam.groups` | Groups + membership | `GROUP`, `MEMBER_OF` | 4h | |
| `iam.policies` | Managed + inline policies, all versions | the grants themselves | 4h | ★ |
| `iam.boundaries` | Permission boundaries per principal | constraints | 4h | ★ |
| `iam.providers` | SAML + OIDC identity providers | federation trust edges | 12h | ★ |
| `iam.last_accessed` | `GetServiceLastAccessedDetails` | the **USED** state → CIEM | 24h | |
| `access_analyzer` | External-access + unused-access findings | free ground truth, confidence lift | 12h | |
| `sts` | Caller identity, assumable set validation | preflight + closure verification | preflight | |
| `identity_center` | Permission sets, assignments, groups | `ROLE`, `CAN_ASSUME` across accounts | 4h | ★ |

### Compute and containers

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `ec2.instances` | Instances, instance profiles, userdata metadata | `ASSET`, `RUNS_ON` role binding | 4h | ★ |
| `ec2.images` | AMIs, sharing permissions | exposure findings | 12h | |
| `ec2.volumes` | EBS volumes, snapshots, encryption, sharing | `DATASTORE`, encryption properties | 12h | |
| `lambda` | Functions, roles, env var **names**, layers, URLs | `ASSET`, `RUNS_ON`, escalation ingredients | 4h | ★ |
| `ecs` | Clusters, services, task definitions, task roles | `ASSET`, `RUNS_ON` | 4h | |
| `eks` | Clusters, node groups, **access entries**, OIDC issuer | `ASSET`, the K8s↔IAM bridge | 4h | ★ |
| `ecr` | Repositories, policies, image scan findings | `ARTIFACT`, `CONTAINS` | 12h | |
| `batch` | Compute environments, job roles | `RUNS_ON` | 12h | |
| `autoscaling` | ASGs, launch templates | equivalence-class grouping | 12h | |

### Data

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `s3.buckets` | Buckets, ACLs, public access block, encryption | `DATASTORE`, exposure properties | 4h | ★ |
| `s3.policies` | Bucket policies | resource-policy grants → closure | 4h | ★ |
| `s3.inventory` | Object-level inventory | classification input | 7d | |
| `rds` | Instances, clusters, snapshots, parameter groups | `DATASTORE`, exposure, encryption | 4h | ★ |
| `dynamodb` | Tables, resource policies, encryption | `DATASTORE` | 12h | |
| `redshift` | Clusters, snapshots, IAM roles | `DATASTORE` | 12h | |
| `efs_fsx` | File systems, mount targets, policies | `DATASTORE` | 12h | |

### Network

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `vpc` | VPCs, subnets, route tables, peering, TGW | `NETWORK`, `ROUTES_TO` | 12h | ★ |
| `ec2.security_groups` | Security groups, NACLs | `ROUTES_TO`, `EXPOSES` | 4h | ★ |
| `elb` | ALB/NLB/CLB, target groups, listeners | `LOAD_BALANCER`, `EXPOSES` | 12h | |
| `route53` | Hosted zones, records, resolver rules | `DNS_NAME` | 12h | |
| `vpc_endpoints` | Interface + gateway endpoints, policies | `ROUTES_TO`, condition context | 12h | |
| `vpc_flow_logs` | Flow log records | aggregated `CONNECTS_TO` | continuous | |

### Secrets, keys, operations

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `secretsmanager` | Secret metadata + resource policies (**never values**) | `SECRET`, `CAN_READ` → identity bridge | 12h | ★ |
| `ssm.parameters` | Parameter metadata, SecureString refs | `SECRET` | 12h | |
| `ssm.instances` | Managed instances, session/command capability | `CAN_EXECUTE` escalation input | 4h | ★ |
| `kms` | Keys, key policies, grants, rotation state | `CAN_READ` via decrypt, encryption posture | 12h | ★ |
| `acm` | Certificates, expiry | `CERTIFICATE` | 24h | |
| `cloudformation` | Stacks, stack roles, drift | `CAN_DEPLOY` escalation input | 12h | |
| `config` | Configuration history, compliance rules | change history, posture findings | 12h | |
| `cloudtrail` | Management + data events | `EVENT_SUMMARY`, the USED state | 15m | |
| `sns_sqs` | Topics, queues, resource policies | `CONNECTS_TO`, cross-account exposure | 12h | |
| `apigateway` | APIs, authorizers, resource policies | `EXPOSES`, `APPLICATION` | 12h | |
| `bedrock` | Model access, agents, knowledge bases, guardrails | `AI_MODEL`, `AI_AGENT`, `RAG_APPLICATION` | 4h | ★ |
| `guardduty` | Findings | `FINDING` overlay | 15m | |
| `securityhub` | Aggregated findings | `FINDING` overlay | 1h | |
| `inspector` | Vulnerability findings | `VULNERABILITY` properties | 12h | |
| `macie` | Sensitive data discovery results | `DATA_CLASS` properties | 24h | |

**42 collectors.**

---

## 2 · Microsoft Azure

```
  api_surface   configuration (+ log_stream for activity_log)
  functions     collect · respond (separate)
  auth          Managed Identity → workload identity federation → SP secret
  least priv    Reader at management group root + Security Reader;
                Key Vault requires a separate data-plane role
  coverage      per subscription + resource group
  ⚠             Azure RBAC and Entra roles are separate systems.
                Entra lives in domain 02. The BRIDGE between them
                (elevateAccess) is evaluated in closure, not collected.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `management_groups` | MG hierarchy, subscriptions | scope tree for inheritance | 24h | ★ |
| `rbac.assignments` | Role assignments at every scope | grants | 4h | ★ |
| `rbac.definitions` | Built-in + custom role definitions | what each role means | 12h | ★ |
| `rbac.deny_assignments` | Deny assignments | constraints — beat Owner | 4h | ★ |
| `policy` | Azure Policy assignments, exemptions, compliance | constraints, posture findings | 12h | |
| `resources` | Resource Graph inventory | `ASSET` across all types | 4h | ★ |
| `compute.vms` | VMs, extensions, **managed identity bindings** | `ASSET`, `RUNS_ON` | 4h | ★ |
| `compute.scalesets` | VMSS | equivalence classes | 12h | |
| `storage.accounts` | Accounts, containers, public access, keys | `DATASTORE`, exposure | 4h | ★ |
| `sql` | SQL DB/MI, firewall rules, AAD admins | `DATASTORE` | 4h | |
| `cosmos` | Accounts, databases, firewall | `DATASTORE` | 12h | |
| `keyvault` | Vaults, **access policies**, RBAC, secret metadata | `SECRET`, `CAN_READ` → identity bridge | 12h | ★ |
| `aks` | Clusters, node pools, admin credential capability | K8s↔Entra bridge | 4h | ★ |
| `appservice` | Web apps, functions, config, managed identity | `APPLICATION`, `RUNS_ON` | 4h | |
| `network.vnet` | VNets, subnets, peering, route tables | `NETWORK`, `ROUTES_TO` | 12h | ★ |
| `network.nsg` | NSGs, ASGs, effective rules | `ROUTES_TO`, `EXPOSES` | 4h | ★ |
| `network.gateways` | VPN, ExpressRoute, Bastion, Front Door | cross-boundary `ROUTES_TO` | 12h | |
| `automation` | Automation accounts, RunAs, hybrid workers | escalation ingredients | 12h | |
| `logicapps` | Workflows, connections, managed identity | escalation ingredients | 12h | |
| `container_registry` | ACR, tokens, policies | `ARTIFACT` | 12h | |
| `openai` | Azure OpenAI deployments, models, keys | `AI_MODEL`, `AI_APPLICATION` | 4h | ★ |
| `defender_cloud` | Posture findings, secure score | `FINDING` overlay | 1h | |
| `activity_log` | Control-plane operations | `EVENT_SUMMARY`, USED state | 15m | |

**23 collectors.**

---

## 3 · Google Cloud

```
  api_surface   configuration (+ log_stream for audit_logs)
  functions     collect · respond (separate)
  auth          Workload Identity Federation → service account key
  least priv    roles/viewer + roles/iam.securityReviewer +
                roles/cloudasset.viewer at org scope
  coverage      per project, and per org for hierarchy collectors
  ⚠             IAM inherits downward and is additive. Collecting
                project-level bindings alone understates effective
                permission — org and folder bindings are mandatory.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `resource_manager` | Org, folders, projects, hierarchy | scope tree for inheritance | 24h | ★ |
| `iam.policies` | Allow policies at every level | grants | 4h | ★ |
| `iam.deny` | IAM Deny policies | constraints, evaluated first | 4h | ★ |
| `iam.roles` | Predefined + custom roles | what each role means | 12h | ★ |
| `iam.service_accounts` | SAs, keys, **impersonation grants** | `IDENTITY`, `CAN_ASSUME` chains | 4h | ★ |
| `iam.wif` | Workload Identity Federation pools, providers | cross-cloud trust edges | 12h | ★ |
| `org_policy` | Org policy constraints | constraints | 12h | |
| `cloud_asset` | Asset inventory across all services | `ASSET` | 4h | ★ |
| `compute.instances` | Instances, SA bindings, metadata | `ASSET`, `RUNS_ON` | 4h | ★ |
| `gcs` | Buckets, IAM, public access, uniform access | `DATASTORE`, exposure | 4h | ★ |
| `cloudsql` | Instances, authorized networks, IAM auth | `DATASTORE` | 4h | |
| `bigquery` | Datasets, tables, IAM, authorized views | `DATASTORE` | 12h | |
| `gke` | Clusters, node pools, workload identity bindings | K8s↔IAM bridge | 4h | ★ |
| `cloudfunctions` | Functions, SA bindings, triggers | `ASSET`, `RUNS_ON` | 4h | |
| `cloudrun` | Services, SA bindings, ingress | `APPLICATION`, `EXPOSES` | 4h | |
| `cloudbuild` | Triggers, **default SA privilege** | escalation ingredient | 12h | ★ |
| `secret_manager` | Secret metadata + IAM (never values) | `SECRET` → identity bridge | 12h | ★ |
| `kms` | Keyrings, keys, IAM | `CAN_READ` via decrypt | 12h | |
| `vpc` | Networks, subnets, firewall rules, peering | `NETWORK`, `ROUTES_TO` | 12h | ★ |
| `vertex_ai` | Models, endpoints, agents, datasets | `AI_MODEL`, `AI_AGENT` | 4h | ★ |
| `scc` | Security Command Center findings | `FINDING` overlay | 1h | |
| `audit_logs` | Admin activity + data access | `EVENT_SUMMARY`, USED state | 15m | |

**22 collectors.**

---

## 4 · Oracle Cloud (OCI)

```
  api_surface   configuration
  auth          instance principal → API signing key
  least priv    read-only policy at tenancy scope
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `compartments` | Compartment tree | scope hierarchy | 24h | ★ |
| `iam.users` | Users, API keys, auth tokens | `IDENTITY` | 4h | ★ |
| `iam.groups` | Groups, membership | `GROUP`, `MEMBER_OF` | 4h | |
| `iam.policies` | Policy statements | grants | 4h | ★ |
| `iam.dynamic_groups` | Dynamic group matching rules | `IDENTITY`, membership by rule | 12h | |
| `compute` | Instances, instance principals | `ASSET`, `RUNS_ON` | 4h | |
| `object_storage` | Buckets, pre-authenticated requests, visibility | `DATASTORE`, exposure | 12h | |
| `database` | DB systems, autonomous DBs | `DATASTORE` | 12h | |
| `vcn` | VCNs, subnets, security lists, NSGs | `NETWORK`, `ROUTES_TO` | 12h | |
| `vault` | Vaults, keys, secret metadata | `SECRET` | 12h | |
| `audit` | Audit events | `EVENT_SUMMARY` | 15m | |

**11 collectors.**

---

## 5 · Alibaba Cloud

```
  api_surface   configuration
  auth          RAM role → AccessKey
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `resource_directory` | Account structure, folders | scope tree | 24h | |
| `ram.users` | RAM users, AK pairs, MFA | `IDENTITY` | 4h | ★ |
| `ram.roles` | Roles + trust policies | `ROLE`, `CAN_ASSUME` | 4h | ★ |
| `ram.policies` | System + custom policies | grants | 4h | ★ |
| `ecs` | Instances, RAM role bindings | `ASSET`, `RUNS_ON` | 4h | |
| `oss` | Buckets, ACLs, policies | `DATASTORE`, exposure | 12h | |
| `rds` | Instances, whitelist | `DATASTORE` | 12h | |
| `vpc` | VPCs, vSwitches, security groups | `NETWORK`, `ROUTES_TO` | 12h | |
| `actiontrail` | Audit events | `EVENT_SUMMARY` | 15m | |

**9 collectors.**

---

## 6 · Kubernetes (generic — EKS/AKS/GKE/self-hosted)

```
  api_surface   configuration
  functions     collect only (response is out of scope)
  auth          service account token with a read-only ClusterRole
  least priv    a dedicated ClusterRole: get/list/watch on the
                resources below. NEVER cluster-admin.
  coverage      per cluster, per namespace for namespaced resources
  ⚠             dep: the cloud connector for the same cluster. The
                IRSA / Workload Identity / Managed Identity
                annotation is the bridge, and BOTH sides must be
                collected or the bridge does not exist.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `namespaces` | Namespaces, labels, quotas | scope | 12h | |
| `service_accounts` | SAs + **cloud identity annotations** | `IDENTITY`, the cloud bridge | 4h | ★ |
| `roles` | Roles + ClusterRoles | what each grants | 4h | ★ |
| `bindings` | RoleBindings + ClusterRoleBindings | grants | 4h | ★ |
| `workloads` | Deployments, DaemonSets, StatefulSets, Jobs, CronJobs | `WORKLOAD`, `RUNS_AS` | 4h | ★ |
| `pods` | Pods, SA mounts, security context, hostPath | `CONTAINER`, `RUNS_ON`, privilege findings | 4h | |
| `nodes` | Nodes, labels, **node instance role** | `ASSET`, node→cloud role bridge | 4h | ★ |
| `secrets` | Secret metadata and types (**never values**) | `SECRET` | 12h | ★ |
| `services_ingress` | Services, Ingress, LoadBalancers | `EXPOSES`, `NETWORK` | 4h | |
| `network_policies` | NetworkPolicies | `PROTECTS`, segmentation gaps | 12h | |
| `admission` | Mutating + validating webhook configs | escalation ingredient | 12h | ★ |
| `csr` | CertificateSigningRequests, approvers | escalation ingredient | 12h | |
| `audit_log` | Kubernetes audit events | `EVENT_SUMMARY`, USED state | 15m | |

**13 collectors.**

---

## 7 · OpenShift

```
  api_surface   configuration
  auth          service account token
  ⚠             everything in the Kubernetes connector applies.
                These are the OpenShift-specific additions.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Projects (namespace equivalents) | scope | 12h | |
| `scc` | SecurityContextConstraints + bindings | privilege escalation surface | 12h | ★ |
| `routes` | Routes | `EXPOSES` | 4h | |
| `imagestreams` | ImageStreams, registries | `ARTIFACT` | 12h | |
| `builds` | BuildConfigs, build SAs | `CAN_DEPLOY` | 12h | |
| `oauth` | OAuth clients, identity providers | federation trust | 12h | ★ |

**6 collectors** (plus the 13 shared with Kubernetes).

---

## 8 · VMware vSphere / vCenter

```
  api_surface   configuration
  auth          read-only vCenter role, dedicated service account
  least priv    "Read-only" role at the vCenter root
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `vms` | Virtual machines, guest info, tools state | `ASSET` | 4h | ★ |
| `hosts` | ESXi hosts, versions, patch state | `ASSET` | 12h | ★ |
| `clusters` | Clusters, DRS/HA config, resource pools | grouping | 12h | |
| `datastores` | Datastores, capacity, backing | `DATASTORE` | 12h | |
| `networks` | Port groups, vSwitches, VLANs, NSX segments | `NETWORK` | 12h | ★ |
| `permissions` | vCenter roles, permissions, SSO users | `IDENTITY`, `CAN_MODIFY` on infra | 12h | ★ |
| `snapshots` | Snapshot inventory, age | data-exposure findings | 24h | |
| `tags` | Tags and categories | business context, ABAC input | 12h | |

**8 collectors.**

---

## 9 · Microsoft Hyper-V / SCVMM

```
  api_surface   configuration
  auth          WinRM with a delegated read-only account
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `vms` | Virtual machines, state, integration services | `ASSET` | 4h | ★ |
| `hosts` | Hyper-V hosts, cluster membership | `ASSET` | 12h | ★ |
| `virtual_networks` | Virtual switches, VLANs | `NETWORK` | 12h | |
| `storage` | VHD/VHDX locations, shared storage | `DATASTORE` | 12h | |
| `permissions` | SCVMM user roles and scopes | `CAN_MODIFY` on infra | 12h | ★ |

**5 collectors.**

---

## 10 · Nutanix

```
  api_surface   configuration
  auth          Prism API, read-only role
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `vms` | VMs, categories | `ASSET` | 4h | ★ |
| `clusters` | Clusters, hosts, versions | `ASSET` | 12h | ★ |
| `storage_containers` | Containers, volume groups | `DATASTORE` | 12h | |
| `networks` | Subnets, virtual switches | `NETWORK` | 12h | |
| `projects_roles` | Projects, role mappings, directory config | `IDENTITY`, `CAN_MODIFY` | 12h | ★ |

**5 collectors.**

---

## Domain summary

| Connector | Collectors | api_surface |
|---|---|---|
| AWS | 42 | configuration + log_stream |
| Azure | 23 | configuration + log_stream |
| Google Cloud | 22 | configuration + log_stream |
| Oracle Cloud | 11 | configuration |
| Alibaba Cloud | 9 | configuration |
| Kubernetes | 13 | configuration |
| OpenShift | 6 | configuration |
| VMware vSphere | 8 | configuration |
| Hyper-V / SCVMM | 5 | configuration |
| Nutanix | 5 | configuration |
| **Total** | **144** | |

*(144 enumerated against doc 03's estimate of 122 — the difference is IAM and network being broken out properly rather than treated as single collectors.)*

### The dependency worth calling out

```
  eks / aks / gke  ──dep──►  kubernetes.service_accounts
  kubernetes.nodes ──dep──►  ec2.instances / compute.vms / compute.instances

  The K8s↔cloud identity bridge requires BOTH sides. A customer with
  Kubernetes but no cloud connector gets a cluster graph with no
  route to cloud resources — every workload-identity path is
  invisible, not shortened.
```

---

*Next: [Identity and access](02-identity-access.md)*

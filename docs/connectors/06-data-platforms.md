# Domain 06 — Data Platforms

**18 connectors · 99 collectors** · [Index](00-index.md)

Band 4, isolated resource pool. These connectors must never starve identity or cloud collection — a classification crawl over 40 TB of file shares is the single most common way a collector like this falls over (`../04 §26`).

⚠ **The distinction that matters in this domain:** the cloud connector sees the *resource* — that an RDS instance exists, is public, and is encrypted. This domain sees *inside* it — schemas, database-local users, grants, and what data is actually there. They are different connectors with different credentials answering different questions, and both are needed.

⚠ **Every database has its own IAM that nobody models.** A cloud role granting `rds:DescribeDBInstances` says nothing about who can `SELECT` from the payments table. Database-internal grants are a permission system as real as cloud IAM, and this domain is the only place they appear.

---

> **⚠ ALIGNED TO THE LOW LEVEL DESIGN.**
> `../LLD-edge-collector-v1.0.md` is the implementation boundary and takes
> precedence over this document. It supersedes
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` for collector internals.
> Content here that extends the LLD is a **PROPOSED EXTENSION**.
> Open escalations: `../edge-collector/13-escalations.md`.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · Oracle Database

```
  api_surface   configuration
  auth          dedicated read-only DB account
  least priv    SELECT on the DBA_* catalog views only. No data access.
  ⚠             classification requires sampling actual rows — a
                separate, explicitly-consented collector.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `instances` | Instances, versions, patch level, PDB structure | `DATASTORE` | 12h | ★ |
| `users_roles` | Database users, roles, profiles, lock state | `IDENTITY` (database-local), `ROLE` | 4h | ★ |
| `grants` | System and object privileges, role grants | `CAN_READ`/`CAN_WRITE` inside the DB | 4h | ★ |
| `schemas` | Schemas, tables, columns, sizes | `DATASTORE` structure | 12h | ★ |
| `classification` | Sampled column content and metadata | `DATA_CLASS` properties | 7d | |
| `security_config` | TDE, auditing, network ACLs, listener config | encryption and posture properties | 12h | |
| `audit_trail` | Unified audit records | `EVENT_SUMMARY`, USED state | 4h | |

**7 collectors.**

---

## 2 · Microsoft SQL Server

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `instances` | Instances, editions, patch level | `DATASTORE` | 12h | ★ |
| `logins_users` | Server logins, database users, AD mappings | `IDENTITY`, **AD linkage** | 4h | ★ |
| `roles_permissions` | Server and database roles, explicit permissions | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `schemas` | Databases, schemas, tables, columns | structure | 12h | ★ |
| `classification` | Built-in sensitivity labels + sampling | `DATA_CLASS` | 7d | |
| `security_config` | TDE, Always Encrypted, audit specs, endpoints | posture | 12h | |
| `linked_servers` | Linked server definitions and credentials | **cross-database `CAN_ASSUME`** | 12h | ★ |
| `audit` | SQL Server audit events | `EVENT_SUMMARY` | 4h | |

**8 collectors.** `linked_servers` is quietly dangerous — a linked server with stored credentials is a lateral-movement edge between database instances that no cloud connector can see.

---

## 3 · PostgreSQL

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `instances` | Version, extensions, replication config | `DATASTORE` | 12h | ★ |
| `roles` | Roles, membership, attributes (SUPERUSER, etc.) | `IDENTITY`, `MEMBER_OF` | 4h | ★ |
| `grants` | Schema, table, column and RLS privileges | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `schemas` | Databases, schemas, tables, columns | structure | 12h | ★ |
| `classification` | Sampled content | `DATA_CLASS` | 7d | |
| `security_config` | `pg_hba.conf` rules, SSL, encryption | `AUTHENTICATES_TO`, posture | 12h | ★ |
| `extensions` | Installed extensions (`dblink`, `postgres_fdw`) | cross-database reachability | 12h | |

**7 collectors.**

---

## 4 · MySQL / MariaDB

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `instances` | Version, replication topology | `DATASTORE` | 12h | ★ |
| `users` | Users, host patterns, auth plugins | `IDENTITY` | 4h | ★ |
| `grants` | Global, database, table, column grants | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `schemas` | Databases, tables, columns | structure | 12h | ★ |
| `classification` | Sampled content | `DATA_CLASS` | 7d | |
| `security_config` | SSL, `local_infile`, plugin config | posture | 12h | |

**6 collectors.**

---

## 5 · MongoDB

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `clusters` | Replica sets, shards, version | `DATASTORE` | 12h | ★ |
| `users_roles` | Users, built-in and custom roles | `IDENTITY`, `ROLE` | 4h | ★ |
| `privileges` | Role privileges per database/collection | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `databases` | Databases, collections, index metadata | structure | 12h | ★ |
| `classification` | Sampled document fields | `DATA_CLASS` | 7d | |
| `security_config` | Auth mechanism, TLS, network binding | posture, exposure findings | 12h | ★ |

**6 collectors.**

---

## 6 · Elasticsearch / OpenSearch

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `clusters` | Cluster health, nodes, version | `DATASTORE` | 12h | ★ |
| `indices` | Indices, mappings, sizes, aliases | structure | 12h | ★ |
| `users_roles` | Users, roles, role mappings | `IDENTITY`, `ROLE` | 4h | ★ |
| `privileges` | Index and cluster privileges, field/document security | `CAN_READ` | 4h | ★ |
| `classification` | Sampled document fields | `DATA_CLASS` | 7d | |
| `security_config` | TLS, anonymous access, network binding | exposure findings | 12h | ★ |

**6 collectors.**

---

## 7 · Cassandra / ScyllaDB

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `clusters` | Cluster topology, datacenters, version | `DATASTORE` | 12h | ★ |
| `keyspaces` | Keyspaces, tables, replication | structure | 12h | ★ |
| `roles` | Roles and their grants | `IDENTITY`, `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `security_config` | Authenticator, authorizer, TLS | posture | 12h | |

**4 collectors.**

---

## 8 · Snowflake

```
  ⚠ Snowflake's role hierarchy is deep, inherited and frequently
    misunderstood. Its own permission closure is a genuine
    sub-problem, not a flat grant list.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `account_config` | Account params, network policies, SSO/SCIM config | posture, federation trust | 12h | ★ |
| `users` | Users, default roles, MFA, key-pair auth | `IDENTITY` | 4h | ★ |
| `roles` | Roles and **role hierarchy** | `ROLE`, `MEMBER_OF` transitively | 4h | ★ |
| `grants` | Grants to roles on all object types | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `databases` | Databases, schemas, tables, views | structure | 12h | ★ |
| `shares` | Inbound and outbound data shares | **cross-account data egress** | 12h | ★ |
| `masking_policies` | Masking and row-access policies | `PROTECTS` on columns | 12h | |
| `classification` | Snowflake classification + sampling | `DATA_CLASS` | 7d | |
| `access_history` | Query and access history | `EVENT_SUMMARY`, USED state | 4h | |

**9 collectors.** `shares` matters: an outbound share is data leaving the account entirely, invisible to every network control.

---

## 9 · Databricks

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `workspaces` | Workspaces, config, network access | scope, exposure | 12h | ★ |
| `unity_catalog` | Catalogs, schemas, tables, lineage | `DATASTORE` structure | 12h | ★ |
| `grants` | Unity Catalog privileges | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `users_groups` | Users, groups, service principals, SCIM | `IDENTITY`, `MEMBER_OF` | 4h | ★ |
| `clusters` | Clusters, instance profiles, **cloud role bindings** | `RUNS_AS` → cloud bridge | 4h | ★ |
| `tokens` | Personal access token metadata, expiry | `SECRET`, credential hygiene | 12h | ★ |
| `jobs` | Jobs, their run-as identity, schedules | `PIPELINE`, `RUNS_AS` | 4h | |
| `secrets` | Secret scope names and ACLs (never values) | `SECRET` | 12h | |
| `model_serving` | Served models and endpoints | `AI_MODEL`, `EXPOSES` | 4h | ★ |

**9 collectors.**

---

## 10 · BigQuery *(deep access)*

```
  ⚠ the GCP connector's `bigquery` collector reads the CONTROL PLANE
    — datasets, IAM, exposure. THIS connector reads INSIDE: table
    schemas, column-level policy tags, authorized views, and
    classification. Use both.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `datasets_tables` | Dataset, table and column structure | structure | 12h | ★ |
| `authorized_views` | Authorized views and datasets | **indirect `CAN_READ` paths** | 12h | ★ |
| `policy_tags` | Column-level policy tags and taxonomies | `DATA_CLASS`, `PROTECTS` | 12h | ★ |
| `row_access` | Row-level access policies | `PROTECTS` | 12h | |
| `classification` | DLP scan results or sampling | `DATA_CLASS` | 7d | |
| `job_history` | Query history, who read what | `EVENT_SUMMARY`, USED state | 4h | |

**6 collectors.**

---

## 11 · Amazon Redshift *(deep access)*

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `schemas` | Databases, schemas, tables, columns | structure | 12h | ★ |
| `users_groups` | Database users, groups, IAM-federated roles | `IDENTITY`, IAM linkage | 4h | ★ |
| `grants` | Object and column privileges | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `datashares` | Datashares to other clusters/accounts | cross-account data egress | 12h | ★ |
| `classification` | Sampled content | `DATA_CLASS` | 7d | |
| `query_history` | Query and connection history | `EVENT_SUMMARY` | 4h | |

**6 collectors.**

---

## 12 · Azure Synapse *(deep access)*

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `workspaces` | Workspace config, managed VNet, firewall | exposure | 12h | ★ |
| `pools_schemas` | SQL/Spark pools, databases, tables | structure | 12h | ★ |
| `permissions` | Workspace RBAC + SQL-level grants | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `linked_services` | Linked services and their credentials | **cross-service `CAN_ASSUME`** | 12h | ★ |
| `pipelines` | Data pipelines, triggers, run-as identity | `PIPELINE`, `RUNS_AS` | 4h | |
| `classification` | Sensitivity labels + sampling | `DATA_CLASS` | 7d | |

**6 collectors.**

---

## 13 · SMB / NFS file shares

```
  api_surface   configuration
  auth          read-only service account with traverse rights
  ⚠             the most expensive collector in the entire catalog.
                40 TB of shares cannot be enumerated in a cycle —
                this is a ROLLING, PARTITIONED crawl with coverage
                reported as a percentage and an age distribution.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `shares` | Share list, share-level permissions | `DATASTORE` | 12h | ★ |
| `acls` | NTFS/POSIX ACLs on directories | `CAN_READ`/`CAN_WRITE`, **AD linkage** | 24h | ★ |
| `open_shares` | Everyone/Authenticated Users exposure | exposure findings | 12h | ★ |
| `file_metadata` | File names, sizes, ages, owners (never content) | structure, stale-data findings | 7d | |
| `classification` | Sampled file content | `DATA_CLASS` | 7d rolling | |

**5 collectors.** The `acls` collector is where file shares connect to the identity graph — a share ACL granting `Domain Users` read on `\\fs01\finance` is a `CAN_READ` edge from 12,000 people to payroll data.

---

## 14 · SharePoint / OneDrive

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `sites` | Site collections, sites, ownership | `DATASTORE` | 12h | ★ |
| `libraries` | Document libraries, lists | structure | 12h | ★ |
| `permissions` | Site, library and item permissions, groups | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `sharing_links` | Anonymous, org-wide and external sharing links | **exposure findings** | 4h | ★ |
| `external_users` | Guest users with access | external `CAN_READ` | 4h | ★ |
| `sensitivity_labels` | Purview labels applied | `DATA_CLASS` | 12h | |
| `classification` | Content sampling where labels are absent | `DATA_CLASS` | 7d | |

**7 collectors.** `sharing_links` is the highest-value collector here — an anonymous link to a document library is data exposed to the internet with no network path involved.

---

## 15 · Box

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `folders` | Folder tree, ownership | `DATASTORE` | 12h | ★ |
| `collaborations` | Collaborations, roles, external collaborators | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `shared_links` | Shared link scope and access | exposure findings | 4h | ★ |
| `users_groups` | Users, groups, external users | `IDENTITY` | 4h | |
| `classification` | Box Shield classification + sampling | `DATA_CLASS` | 7d | |
| `events` | Access and admin events | `EVENT_SUMMARY` | 4h | |

**6 collectors.**

---

## 16 · Dropbox Business

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `teams_members` | Team members, roles, external members | `IDENTITY` | 4h | ★ |
| `shared_folders` | Shared folders and membership | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `shared_links` | Shared link visibility and expiry | exposure findings | 4h | ★ |
| `classification` | Content sampling | `DATA_CLASS` | 7d | |
| `events` | Team and file events | `EVENT_SUMMARY` | 4h | |

**5 collectors.**

---

## 17 · Google Drive

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `shared_drives` | Shared drives, restrictions, org units | `DATASTORE` | 12h | ★ |
| `permissions` | File and folder permissions, roles | `CAN_READ`/`CAN_WRITE` | 4h | ★ |
| `sharing_settings` | Link sharing, external and domain-wide sharing | exposure findings | 4h | ★ |
| `labels` | Drive labels and classification | `DATA_CLASS` | 12h | |
| `classification` | DLP results or sampling | `DATA_CLASS` | 7d | |
| `activity` | Drive activity events | `EVENT_SUMMARY` | 4h | |

**6 collectors.**

---

## 18 · NetApp / Dell Isilon / enterprise NAS

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `volumes_shares` | Volumes, qtrees, exports, shares | `DATASTORE` | 12h | ★ |
| `export_policies` | NFS export rules, SMB share ACLs | `CAN_READ`, exposure | 12h | ★ |
| `quotas_usage` | Capacity, usage, growth | context for classification scope | 24h | |
| `snapshots` | Snapshot policies and retention | data-copy exposure | 24h | |
| `admins` | Storage admin accounts and roles | `IDENTITY`, `CAN_MODIFY` | 24h | ★ |

**5 collectors.**

---

## Domain summary

| Connector | Collectors |
|---|---|
| Oracle Database | 7 |
| Microsoft SQL Server | 8 |
| PostgreSQL | 7 |
| MySQL / MariaDB | 6 |
| MongoDB | 6 |
| Elasticsearch / OpenSearch | 6 |
| Cassandra / ScyllaDB | 4 |
| Snowflake | 9 |
| Databricks | 9 |
| BigQuery (deep) | 6 |
| Redshift (deep) | 6 |
| Azure Synapse (deep) | 6 |
| SMB / NFS shares | 5 |
| SharePoint / OneDrive | 7 |
| Box | 6 |
| Dropbox Business | 5 |
| Google Drive | 6 |
| NetApp / Isilon / NAS | 5 |
| **Total** | **114** |

### Three patterns that repeat across every connector here

```
  1  DATABASE-INTERNAL IAM
     Every platform has its own users, roles and grants, and none of
     it appears in cloud IAM. A principal with rds:Describe* has no
     SELECT rights; a database role with SELECT on payments has no
     cloud identity. The two permission systems meet only at
     authentication, and that join is what makes a complete path.

  2  SHARING AND FEDERATION AS EXPOSURE
     shared_links · sharing_settings · shares · datashares ·
     linked_servers · authorized_views
     → data leaving the boundary with no network path involved.
       No firewall rule records an anonymous SharePoint link.

  3  CLASSIFICATION IS ALWAYS THE MOST EXPENSIVE COLLECTOR
     and almost always deferrable. Where a customer runs DLP,
     Purview, Macie or Snowflake classification, ingest their
     labels as an overlay instead. Building our own crawl
     duplicates a control they already paid for
     (../engines/08-posture-correlation-classification.md §8.4).
```

### Why crown jewels depend entirely on this domain

Criticality is what makes a path worth showing. A path terminating at "a database" ranks nowhere; the same path terminating at "4.2M records, PII + PCI" is the top finding. That label comes from here — either from our classification collectors or from an ingested DLP overlay.

**Without this domain, the graph has no crown jewels, and without crown jewels the path engine has nothing to compute toward.**

---

*Next: [AI platforms](07-ai-platforms.md)*

# Domain 07 — AI Platforms

**15 connectors · 71 collectors** · [Index](00-index.md)

Band 4. This domain plus [the Agent](10-agent.md) produces the entities behind the product's most differentiated finding — the **AI Privilege Gap** — and the layer no competitor covers: unmanaged, unregistered AI.

⚠ **Cloud-native AI services are collectors inside their cloud connector**, not connectors here. Bedrock lives in AWS, Azure OpenAI in Azure, Vertex AI in GCP. They share the cloud credential and rate-limit domain, so making them separate connectors would break the counting unit (`../00-index.md`).

⚠ **The distinction that defines this domain:** every competitor starts from a **registry** — Entra Agent ID, Copilot Studio, an enterprise agent platform, a public MCP directory. This domain covers registries too, but the differentiated half is the unregistered layer, and that comes from the Agent.

---

## 1 · OpenAI (organisation / admin API)

```
  api_surface   configuration
  auth          admin API key, read-only where scopes permit
  ⚠             this is the ORG ADMIN API — projects, keys, members,
                usage. It is not prompt inspection. Prompt content
                comes from the AI Gateway, which is a different
                component entirely.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organization` | Org settings, verified domains, SSO config | scope, federation trust | 12h | ★ |
| `projects` | Projects, their service accounts, rate limits | `AI_APPLICATION` scope | 4h | ★ |
| `members` | Org and project members, roles | `IDENTITY`, `CAN_ACCESS` | 4h | ★ |
| `api_keys` | Key metadata, owner, scope, last used (never values) | `SECRET` → identity bridge | 4h | ★ |
| `models` | Available and fine-tuned models | `AI_MODEL` | 12h | ★ |
| `usage` | Token and request usage by project and key | `EVENT_SUMMARY`, dormancy | 4h | |
| `assistants` | Assistants, their tools and file attachments | `AI_AGENT`, `INVOKES` | 4h | ★ |
| `vector_stores` | Vector stores and attached files | `VECTOR_DATABASE`, `RETRIEVES_FROM` | 4h | ★ |

**8 collectors.**

---

## 2 · Anthropic (organisation admin API)

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organization` | Org settings, domain verification | scope | 12h | ★ |
| `workspaces` | Workspaces, their limits and membership | `AI_APPLICATION` scope | 4h | ★ |
| `members` | Users, roles, workspace assignments | `IDENTITY`, `CAN_ACCESS` | 4h | ★ |
| `api_keys` | Key metadata, owner, workspace, last used | `SECRET` → identity bridge | 4h | ★ |
| `usage` | Token usage by workspace and key | `EVENT_SUMMARY`, dormancy | 4h | |

**5 collectors.**

---

## 3 · Google AI / Gemini (non-Vertex)

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | AI Studio projects and settings | scope | 12h | ★ |
| `api_keys` | Key metadata, restrictions, last used | `SECRET` → identity bridge | 4h | ★ |
| `models` | Available and tuned models | `AI_MODEL` | 12h | |
| `usage` | Request and token usage | `EVENT_SUMMARY` | 4h | |

**4 collectors.**

---

## 4 · Hugging Face

```
  ⚠ a genuine supply-chain surface. Models and datasets pulled from
    a public hub into production inference are third-party code
    executing with an application's identity.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organizations` | Orgs, members, roles | `IDENTITY`, `MEMBER_OF` | 12h | ★ |
| `models` | Owned and private models, visibility | `AI_MODEL` | 12h | ★ |
| `datasets` | Datasets and visibility | `DATASTORE`, exposure findings | 12h | ★ |
| `spaces` | Spaces, their hardware and secrets | `APPLICATION`, `SECRET` | 12h | |
| `tokens` | Access token metadata, scope, expiry | `SECRET` | 12h | ★ |
| `external_models` | Public models referenced by internal apps | supply-chain provenance | 24h | ★ |

**6 collectors.**

---

## 5 · GitHub Copilot (admin)

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `seats` | Assigned seats, last activity | `IDENTITY` `USES` `AI_APPLICATION` | 4h | ★ |
| `policies` | Org policies, public code filter, IDE restrictions | `PROTECTS`, policy gaps | 12h | ★ |
| `usage` | Suggestion and acceptance metrics | `EVENT_SUMMARY`, adoption | 24h | |
| `extensions` | Copilot Extensions and their permissions | `AI_AGENT`, `INVOKES` | 12h | ★ |

**4 collectors.**

---

## 6 · Pinecone

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `projects` | Projects and environments | scope | 12h | ★ |
| `indexes` | Indexes, dimensions, metrics, size | `VECTOR_DATABASE` | 4h | ★ |
| `namespaces` | Namespaces and record counts | **multi-tenancy boundary** | 4h | ★ |
| `api_keys` | Key metadata and scope | `SECRET` → identity bridge | 4h | ★ |
| `access_config` | Network access, private endpoints, RBAC | exposure findings | 12h | ★ |

**5 collectors.**

---

## 7 · Weaviate

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `schema` | Classes, properties, vectorizer modules | `VECTOR_DATABASE` structure | 12h | ★ |
| `tenants` | Multi-tenancy configuration | tenancy boundary | 4h | ★ |
| `users_roles` | RBAC users, roles, permissions | `IDENTITY`, `CAN_READ` | 4h | ★ |
| `api_keys` | Key metadata | `SECRET` | 12h | ★ |
| `security_config` | Auth mode, anonymous access, TLS | **exposure findings** | 12h | ★ |

**5 collectors.**

---

## 8 · Qdrant

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `collections` | Collections, vector config, payload schema | `VECTOR_DATABASE` | 4h | ★ |
| `api_keys` | Key metadata, read-only flags | `SECRET` | 12h | ★ |
| `security_config` | Auth enablement, TLS, network binding | exposure findings | 12h | ★ |
| `cluster` | Cluster topology and shards | scope | 12h | |

**4 collectors.**

---

## 9 · Milvus / Zilliz

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `collections` | Collections, schemas, partitions | `VECTOR_DATABASE` | 4h | ★ |
| `users_roles` | Users, roles, privilege grants | `IDENTITY`, `CAN_READ` | 4h | ★ |
| `security_config` | Auth, TLS, network access | exposure findings | 12h | ★ |
| `clusters` | Cluster topology (Zilliz Cloud) | scope | 12h | |

**4 collectors.**

---

## 10 · Chroma

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `collections` | Collections and metadata | `VECTOR_DATABASE` | 4h | ★ |
| `security_config` | Auth provider, token config, network binding | **exposure findings** | 12h | ★ |
| `tenants_databases` | Tenant and database separation | tenancy boundary | 12h | |

**3 collectors.** Chroma is frequently run unauthenticated in development and then quietly promoted, which makes `security_config` the load-bearing collector.

---

## 11 · LangSmith / LangChain platform

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organizations` | Orgs, workspaces, members | `IDENTITY`, scope | 12h | ★ |
| `projects` | Tracing projects mapped to applications | `AI_APPLICATION` | 4h | ★ |
| `agents_chains` | Deployed chains and agents, their tools | `AI_AGENT`, `INVOKES` | 4h | ★ |
| `datasets` | Evaluation datasets and their contents metadata | `DATASTORE` | 12h | |
| `api_keys` | Key metadata and scope | `SECRET` | 12h | ★ |
| `deployments` | Hosted deployments and endpoints | `EXPOSES` | 4h | |

**6 collectors.**

---

## 12 · MCP registry and server discovery

```
  ⚠ this connector covers REGISTERED MCP — enterprise registries,
    public directories, and servers declared in repos.
    UNREGISTERED MCP on laptops comes from the Agent, and that is
    the layer nobody else covers.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `registries` | Configured enterprise and public registries | scope | 12h | ★ |
| `servers` | Registered MCP servers, transport, endpoint | `MCP_SERVER` | 4h | ★ |
| `tools` | Tools exposed per server, their schemas | `MCP_TOOL`, `INVOKES` | 4h | ★ |
| `tool_descriptions` | Full tool descriptions and annotations | **tool-poisoning detection** | 4h | ★ |
| `credentials` | Credential requirements per server (types only) | `SECRET` presence | 4h | ★ |
| `repo_declarations` | `mcp.json` / `.mcp/` committed in repos | `MCP_SERVER` from code | 4h | ★ |
| `reputation` | Package provenance, publisher, download counts | supply-chain risk properties | 24h | |

**7 collectors.** `tool_descriptions` is subtle and important: a malicious instruction embedded in a tool's *description* is read by the model as trusted context — a tool-poisoning attack that never touches the tool's code.

---

## 13 · Glean

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `datasources` | Connected sources and their sync scope | `RETRIEVES_FROM` | 12h | ★ |
| `permissions` | Permission model and identity mapping per source | **retrieval-vs-query ACL gap** | 4h | ★ |
| `users` | Users and their access scope | `IDENTITY`, `CAN_ACCESS` | 4h | ★ |
| `agents` | Configured agents and actions | `AI_AGENT`, `INVOKES` | 4h | ★ |
| `activity` | Search and retrieval activity | `EVENT_SUMMARY`, excessive retrieval | 4h | |

**5 collectors.** This is the canonical **RAG permission gap** connector: the retrieval identity's access versus the querying user's access is exactly the `IAM-084` finding.

---

## 14 · Enterprise AI applications (Writer, Jasper, and similar)

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `organization` | Org config, SSO, data-retention settings | scope, posture | 12h | ★ |
| `users_teams` | Users, teams, roles | `IDENTITY`, `MEMBER_OF` | 4h | ★ |
| `connectors` | Connected data sources | `RETRIEVES_FROM` | 4h | ★ |
| `api_keys` | Key metadata | `SECRET` | 12h | |
| `usage` | Activity and volume | `EVENT_SUMMARY` | 24h | |

**5 collectors.**

---

## 15 · Local model runtimes (Ollama, vLLM, LM Studio — server-side)

```
  ⚠ the AGENT covers laptop runtimes. THIS connector covers
    SERVER-side self-hosted inference: a vLLM endpoint on a GPU
    node, an Ollama instance someone stood up on a shared server.
    Frequently unauthenticated and reachable on the internal network.
```

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `endpoints` | Discovered inference endpoints, ports, TLS | `AI_MODEL` endpoint, `EXPOSES` | 4h | ★ |
| `models` | Loaded models, sizes, provenance | `AI_MODEL` | 4h | ★ |
| `auth_config` | Whether authentication is enabled at all | **exposure findings** | 4h | ★ |
| `usage` | Request volume where exposed | `EVENT_SUMMARY` | 4h | |

**4 collectors.**

---

## Domain summary

| Connector | Collectors |
|---|---|
| OpenAI | 8 |
| Anthropic | 5 |
| Google AI / Gemini | 4 |
| Hugging Face | 6 |
| GitHub Copilot | 4 |
| Pinecone | 5 |
| Weaviate | 5 |
| Qdrant | 4 |
| Milvus / Zilliz | 4 |
| Chroma | 3 |
| LangSmith | 6 |
| MCP registry | 7 |
| Glean | 5 |
| Enterprise AI apps | 5 |
| Local runtimes (server) | 4 |
| **Total** | **75** |

### What this domain contributes to the graph

```
  AI_APPLICATION    the tools in use, sanctioned or not
  AI_MODEL          what is being called, and where it is hosted
  AI_AGENT          agents, and critically their RUNS_AS identity
  MCP_SERVER        registered servers and their tools
  MCP_TOOL          what each agent can actually invoke
  VECTOR_DATABASE   retrieval targets and their access model
  SECRET            AI credentials, and therefore identity edges
```

### The two collector classes that carry the differentiation

**1 · `api_keys` everywhere.** An AI API key is an identity edge, not a secrets-hygiene item. Whoever can read it becomes whatever the key authenticates as, and AI keys are handled far more casually than cloud credentials — pasted into notebooks, committed to repos, shared in chat.

**2 · `security_config` on every vector database.** Weaviate, Qdrant, Milvus and Chroma are all commonly deployed with authentication disabled, on the assumption they sit behind something. An unauthenticated vector store holding embeddings of internal documents is a data breach with no data store involved, and this is the only place it becomes visible.

### And the gap this domain cannot close

Everything here is a **registry, an org API, or a server endpoint**. All of it is, by definition, AI somebody registered, provisioned or deployed.

The layer that matters most — an MCP filesystem server rooted at a developer's work directory, configured by hand, holding a GitHub token, on a laptop nobody surveyed — appears in no registry and no admin API. That comes from [the Agent](10-agent.md), and it is the one thing in this whole catalog that no competitor currently collects.

---

*Next: [Business context](08-business-context.md)*

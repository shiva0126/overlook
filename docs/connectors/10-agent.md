# Domain 10 — The Overlook Agent

**1 connector · 9 collectors** · [Index](00-index.md)

Continuous, its own ingress class. The agent is not a connector in the usual sense — it is our own software reporting in — but it has collectors, cadences and coverage semantics like everything else, so it belongs in this catalog.

**This is the differentiated one.** Every AI-security competitor surveyed starts from a registry: Entra Agent ID, Copilot Studio, an enterprise agent platform, a public MCP directory (`../../07-competitive-landscape.md §3`). The agent covers the layer that has no registry, and nobody else collects it.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## The scope rule that defines the agent

```
  COLLECT ONLY WHAT NO API CAN PROVIDE.

  This single rule is why the agent is ~102 MB/day across 8,500
  endpoints instead of terabytes.
```

**Deliberately NOT collected:**

| Not collected | Where it comes from instead | Why |
|---|---|---|
| Full process trees | CrowdStrike / Defender / FortiEDR API | 8,500 hosts × continuous capture ≈ **2.4 billion records/day** |
| Network connections | EDR API | same, plus the EDR's kernel driver is already approved |
| File events | EDR API | same |
| Logon events | EDR API + directory | the directory is authoritative |
| Host inventory | EDR + UEM | both already have it, with better coverage |
| Vulnerability data | Scanner API | not our job |

The agent is **thin, userland, read-mostly, no kernel driver**. That is what makes it shippable and what stops it being blamed for a blue screen (`../../01-system-design.md §12.1`).

---

## Connector header

```
  api_surface   agent
  functions     collect · respond (SEPARATE PACKAGE, not installed
                by default)
  transport     agent-initiated mTLS, outbound only, NO listening port
  auth          enrollment token → client certificate
  buffering     local, bounded: 24h or 200 MB, whichever first
  ack           collector journals + fsync, then acks; agent prunes
  coverage      per-host, per-collector; a host that did not report
                is STALE, never retracted
  resource caps CPU < 1% avg / < 5% peak · RAM < 150 MB
                disk IO < 5 MB/s during scans · net < 50 MB/day
                self-enforced by a watchdog that yields under load
```

---

## Collectors

| Collector | Pulls | Produces | Cadence | ★ |
|---|---|---|---|---|
| `host_context` | Hostname, OS, logged-on user, domain membership | heartbeat + the **resolution anchor** for everything else | 60s | ★ |
| `mcp_configs` | `~/.claude/`, `~/.cursor/`, `claude_desktop_config.json`, Windsurf, Zed and equivalents | `MCP_SERVER`, `MCP_TOOL`, `INVOKES`, credential **presence** | 4h | ★ |
| `local_models` | ollama, LM Studio, vLLM, llama.cpp processes and loaded models | `AI_MODEL`, `RUNS_ON` | 4h | ★ |
| `ide_assistants` | IDE AI extensions and the endpoints they call | `AI_APPLICATION`, `USES` | 4h | ★ |
| `ai_sdks` | AI SDK and agent-framework presence in local environments | `AI_APPLICATION`, development context | 12h | |
| `ai_credentials` | Credential **presence** in AI configs — type and location only | `SECRET` → identity bridge | 4h | ★ |
| `browser_ai_domains` | Browser process → AI domain relationships with owning user | `USES`, shadow AI | 1h | ★ |
| `agent_frameworks` | Self-hosted agent frameworks and their configured tools | `AI_AGENT`, `INVOKES` | 4h | ★ |
| `local_rag` | Local vector stores and indexed source directories | `VECTOR_DATABASE`, `RETRIEVES_FROM` | 12h | |

**9 collectors.**

---

## What each collector actually finds

### `host_context` — the anchor

Every other collector's output is meaningless without knowing *whose machine this is*. It reports the local username and domain membership; **it does not assert identity**. Resolution happens at E6, using the Resolution Directory or the UEM enrollment binding.

This is also the heartbeat, so its absence is how a silent agent is detected.

### `mcp_configs` — the flagship

```
  READS     server name, transport, command, arguments
            → in particular the ROOT PATH of filesystem servers
            environment variable NAMES
            → never values, never read, never transmitted

  PRODUCES  MCP_SERVER on this host
            MCP_TOOL for each declared tool
            AI_AGENT INVOKES MCP_SERVER
            SECRET presence, typed (github_token, aws_key, db_string)

  FINDS     mcp-filesystem rooted at a work or home directory
            servers holding long-lived credentials
            servers from unvetted packages
            the same server configured on N machines
```

### `ai_credentials` — presence, never value

```
  DETECTS   the NAME and TYPE of a credential in an AI config,
            and its location

  NEVER     reads, stores, transmits or hashes the value

  WHY IT MATTERS
    A GitHub token in an MCP config is an IDENTITY EDGE. Whoever
    can read that file inherits whatever the token authenticates
    as. Detecting its presence and type is sufficient to build the
    edge; reading its value would make us the risk we are reporting.
```

### `browser_ai_domains` — shadow AI without interception

Establishes *user → browser process → AI service* without any TLS interception on our part. Combined with proxy or DNS data from domain 05, it gives the volume; combined with the directory, it gives the person.

### `agent_frameworks` and `local_rag` — the self-hosted layer

A team that wrote their own agent with LangChain, or stood up a local Chroma index over the finance share, appears in **no registry and no admin API**. These two collectors are the only source.

---

## Platform notes

```
  WINDOWS   service running as LocalSystem
            EV code-signing certificate
            AV/EDR exclusion documentation shipped with the product

  macOS     notarized, and requires a PPPC profile deployed via MDM.
            Reading ~/Library and ~/.claude triggers TCC consent
            prompts unless pre-approved. SHIP THE PROFILE — this is
            the most underestimated deployment obstacle for the agent.

  LINUX     systemd unit, static or musl build to avoid glibc
            version issues across distributions

  CONTAINERS  do NOT deploy per container. A DaemonSet reading the
            node, plus the Kubernetes API for workload identity
            mapping, covers it.
```

---

## Coverage semantics

```
  An agent that did not report is NOT evidence that its findings
  are gone.

    reported this cycle        → coverage window for that host
    did not report             → host marked STALE with last-seen
    never enrolled             → not in scope; a COVERAGE GAP finding

  A laptop that was closed for a week must not have its MCP server
  tombstoned. The rule from E12 applies identically here: absence of
  observation is not observation of absence.
```

---

## The response executor

```
  SEPARATE PACKAGE. NOT INSTALLED BY DEFAULT.

  A customer can deploy the entire agent fleet with provably zero
  execution capability, and that must be verifiable in the
  Controller.

  When installed, every command requires:
    signature from the collector, validated against a pinned key
    unused nonce
    unexpired TTL
    pre-flight verification that the target still matches
    local policy permitting the action class
    the target not being on the protected-asset list

  And the agent's OWN timer releases any containment on TTL expiry,
  even if the collector is unreachable — a quarantine that survives
  a management-plane outage is an outage of the customer's business.
```

---

## What the agent contributes that nothing else can

```
  Registry-based competitors see:
    Entra Agent ID · Copilot Studio agents · enterprise platforms ·
    public MCP directories

  The agent sees:
    ✓ an MCP filesystem server rooted at ~/work on a developer laptop
    ✓ a GitHub token sitting in that server's env config
    ✓ ollama running a local model on a workstation
    ✓ an IDE assistant reaching an endpoint nobody approved
    ✓ a hand-written LangChain agent with three configured tools
    ✓ a local Chroma index built over a finance directory

  None of these appears in any registry, any admin API, or any
  cloud console. They are configured by hand, by individuals, and
  they hold real credentials.
```

And the finding that only exists because of this collector set:

```
  Priya's laptop runs mcp-filesystem rooted at ~/work
       ↑ agent · mcp_configs

  its config holds a GitHub token
       ↑ agent · ai_credentials (presence only)

  the laptop is enrolled to Priya
       ↑ UEM · devices (authoritative binding)

  Priya can write to meridian/payments-api
       ↑ GitHub · collaborators

  that repo has an OIDC trust into an AWS role
       ↑ GitHub · oidc_trusts + AWS · iam.roles

  that role reaches prod-payments-db, 4.2M records, PII + PCI
       ↑ AWS · iam.policies + DLP · classification

  SIX SOURCES. Remove the agent and the path has no beginning.
```

---

## Domain summary

| Connector | Collect | Respond |
|---|---|---|
| Overlook Agent | 9 | 1 (separate package) |

---

*End of the connector catalog. Back to the [index](00-index.md).*

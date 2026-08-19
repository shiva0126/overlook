# Overlook – Edge, Agent, AI Gateway and SaaS Architecture

## 1. Purpose

This document defines the proposed deployment and visibility architecture for **Overlook**, covering:

- Overlook Edge Analytics Node
- Overlook Agent
- Overlook AI Gateway
- Overlook SaaS / TrustGraph
- On-premises and cloud deployment
- CSPM, DSPM, ASPM, Identity, Network, Runtime, and AI Security
- Data privacy and local processing
- Host response actions
- AI visibility including Shadow AI, prompts, agents, MCP, and RAG
- Modular deployment levels

The core design principle is:

> **Sensitive customer telemetry is processed inside the customer's environment. Overlook SaaS receives only the minimum security facts, graph relationships, risk metadata, and evidence references required for correlation and attack-path analysis.**

---

> **⚠ SUPERSEDED — HISTORICAL RECORD ONLY.**
> This is the original architecture brief. It is retained because the
> escalations in `edge-collector/13-escalations.md` reference the
> reasoning in it. It is **not** a source of truth for anything.
> Current authority: `LLD-edge-collector-v1.0.md`, then
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1`.
> Do not build from this document.

---

# 2. High-Level Architecture

```text
                           OVERLOOK SaaS
                               |
                         Unified TrustGraph
                               |
              +----------------+----------------+
              |                                 |
        Attack Path Engine                 Risk Engine
              |                                 |
              +---------------+-----------------+
                              |
                       Security Facts
                              ^
                              |
               +--------------+--------------+
               |                             |
      OVERLOOK EDGE NODE              OVERLOOK AI GATEWAY
               |                             |
      APIs / Logs / Flows              Prompts / Responses
      Cloud / Identity                 AI Agents / Tools
      ASPM / DSPM                      MCP / RAG
      Network / Security               AI API Activity
               |
               |
         OVERLOOK AGENTS
               |
       Host Runtime Visibility
       Shadow AI Detection
       Process / Network Context
       Host Response Actions
```

---

# 3. Core Components

## 3.1 Overlook Edge Analytics Node

The **Overlook Edge Analytics Node** is the primary customer-side processing component.

It should be deployable in:

- On-premises data centers
- VMware / Hyper-V
- AWS
- Azure
- GCP
- Private cloud environments

### Main responsibilities

- Connector management
- REST API polling
- Syslog ingestion
- CEF / LEEF / JSON parsing
- NetFlow / IPFIX ingestion
- Webhook ingestion
- Cloud API integration
- Agent communication
- Automatic parsing
- Normalization
- Entity resolution
- Local analytics
- Local correlation
- Security Fact generation
- Local analytics database
- Flow control
- Buffering
- Compression
- Encryption
- Privacy filtering
- Credential vault
- Connector health
- Parser health
- Local management UI
- Response orchestration
- Offline buffering
- Secure synchronization with Overlook SaaS

### Processing flow

```text
Source
  |
  v
Ingestion
  |
  v
Auto Parser
  |
  v
Normalization
  |
  v
Entity Resolution
  |
  v
Local Analytics
  |
  v
Security Fact Generation
  |
  +----> Local Analytics Dataset
  |          |
  |       Compress
  |          |
  |       Encrypt
  |          |
  |       Local Storage
  |
  v
Privacy Gate
  |
  v
Minimum Security Facts
  |
  v
Overlook SaaS
```

---

# 4. Overlook Agent

The **Overlook Agent** is not intended to replace a full EDR platform.

Its purpose is to provide:

1. Host/runtime context
2. Improved Shadow AI visibility
3. Local host response actions
4. Selected telemetry that cannot be obtained through APIs alone

## 4.1 Runtime Visibility

The agent may collect:

- Running processes
- Parent-child process relationships
- Process metadata
- Command metadata
- User/session context
- Listening ports
- Active network connections
- DNS requests
- Local users
- Local groups
- Services
- Selected local configuration
- AI-related processes
- Local AI model processes
- MCP client/server configuration
- Browser/process-to-AI-domain relationships

Example:

```text
User
 |
 v
Chrome
 |
 v
chatgpt.com
```

or:

```text
Developer
 |
 v
VS Code
 |
 v
AI Coding Assistant
```

---

# 5. Overlook Agent Response Actions

Host-level response actions should be executed through the Overlook Agent.

Recommended initial actions:

## 5.1 Quarantine Host

Isolate the affected machine while retaining communication with Overlook.

```text
Server
 |
 +--X Internet
 +--X Internal Network
 +--X Production Systems
 |
 +---- Overlook Edge
```

## 5.2 Terminate Process

Kill a suspicious process identified by Overlook.

Example:

```text
WINWORD.EXE
   |
   v
powershell.exe
   |
   v
Suspicious outbound connection
```

Response:

```text
Terminate powershell.exe
```

## 5.3 Block Connection

Apply a targeted local firewall block.

Possible targets:

- IP address
- IP + port
- Subnet
- Destination connection

## 5.4 Lock Local Account / Session

Possible actions:

- Disable local account
- Lock local account
- Terminate local session
- Terminate SSH session

---

# 6. Response Architecture

Overlook SaaS should not directly control agents.

Recommended response flow:

```text
Overlook SaaS
      |
      v
Response Request
      |
      v
Overlook Edge
      |
      v
Policy / Approval / Signature Validation
      |
      v
Overlook Agent
      |
      +---- Quarantine
      +---- Kill Process
      +---- Block Connection
      +---- Lock Account
```

Every sensitive response action should support:

```text
Recommend
   |
Impact Preview
   |
Approval
   |
Signed Command
   |
Execute
   |
Verify
   |
Rollback
```

Recommended response controls:

- RBAC
- Four-eyes approval
- Command expiration
- Nonce
- Digital signature
- Audit trail
- TTL-based quarantine
- Rollback
- Maintenance-window awareness

---

# 7. Overlook AI Gateway

The endpoint agent and collector cannot reliably inspect all AI prompts because most browser and API traffic is encrypted.

Therefore, deep AI interaction visibility should be handled by an optional **Overlook AI Gateway**.

## 7.1 AI Gateway Responsibilities

- Prompt inspection
- Response inspection
- AI model identification
- AI application identification
- AI agent activity
- MCP tool calls
- AI tool invocation
- RAG activity
- Vector retrieval metadata
- Sensitive-data detection
- Prompt injection detection
- Credential/secret detection
- Source-code leakage detection
- AI policy enforcement
- AI security fact generation

Architecture:

```text
User / Application
       |
       v
Overlook AI Gateway
       |
       +---- Local Prompt Inspection
       +---- Sensitive Data Detection
       +---- Prompt Injection Detection
       +---- Tool Call Inspection
       +---- MCP Inspection
       +---- RAG Inspection
       |
       v
External / Internal AI Model
```

---

# 8. AI Security Capabilities

Overlook should expose AI Security as a first-class exposure domain.

```text
AI Security
 |
 +-- AI Inventory
 +-- Shadow AI
 +-- AI Usage
 +-- Prompt Security
 +-- Data Exposure
 +-- AI Agents
 +-- MCP / Tools
 +-- RAG Security
 +-- Model Security
 +-- AI Identities
 +-- AI Attack Paths
 +-- AI Risk
```

---

# 9. AI Inventory

Overlook should discover:

- ChatGPT
- Microsoft Copilot
- Claude
- Gemini
- GitHub Copilot
- Perplexity
- Azure OpenAI
- AWS Bedrock
- Google Vertex AI
- Internal LLM applications
- Self-hosted models
- Local models
- AI agents
- RAG applications
- Vector databases
- AI gateways
- MCP servers
- AI plugins/tools
- Model endpoints
- AI service accounts
- AI API keys

Example:

```text
Finance Agent

Model:
External / Internal LLM

Identity:
svc-finance-ai

Tools:
SharePoint
Database
Email
Browser

Privilege:
High

Autonomous Actions:
Enabled
```

---

# 10. Shadow AI

Shadow AI should be visible in Overlook.

Potential telemetry sources:

```text
Overlook Agent
+
DNS
+
Firewall
+
Proxy
+
SWG
+
CASB
+
Cloud APIs
```

Example relationship:

```text
User
 |
 v
Device
 |
 v
Browser
 |
 v
Unknown AI Service
```

Overlook should show:

- Detected AI applications
- Approved AI applications
- Unapproved AI applications
- Users accessing Shadow AI
- Departments using Shadow AI
- First seen
- Last seen
- Traffic volume
- Sensitive data interaction
- Risk score

---

# 11. Prompt Visibility

Prompts should support multiple privacy modes.

## Mode 1 – Metadata Only

Default mode.

Overlook SaaS receives:

```text
Prompt Event

Identity: ID-821
AI Application: ChatGPT
Prompt Length: 821
Sensitive Data: Yes
Secrets: No
Internal Source Data: Yes
Risk: High
```

No raw prompt content leaves the customer's environment.

## Mode 2 – Local Prompt Inspection

Recommended enterprise mode.

```text
Prompt
  |
  v
Overlook AI Gateway / Edge
  |
  +---- PII?
  +---- PCI?
  +---- PHI?
  +---- Secret?
  +---- Source Code?
  +---- Prompt Injection?
  |
  v
Security Fact
  |
  v
Overlook SaaS
```

## Mode 3 – Full Prompt Capture

Optional and explicitly customer-controlled.

This should be disabled by default.

If enabled, it requires:

- Strict RBAC
- Retention controls
- Privacy controls
- Audit logging
- Encryption
- Customer approval

---

# 12. Prompt Security

Overlook should be capable of identifying:

- Direct prompt injection
- Indirect prompt injection
- Jailbreak attempts
- Sensitive information in prompts
- Credentials in prompts
- Secrets in prompts
- Source-code leakage
- System prompt leakage
- Suspicious instructions
- Malicious links
- Data exfiltration instructions

Example:

```text
Employee
   |
   v
External AI
   |
   v
Prompt contains AWS Access Key
   |
   v
CRITICAL AI DATA EXPOSURE
```

---

# 13. AI Agent Security

Overlook should answer:

> What can this AI agent ultimately control?

Example:

```text
Finance Agent
      |
      v
Service Account
      |
      v
AWS Role
      |
      v
Production Database
```

Potential AI agent attributes:

- Model
- Application
- Owner
- Service account
- Permissions
- Tools
- MCP servers
- Data access
- External connectivity
- Autonomous capabilities
- Production access

---

# 14. AI Blast Radius

Example:

```text
Agent Compromised

Can Reach:
14 systems

Can Read:
3 sensitive databases

Can Modify:
2 applications

Can Execute:
Cloud functions

Can Send:
External email
```

This should feed directly into Overlook TrustGraph and attack-path analysis.

---

# 15. MCP Security

Overlook should discover:

- MCP servers
- MCP clients
- MCP tools
- MCP credentials
- MCP permissions
- MCP data access
- MCP-to-cloud relationships
- MCP-to-repository relationships
- MCP-to-database relationships

Example:

```text
AI Agent
   |
   v
MCP Filesystem
   |
   v
/finance/
   |
   v
Payroll.xlsx
```

Another example:

```text
User
 |
 v
AI Agent
 |
 v
GitHub MCP
 |
 v
Repository
 |
 v
CI/CD
 |
 v
AWS Production
```

---

# 16. AI Tool-Call Security

Overlook should model:

```text
Prompt
  |
  v
AI Agent
  |
  v
Tool
  |
  v
Identity
  |
  v
Action
  |
  v
Asset
```

Example:

```text
Agent:
DevOps-Agent

Tool:
AWS API

Action:
Create / Modify Infrastructure

Identity:
svc-devops-ai

Privilege:
Administrator
```

---

# 17. User Privilege vs AI Privilege

Overlook should detect when an AI agent has more privilege than the user requesting the action.

Example:

```text
Developer

Direct AWS Permission:
Read Only

        BUT

Developer
    |
    v
DevOps AI Agent
    |
    v
Service Account
    |
    v
AWS Administrator
```

Finding:

```text
AI PRIVILEGE GAP

User privilege:
READ

Agent privilege:
ADMIN

Severity:
CRITICAL
```

---

# 18. RAG Security

Overlook should discover:

- RAG applications
- Vector databases
- Vector indexes
- Source documents
- Data sensitivity
- Retrieval permissions
- Application identity
- Model identity
- Cross-user access
- Excessive retrieval
- Sensitive documents indexed
- Data poisoning risk
- Indirect prompt injection
- Unauthorized retrieval

Example:

```text
AI Application
      |
      v
Vector Database
      |
      v
Internal Documents
      |
      v
Sensitive Payroll Data
```

---

# 19. Model Inventory

Overlook should maintain a model inventory containing:

- Model
- Model provider
- Version
- Owner
- Application
- Hosting location
- Environment
- Data classification
- External/internal
- Fine-tuned/not fine-tuned
- Approved/unapproved
- Associated agents
- Associated datasets
- Associated credentials

---

# 20. AI Credential Security

Track AI-related credentials such as:

- OpenAI API keys
- Azure OpenAI credentials
- AWS Bedrock roles
- Vertex AI credentials
- Hugging Face tokens
- AI service accounts
- MCP credentials
- Vector DB credentials

Example:

```text
Developer
   |
   v
Repository
   |
   v
AI API Key
   |
   v
External Model
```

---

# 21. CSPM

The Edge Collector should provide cloud security posture capabilities for:

- AWS
- Azure
- GCP
- Kubernetes

Core CSPM features:

- Cloud asset inventory
- Misconfiguration detection
- Public exposure
- Security group analysis
- NSG analysis
- IAM posture
- Encryption posture
- Logging posture
- Kubernetes posture
- Storage exposure
- Cloud vulnerability ingestion
- Toxic combinations
- Cloud attack paths

---

# 22. DSPM

DSPM should perform sensitive data discovery and classification locally.

Potential sources:

- S3
- Azure Blob
- GCS
- RDS
- SQL Server
- Oracle
- PostgreSQL
- MongoDB
- File shares
- NAS
- SharePoint
- Data warehouses
- Vector databases

Local processing:

```text
Data Source
   |
   v
Read-only Scan
   |
   v
Local Classification
   |
   +---- PII
   +---- PCI
   +---- PHI
   +---- Secrets
   +---- Custom Sensitive Data
   |
   v
Security Fact
```

Example:

```text
DATASTORE-928
      |
      v
Contains PII

Records:
~4.2M

Sensitivity:
Critical

Exposure:
Internet Accessible
```

Raw data should not leave the customer environment.

---

# 23. ASPM

ASPM should map development assets to runtime assets.

Sources may include:

- GitHub
- GitLab
- Bitbucket
- Jenkins
- Azure DevOps
- GitHub Actions
- SAST
- SCA
- DAST
- IaC scanners
- Secret scanners
- Container scanners
- Artifact registries

Graph example:

```text
Developer
   |
   v
Repository
   |
   v
Pipeline
   |
   v
Container
   |
   v
Kubernetes
   |
   v
Production Application
```

---

# 24. Identity Security

Identity should be a core cross-domain capability.

Sources:

- Active Directory
- Entra ID
- Okta
- AWS IAM
- GCP IAM
- Kubernetes RBAC
- GitHub
- Service accounts
- API keys
- SSH keys
- Workload identities

Features:

- User discovery
- Group relationships
- Role relationships
- Service accounts
- Effective permissions
- Privilege inheritance
- Excessive privilege
- Dormant privilege
- Cross-account trust
- Privilege escalation
- Identity blast radius

---

# 25. Network Exposure

Overlook should initially focus on network reachability rather than becoming a full NDR platform.

Sources:

- FortiGate
- Palo Alto
- Check Point
- Cisco
- Juniper
- F5
- Radware
- AWS VPC
- Azure VNet
- GCP VPC
- Security Groups
- NSGs
- NACLs
- Route tables
- VPN
- NAT
- Load balancers
- DNS
- NetFlow
- IPFIX

Features:

- Network asset inventory
- Firewall rule analysis
- Security group analysis
- Network path computation
- Public exposure
- East-west reachability
- Network blast radius
- Segmentation gaps

---

# 26. Unified TrustGraph

All modules should generate common entities and relationships.

## Core entity types

```text
IDENTITY
USER
GROUP
ROLE
CREDENTIAL
ASSET
WORKLOAD
CONTAINER
APPLICATION
REPOSITORY
PIPELINE
NETWORK
FIREWALL
SECURITY_GROUP
DATABASE
DATASTORE
DATA_CLASS
VULNERABILITY
SECRET
SECURITY_CONTROL
AI_APPLICATION
AI_MODEL
AI_AGENT
MCP_SERVER
MCP_TOOL
VECTOR_DATABASE
PROMPT_EVENT
```

## Core relationships

```text
OWNS
USES
CAN_ACCESS
CAN_ASSUME
CAN_MODIFY
CAN_DEPLOY
CONNECTS_TO
ROUTES_TO
CONTAINS
EXPOSES
RUNS_ON
AUTHENTICATES_TO
TRUSTS
MEMBER_OF
PROTECTS
CAN_READ
CAN_WRITE
CAN_EXECUTE
RETRIEVES_FROM
INVOKES
```

---

# 27. AI Attack Paths

Overlook should correlate AI exposure with the rest of the security environment.

Example 1:

```text
Employee
   |
   v
Shadow AI
   |
   v
Sensitive Prompt
   |
   v
External Model
```

Example 2:

```text
Attacker
   |
   v
Indirect Prompt Injection
   |
   v
AI Agent
   |
   v
MCP Server
   |
   v
Service Account
   |
   v
Production
```

Example 3:

```text
Developer
   |
   v
AI Agent
   |
   v
Admin Service Account
   |
   v
AWS
   |
   v
Production DB
   |
   v
PII
```

---

# 28. Hybrid Attack Path Example

```text
On-Prem Active Directory
        |
        v
Developer
        |
        v
GitHub Cloud
        |
        v
CI/CD Pipeline
        |
        v
AWS Role
        |
        v
EC2
        |
        v
VPN
        |
        v
On-Prem Oracle Database
        |
        v
Sensitive Customer Data
```

This is the core value of the Overlook TrustGraph.

---

# 29. Data Privacy Model

The architecture should maintain a clear separation between customer-side telemetry and Overlook SaaS.

## Customer Environment

May contain:

- Raw logs
- API responses
- Syslog messages
- NetFlow/IPFIX
- Endpoint runtime telemetry
- Prompt contents
- AI responses
- Sensitive files
- Sensitive database values
- Source code
- Credentials
- Configuration
- Normalized local events

## Overlook SaaS

Should normally receive only:

- Tokenized identity IDs
- Asset IDs
- AI application IDs
- Model IDs
- Relationships
- Permissions
- Exposure states
- Risk scores
- Security Facts
- Attack-path edges
- Aggregated behavior
- Evidence hashes
- Confidence
- First/last seen timestamps

Example:

```text
RAW LOCAL EVENT

akilesh@company.com
connected to
prod-db.company.local
using privileged SSH

             |
             v

SECURITY FACT

IDENTITY-829
     |
PRIVILEGED_ACCESS
     |
     v
ASSET-372
```

---

# 30. Compression and Encryption

Locally retained datasets should follow:

```text
Normalized / Derived Dataset
          |
          v
Compression
          |
          v
Encryption
          |
          v
Encrypted Local Storage
```

Recommended baseline:

- Compression: Zstandard or equivalent
- Data encryption: AES-256-GCM
- Transport: TLS 1.3
- Mutual authentication: mTLS
- Integrity: SHA-256 / SHA-384
- Command signing: ECDSA or Ed25519
- Key derivation: HKDF
- Password hashing: Argon2id
- Customer key management: KMS/HSM
- Envelope encryption
- Crypto agility for post-quantum algorithms

---

# 31. Connector Credential Model

Avoid read-only "super-admin" accounts where possible.

Use least-privilege connector roles.

Example:

```text
OverlookAWSInventoryReader
OverlookAzureSecurityReader
OverlookGitHubSecurityReader
OverlookFortiGateReader
```

Response privileges should be separate:

```text
OverlookSecurityReader

        +

OverlookResponseRole
```

Customers can choose whether to enable response privileges.

---

# 32. On-Premises Deployment

```text
                     CUSTOMER DATA CENTER

 Active Directory ------------------+
 VMware / Hyper-V ------------------+
 GitLab / Jenkins ------------------+
 Databases -------------------------+
 File Shares / NAS -----------------+
 Firewalls -------------------------+
 Routers / Switches ----------------+
 NetFlow / IPFIX -------------------+
 EDR -------------------------------+
 Servers + Overlook Agent ----------+
 Internal AI -----------------------+
 MCP / RAG -------------------------+
                                     |
                                     v
                       OVERLOOK EDGE ANALYTICS NODE
                                     |
                              Security Facts
                                     |
                                mTLS / 443
                                     |
                                     v
                              OVERLOOK SaaS
```

For segmented environments, optional lightweight satellite collectors may be added.

---

# 33. Cloud Deployment

```text
                        CUSTOMER CLOUD

 AWS / Azure / GCP ------------------+
 IAM / Entra / GCP IAM --------------+
 Cloud Audit Logs -------------------+
 Security Services -----------------+
 Kubernetes -------------------------+
 GitHub / CI-CD ---------------------+
 Object Storage ---------------------+
 Cloud Databases --------------------+
 VPC / VNet -------------------------+
 Load Balancers ---------------------+
 VMs + Overlook Agent ---------------+
 AI Services ------------------------+
 AI Agents / MCP / RAG --------------+
                                      |
                                      v
                         OVERLOOK EDGE NODE
                         Private Subnet
                         No Public IP
                                      |
                                Security Facts
                                      |
                                  mTLS / 443
                                      |
                                      v
                               OVERLOOK SaaS
```

The Edge Collector should initiate customer-side API calls.

Overlook SaaS should normally not directly access customer cloud accounts.

---

# 34. Modular Deployment Levels

Customers should not be forced to deploy every component.

## Level 1 – Edge Only

```text
Overlook Edge
```

Provides:

- CSPM
- DSPM
- ASPM
- Identity visibility
- Network exposure
- Asset inventory
- AI inventory
- Basic Shadow AI discovery
- TrustGraph
- Attack paths
- Risk intelligence

---

## Level 2 – Edge + Agent

```text
Overlook Edge
     +
Overlook Agent
```

Adds:

- Host runtime visibility
- Process relationships
- User/process context
- Better Shadow AI detection
- Local AI discovery
- MCP configuration visibility
- Network connections
- Host response actions

Response:

- Quarantine Host
- Terminate Process
- Block Connection
- Lock Local Account/Session

---

## Level 3 – Edge + Agent + AI Gateway

```text
Overlook Edge
     +
Overlook Agent
     +
Overlook AI Gateway
```

Adds:

- Prompt Security
- AI Response inspection
- AI Agent Security
- MCP Security
- Tool-call analysis
- RAG Security
- Sensitive AI interaction analysis
- Prompt injection detection
- AI data-loss context
- User privilege vs AI privilege analysis
- AI attack paths

---

# 35. AI Visibility Matrix

| Capability | Edge Collector | Local Agent | AI Gateway / Integration |
|---|---:|---:|---:|
| AI application inventory | Yes | Yes | Optional |
| Shadow AI | Yes | Yes | Optional |
| User to AI mapping | Yes | Yes | IdP/Proxy improves |
| AI API usage | Yes | Yes | Gateway improves |
| Model inventory | Yes | Limited | Cloud/K8s integration |
| AI agent inventory | Yes | Yes | App/API integration |
| MCP server discovery | Yes | Yes | MCP integration improves |
| AI service accounts | Yes | Limited | IAM required |
| AI API key metadata | Yes | Yes | Repo/secret integration |
| Prompt metadata | Yes | Yes | Yes |
| Full prompt content | Sometimes | Limited | Yes |
| Sensitive data in prompts | If visible | Limited | Yes |
| Prompt injection | If visible | Limited | Yes |
| System prompt exposure | App integration | Limited | Yes |
| AI tool calls | Connector/API | Limited | Yes |
| User vs agent privilege | Yes | Limited | IAM + agent context |
| AI attack paths | Contributes | Contributes | Contributes |
| RAG inventory | Yes | Yes | App/vector DB APIs |
| Vector DB relationships | Yes | Limited | API integration |
| AI response actions | Connector | Host only | Gateway policy |

---

# 36. Product Dashboard

The main dashboard should remain lightweight.

Recommended top-level sections:

```text
OVERLOOK

Exposure Score
Critical Attack Paths
Crown Jewels at Risk
Critical Findings
Active Response

Security Coverage:
Cloud
Applications
Data
Identity
Network
Runtime
AI

Risk Contribution

Top Critical Attack Paths

Crown Jewels

Platform Health
```

---

# 37. AI Security Dashboard

The AI dashboard should remain focused.

```text
AI SECURITY

AI Assets
Shadow AI
AI Agents
Risky Prompts
Sensitive Data Events
Critical AI Paths
```

Recommended navigation:

```text
AI Inventory
Shadow AI
Prompts
Data
Agents
MCP
RAG
Attack Paths
```

---

# 38. Product Positioning

Overlook should not be positioned as:

```text
SIEM
+
EDR
+
SOAR
+
CSPM
+
DSPM
+
ASPM
+
NDR
+
AI DLP
```

Instead:

> **Overlook is a privacy-preserving cyber exposure intelligence platform that correlates cloud, application, data, identity, network, runtime, and AI relationships into a unified TrustGraph to identify how attackers or AI agents can ultimately reach and control critical business assets.**

The individual modules provide context.

The primary differentiation should remain:

1. Edge-local analytics
2. Privacy-preserving Security Facts
3. Unified Entity Resolution
4. TrustGraph
5. Cross-domain correlation
6. Hybrid attack paths
7. AI/agent privilege paths
8. Blast radius
9. Change intelligence
10. Break Attack Path response

---

# 39. Recommended Core Product Components

```text
OVERLOOK PLATFORM

1. Overlook Edge Analytics Node
2. Overlook Agent
3. Overlook AI Gateway
4. CSPM
5. DSPM
6. ASPM
7. Identity Exposure
8. Network Exposure
9. AI Security
10. Runtime Security Context
11. Unified Asset Intelligence
12. Entity Resolution Engine
13. TrustGraph
14. Correlation Engine
15. Attack Path Engine
16. Risk Engine
17. Change Intelligence
18. Investigation
19. Response
20. Privacy & Cryptography
21. Compliance
22. Integrations
23. Administration
24. CISO Dashboard
25. Analyst Workspace
```

---

# 40. Final Architectural Principle

The architecture should enforce:

```text
DISCOVER
   |
   v
PROCESS LOCALLY
   |
   v
UNDERSTAND
   |
   v
MINIMIZE DATA
   |
   v
BUILD SECURITY FACTS
   |
   v
CORRELATE
   |
   v
TRUSTGRAPH
   |
   v
ATTACK PATH
   |
   v
PRIORITIZE
   |
   v
BREAK PATH
```

The **Edge Collector** provides broad environment intelligence.

The **Agent** provides runtime visibility and host response.

The **AI Gateway** provides deep AI interaction, prompt, agent, MCP, and RAG visibility.

The **Overlook SaaS** performs the cross-domain intelligence, TrustGraph, attack-path, risk, investigation, and response orchestration.

This separation keeps Overlook modular, privacy-focused, scalable, and suitable for both cloud and on-premises deployments.

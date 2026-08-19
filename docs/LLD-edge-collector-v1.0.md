# Overlook Edge Collector – Low Level Design (LLD)

**Document Type:** Low Level Design  
**Component:** Overlook Edge Collector  
**Version:** 1.0  
**Status:** Proposed  
**Primary Language:** Go  
**UI:** React + TypeScript  

---

## 1. Purpose

The Overlook Edge Collector is deployed inside the customer environment and acts as the trusted bridge between customer security systems and the Overlook SaaS platform.

Its primary responsibilities are:

```text
Collect
   ↓
Buffer
   ↓
Parse
   ↓
Normalize
   ↓
Enrich
   ↓
Generate Security Facts
   ↓
Privacy Minimize
   ↓
Compress
   ↓
Encrypt
   ↓
Forward Metadata
```

Raw customer telemetry should remain within the customer environment unless explicitly configured otherwise.

---

# 2. Design Principles

The Collector must follow these principles:

1. Outbound-first communication.
2. No direct Internet exposure required.
3. Customer credentials remain locally protected.
4. Raw logs are not forwarded by default.
5. Processing should continue during SaaS outages.
6. Collector failure must not impact customer workloads.
7. Components must support horizontal worker scaling.
8. Backpressure must be supported.
9. High-priority security events must not be dropped.
10. All Collector-to-SaaS communication uses mTLS.
11. The Collector should operate independently for temporary periods without SaaS connectivity.
12. Collector upgrades must be signed and controlled.

---

# 3. Logical Architecture

```text
                     CUSTOMER ENVIRONMENT

 ┌──────────────────────────────────────────────────────┐
 │                 OVERLOOK COLLECTOR                   │
 │                                                      │
 │  ┌────────────────────────────────────────────────┐  │
 │  │                CONNECTORS                      │  │
 │  │                                                │  │
 │  │ AWS | Azure | GCP | FortiGate | EDR | GitHub │  │
 │  │ DB | Syslog | REST | Webhook | Agent          │  │
 │  └──────────────────────┬─────────────────────────┘  │
 │                         │                            │
 │                         ▼                            │
 │  ┌────────────────────────────────────────────────┐  │
 │  │           INGESTION GATEWAY                    │  │
 │  │ Auth | Validation | Flow Control | Rate Limit │  │
 │  └──────────────────────┬─────────────────────────┘  │
 │                         │                            │
 │                         ▼                            │
 │                NATS JETSTREAM                       │
 │                         │                            │
 │                         ▼                            │
 │                  PARSER ENGINE                      │
 │                         │                            │
 │                         ▼                            │
 │                NORMALIZATION ENGINE                 │
 │                         │                            │
 │                         ▼                            │
 │                  ENRICHMENT ENGINE                  │
 │                         │                            │
 │                         ▼                            │
 │                SECURITY FACT ENGINE                 │
 │                         │                            │
 │                         ▼                            │
 │                  PRIVACY ENGINE                     │
 │                         │                            │
 │                         ▼                            │
 │               COMPRESS + ENCRYPT                    │
 │                         │                            │
 │                         ▼                            │
 │                METADATA FORWARDER                   │
 │                         │                            │
 │                       mTLS                           │
 └─────────────────────────┼────────────────────────────┘
                           │
                           ▼
                    OVERLOOK SAAS
```

---

# 4. Collector Planes

The Collector is separated into three logical planes.

## 4.1 Data Plane

Responsible for high-volume telemetry processing.

```text
Connector
   ↓
Ingestion Gateway
   ↓
NATS
   ↓
Parser
   ↓
Normalizer
   ↓
Enrichment
   ↓
Fact Engine
   ↓
Privacy Engine
   ↓
Forwarder
```

---

## 4.2 Control Plane

Responsible for configuration and administration.

```text
Collector UI
      │
      ▼
Local Management API
      │
      ├── Connector Manager
      ├── Configuration Manager
      ├── Credential Vault
      ├── Certificate Manager
      ├── Health Manager
      ├── Parser Manager
      └── Update Manager
```

---

## 4.3 Response Plane

Responsible for controlled security response actions.

```text
Overlook SaaS
      │
      │ Signed Response Request
      ▼
Response Gateway
      │
      ├── Authorization
      ├── Policy Validation
      ├── Command Validation
      ├── Audit Logging
      │
      ├──────── Cloud Connector
      │
      └──────── Endpoint Agent
```

---

# 5. Process Architecture

Recommended Collector processes:

```text
Linux
│
├── overlook-collector
│
├── nats-server
│
└── overlook-updater
```

The Collector application itself should remain primarily a modular monolith.

Do not create separate operating-system processes for every module during the initial versions.

---

# 6. Main Collector Process

```text
overlook-collector
│
├── API Server
├── Connector Manager
├── Ingestion Manager
├── Queue Manager
├── Parser Manager
├── Normalization Workers
├── Enrichment Workers
├── Security Fact Workers
├── Privacy Workers
├── Forwarder Workers
├── Agent Gateway
├── Response Gateway
├── Credential Vault
├── Health Manager
└── Configuration Manager
```

---

# 7. Collector Directory Structure

```text
collector/
│
├── cmd/
│   └── collector/
│       └── main.go
│
├── internal/
│   │
│   ├── collector/
│   │   ├── lifecycle.go
│   │   └── config.go
│   │
│   ├── connectors/
│   │   ├── connector.go
│   │   ├── manager.go
│   │   ├── aws/
│   │   ├── azure/
│   │   ├── gcp/
│   │   ├── fortigate/
│   │   ├── syslog/
│   │   ├── github/
│   │   ├── database/
│   │   └── genericrest/
│   │
│   ├── ingestion/
│   │   ├── gateway.go
│   │   ├── validator.go
│   │   ├── ratelimit.go
│   │   └── flowcontrol.go
│   │
│   ├── queue/
│   │   ├── nats.go
│   │   └── subjects.go
│   │
│   ├── parser/
│   │   ├── parser.go
│   │   ├── detector.go
│   │   ├── registry.go
│   │   └── failure.go
│   │
│   ├── normalize/
│   │   ├── schema.go
│   │   └── normalizer.go
│   │
│   ├── enrichment/
│   │   ├── asset.go
│   │   ├── identity.go
│   │   ├── network.go
│   │   ├── cloud.go
│   │   └── threat.go
│   │
│   ├── facts/
│   │   ├── fact.go
│   │   ├── entity.go
│   │   ├── relation.go
│   │   └── builder.go
│   │
│   ├── privacy/
│   │   ├── redact.go
│   │   ├── mask.go
│   │   └── policy.go
│   │
│   ├── crypto/
│   │   ├── encrypt.go
│   │   ├── cert.go
│   │   └── keystore.go
│   │
│   ├── compression/
│   │   └── zstd.go
│   │
│   ├── forwarder/
│   │   ├── batch.go
│   │   ├── sender.go
│   │   ├── retry.go
│   │   └── checkpoint.go
│   │
│   ├── agents/
│   │   ├── gateway.go
│   │   ├── register.go
│   │   └── heartbeat.go
│   │
│   ├── response/
│   │   ├── command.go
│   │   ├── validator.go
│   │   ├── executor.go
│   │   └── audit.go
│   │
│   ├── vault/
│   │   └── vault.go
│   │
│   ├── health/
│   │   ├── metrics.go
│   │   └── status.go
│   │
│   └── api/
│       ├── routes.go
│       └── middleware.go
│
├── schemas/
├── parsers/
├── web/
├── configs/
├── migrations/
└── scripts/
```

---

# 8. Collector Startup Sequence

```text
overlook-collector start
        │
        ▼
Load local configuration
        │
        ▼
Validate Collector identity
        │
        ▼
Open encrypted key store
        │
        ▼
Open State DB
        │
        ▼
Connect to NATS
        │
        ▼
Load parser registry
        │
        ▼
Load configured connectors
        │
        ▼
Initialize worker pools
        │
        ▼
Start health service
        │
        ▼
Connect to Overlook SaaS
        │
        ▼
Start connector collection
```

Startup should fail only for critical dependencies.

For example:

```text
State database unavailable → Collector FAIL

NATS unavailable → Collector FAIL

Overlook SaaS unavailable → Collector CONTINUES

One connector unavailable → Connector DEGRADED

Threat intelligence unavailable → Continue without enrichment
```

---

# 9. Collector Identity

Each Collector receives a unique identity.

Example:

```json
{
  "collector_id": "col-01JX92HD82",
  "tenant_id": "tenant-acme",
  "name": "ACME-Singapore-Collector",
  "environment": "production",
  "site": "Singapore",
  "version": "1.0.0"
}
```

Collector identity must be cryptographically bound to its certificate.

---

# 10. Connector Interface

Every connector implements the common interface.

```go
type Connector interface {
    ID() string
    Type() string

    Validate(ctx context.Context) error

    Start(ctx context.Context) error

    Health(ctx context.Context) Health

    Stop(ctx context.Context) error
}
```

For polling connectors:

```go
type PollingConnector interface {
    Connector

    Collect(
        ctx context.Context,
        checkpoint Checkpoint,
    ) ([]RawEvent, Checkpoint, error)
}
```

---

# 11. Connector Configuration Model

```json
{
  "id": "con-aws-prod",
  "name": "AWS Production",
  "type": "aws",
  "enabled": true,

  "auth": {
    "type": "assume_role",
    "vault_ref": "vault://aws-prod"
  },

  "collection": {
    "security_hub": true,
    "guardduty": true,
    "cloudtrail": true,
    "ec2": true,
    "iam": true,
    "s3": true
  },

  "poll_interval_seconds": 60,

  "priority": 1
}
```

Credentials are referenced.

Credentials themselves must never be returned through normal management APIs.

---

# 12. Supported Collection Methods

Collector framework should support:

```text
REST API polling

REST API push

Webhook

Syslog TCP

Syslog UDP

Syslog TLS

Agent telemetry

Cloud API

Message queue

Database polling

File ingestion

Object storage

Streaming API
```

---

# 13. Raw Event Model

Every received event should first be wrapped inside an internal event envelope.

```json
{
  "event_id": "evt-019289",
  "tenant_id": "tenant-acme",
  "collector_id": "col-sg-01",

  "connector_id": "aws-prod",
  "connector_type": "aws",

  "received_at": "2026-08-18T10:30:10Z",

  "format": "json",

  "priority": 1,

  "payload": {}
}
```

The payload remains only within the Collector processing pipeline.

---

# 14. NATS JetStream Subjects

Recommended subject hierarchy:

```text
overlook.raw.aws
overlook.raw.azure
overlook.raw.fortigate
overlook.raw.syslog
overlook.raw.agent

overlook.parsed

overlook.normalized

overlook.enriched

overlook.fact

overlook.forward

overlook.retry

overlook.deadletter
```

---

# 15. Streams

## RAW Stream

```text
Name:
OVERLOOK_RAW

Subjects:
overlook.raw.*

Storage:
File

Retention:
Limits

Replication:
1
```

---

## PROCESSING Stream

```text
Name:
OVERLOOK_PROCESSING

Subjects:
overlook.parsed
overlook.normalized
overlook.enriched

Storage:
File
```

---

## FORWARD Stream

```text
Name:
OVERLOOK_FORWARD

Subjects:
overlook.fact
overlook.forward
overlook.retry

Storage:
File
```

---

# 16. Consumer Model

```text
RAW
 │
 └── parser-workers
          │
          ▼
      PARSED
          │
 └── normalization-workers
          │
          ▼
     NORMALIZED
          │
 └── enrichment-workers
          │
          ▼
      ENRICHED
          │
 └── fact-workers
          │
          ▼
      SECURITY FACT
          │
 └── privacy-workers
          │
          ▼
      FORWARD
```

Each worker ACKs an event only after successful processing.

---

# 17. Worker Pools

Worker pools should be configurable.

Example:

```yaml
workers:

  parser:
    min: 2
    max: 16

  normalization:
    min: 2
    max: 12

  enrichment:
    min: 2
    max: 8

  facts:
    min: 2
    max: 8

  forwarding:
    min: 2
    max: 8
```

Worker count can dynamically increase based on:

```text
Queue depth
EPS
CPU utilization
Processing latency
```

---

# 18. Parser Framework

Parser interface:

```go
type Parser interface {

    Name() string

    Supports(
        source Source,
        sample []byte,
    ) bool

    Parse(
        ctx context.Context,
        raw RawEvent,
    ) ([]ParsedEvent, error)
}
```

---

# 19. Parser Selection

```text
Incoming Event
       │
       ▼
Check Connector Parser
       │
       ├── Parser configured
       │        ↓
       │     Use parser
       │
       └── No parser
                ↓
        Format detector
                │
         ┌──────┼──────┐
         ↓      ↓      ↓
        JSON   CEF    Syslog
         │      │      │
         └──────┼──────┘
                ↓
          Generic Parser
```

---

# 20. Parser Failure Handling

Parser failures must not drop events.

```text
Parser Error
     │
     ▼
Increment failure counter
     │
     ▼
Store failure metadata
     │
     ▼
overlook.deadletter
```

Dead-letter record:

```json
{
  "event_id": "evt-100232",
  "connector": "fortigate-prod",
  "parser": "fortigate-v7",
  "reason": "unexpected field structure",
  "attempts": 3,
  "first_seen": "...",
  "last_attempt": "..."
}
```

---

# 21. Normalized Schema

Normalized events should use one common schema.

Core fields:

```text
event.id
event.kind
event.type
event.category
event.action
event.outcome
event.severity

timestamp

source.ip
source.port

destination.ip
destination.port

network.protocol
network.transport

user.id
user.name

asset.id
asset.name
asset.hostname
asset.type

process.pid
process.name
process.hash

file.name
file.path
file.hash

cloud.provider
cloud.account
cloud.region
cloud.resource_id

application.id
application.name

database.name

security.rule_id
security.rule_name
```

---

# 22. Enrichment Engine

The enrichment engine adds context.

```text
Normalized Event
      │
      ├── Asset enrichment
      ├── Identity enrichment
      ├── Cloud enrichment
      ├── Network enrichment
      ├── Application enrichment
      └── Threat enrichment
              │
              ▼
        Enriched Event
```

---

# 23. Security Fact

Security Fact is the primary object forwarded to Overlook SaaS.

Example:

```json
{
  "fact_id": "fact-892188",

  "tenant_id": "tenant-acme",

  "collector_id": "col-sg-01",

  "fact_type": "identity_access",

  "subject": {
    "type": "identity",
    "id": "john@acme.com"
  },

  "action": "assume_role",

  "object": {
    "type": "cloud_role",
    "id": "ProductionAdmin"
  },

  "context": {
    "environment": "production",
    "cloud": "aws"
  },

  "severity": "high",

  "risk_score": 78,

  "observed_at": "2026-08-18T10:31:00Z"
}
```

---

# 24. Entity Object

```json
{
  "entity_id": "asset-i12345",

  "entity_type": "cloud_workload",

  "properties": {
    "provider": "aws",
    "account": "123456789",
    "region": "ap-southeast-1",
    "type": "ec2"
  }
}
```

---

# 25. Relationship Object

```json
{
  "source": "john@acme.com",

  "relation": "CAN_ASSUME",

  "target": "ProductionAdmin",

  "confidence": 0.98,

  "first_seen": "...",

  "last_seen": "..."
}
```

These relationships later populate the Overlook security graph.

---

# 26. Privacy Engine

Before forwarding:

```text
Security Fact
      │
      ▼
Privacy Policy
      │
      ├── Remove raw payload
      ├── Remove passwords
      ├── Remove secrets
      ├── Remove API tokens
      ├── Mask selected PII
      ├── Hash identifiers where configured
      └── Remove unnecessary fields
```

---

# 27. Sensitive Field Detection

Sensitive fields include:

```text
password
passwd
secret
token
access_token
refresh_token
authorization
api_key
private_key
session_token
cookie
credential
```

Values should never be forwarded unless explicitly allowed by policy.

---

# 28. Compression

Use Zstandard.

Pipeline:

```text
Fact Batch
   ↓
Serialize
   ↓
ZSTD Compression
   ↓
Encryption
```

---

# 29. Encryption

Local persistent spool:

```text
AES-256-GCM
```

Transport:

```text
TLS 1.3
+
mTLS
```

Collector private keys must never leave the Collector.

---

# 30. Forwarding Batch

Instead of sending one event per HTTPS call:

```text
Facts
 ↓
Batch
 ↓
Compress
 ↓
Encrypt
 ↓
Send
```

Example configurable batch:

```yaml
forwarding:

  max_events: 500

  max_size_mb: 4

  max_wait_seconds: 2
```

Send immediately if any condition is reached.

---

# 31. Forwarding Protocol

Recommended:

```text
Collector
   │
   │ HTTPS / gRPC
   │ TLS 1.3 + mTLS
   ▼
Overlook SaaS Ingestion API
```

Longer-term gRPC streaming can be used where appropriate.

---

# 32. Forward Request

Conceptually:

```http
POST /api/v1/collector/ingest
```

Headers:

```text
X-Collector-ID
X-Tenant-ID
X-Batch-ID
X-Collector-Version
Content-Encoding: zstd
```

Body:

```text
Encrypted compressed payload
```

---

# 33. SaaS Acknowledgment

Response:

```json
{
  "batch_id": "batch-191020",
  "accepted": true,
  "accepted_events": 500,

  "server_time": "...",

  "next_poll_seconds": 2
}
```

Only after acknowledgment does the Collector mark the events as successfully forwarded.

---

# 34. Retry Logic

Recommended retry strategy:

```text
Attempt 1 → immediate

Attempt 2 → 1 sec

Attempt 3 → 5 sec

Attempt 4 → 15 sec

Attempt 5 → 30 sec

Then exponential backoff
```

Example maximum:

```text
5 minutes
```

while data remains safely buffered locally.

---

# 35. SaaS Outage

If SaaS becomes unavailable:

```text
SaaS OFFLINE
     │
     ▼
Forwarding fails
     │
     ▼
NATS buffer
     │
     ▼
Encrypted disk spool
     │
     ▼
Continue collecting
```

Collector UI should show:

```text
SaaS Connectivity: DEGRADED

Buffered Events: 8,281,000

Estimated Buffer Remaining: 31 hours
```

---

# 36. Flow Control

Flow control should consider:

```text
Incoming EPS

Processing EPS

Queue depth

CPU

RAM

Disk availability

Forwarding speed
```

---

# 37. Pressure Levels

### Green

```text
Queue < 50%
CPU < 70%
Disk < 70%
```

Normal operation.

### Yellow

```text
Queue 50–75%
```

Increase worker count.

### Orange

```text
Queue 75–90%
```

Throttle low priority telemetry.

### Red

```text
Queue > 90%
```

Protect high-priority telemetry.

---

# 38. Event Priorities

```text
P0
Critical incident / response telemetry

P1
Security alerts

P2
Configuration / posture findings

P3
Inventory and informational telemetry

P4
Debug / verbose telemetry
```

During pressure:

```text
P0 → Never discard

P1 → Never discard

P2 → Buffer

P3 → Throttle

P4 → Sample / discard if policy permits
```

---

# 39. Rate Limiting

Rate limits should exist:

```text
Per Connector

Per Source IP

Per API

Per Agent

Per Tenant

Per Event Type
```

Token-bucket or equivalent algorithms can be used.

---

# 40. Deduplication

Duplicate fingerprint:

```text
hash(
    connector_id +
    source_event_id +
    timestamp +
    event_type
)
```

Short-term deduplication can initially be implemented using an in-memory TTL cache.

Redis is not mandatory.

---

# 41. Local State Database

For the first implementation use:

```text
SQLite
```

Store:

```text
collector configuration
connector metadata
checkpoints
certificate metadata
agent records
forwarding checkpoint
parser configuration
health state
response audit metadata
```

Do not use SQLite for high-volume raw event storage.

---

# 42. Main Tables

```text
collectors

connectors

connector_checkpoints

parsers

agents

certificates

forwarding_checkpoints

response_commands

audit_events

settings
```

---

# 43. Connector Table

```text
connectors
──────────────────────────────
id
name
type
enabled
auth_reference
config_json
parser
priority
created_at
updated_at
last_health
```

---

# 44. Checkpoint Table

```text
connector_checkpoints
──────────────────────────────
connector_id
checkpoint_type
checkpoint_value
last_event_time
updated_at
```

This prevents duplicate polling after restart.

---

# 45. Credential Vault

Architecture:

```text
Connector
   │
   ▼
Credential Reference
   │
   ▼
Local Credential Vault
   │
   ▼
Encrypted Secret
```

Configuration:

```text
vault://fortigate-prod
```

instead of:

```text
password=MySecretPassword
```

---

# 46. Collector Local API

Local management interface:

```text
/api/v1/health

/api/v1/status

/api/v1/connectors

/api/v1/connectors/{id}

/api/v1/connectors/{id}/test

/api/v1/connectors/{id}/health

/api/v1/parsers

/api/v1/agents

/api/v1/system

/api/v1/metrics
```

---

# 47. Create Connector

```http
POST /api/v1/connectors
```

Example:

```json
{
  "name": "FortiGate Production",

  "type": "fortigate_syslog",

  "listen_port": 6514,

  "protocol": "tls",

  "parser": "fortigate-v7",

  "priority": 1
}
```

---

# 48. Test Connector

```http
POST /api/v1/connectors/{id}/test
```

Response:

```json
{
  "status": "healthy",

  "authentication": "success",

  "latency_ms": 46,

  "permissions": [
    "read-events",
    "read-assets"
  ]
}
```

---

# 49. Collector Health API

```http
GET /api/v1/health
```

Response:

```json
{
  "status": "healthy",

  "nats": "healthy",

  "state_db": "healthy",

  "saas": "connected",

  "connectors": {
    "healthy": 8,
    "warning": 1,
    "failed": 0
  }
}
```

---

# 50. Metrics

Expose metrics such as:

```text
overlook_events_received_total

overlook_events_parsed_total

overlook_parse_failures_total

overlook_events_normalized_total

overlook_facts_generated_total

overlook_events_forwarded_total

overlook_forward_failures_total

overlook_queue_depth

overlook_connector_status

overlook_processing_latency

overlook_saas_latency

overlook_agents_connected
```

---

# 51. Health States

Every component:

```text
HEALTHY

DEGRADED

FAILED

UNKNOWN

DISABLED
```

---

# 52. Connector Health

Example:

```json
{
  "connector_id": "aws-production",

  "status": "healthy",

  "last_success": "...",

  "last_event": "...",

  "events_5m": 19482,

  "errors_5m": 0,

  "latency_ms": 123
}
```

---

# 53. Logging

Collector logs:

```text
collector.log

connector.log

parser.log

forwarder.log

security.log

audit.log
```

Logs should be structured JSON internally.

Example:

```json
{
  "level": "warn",

  "component": "forwarder",

  "message": "SaaS ingestion unavailable",

  "batch_id": "batch-182",

  "retry": 3
}
```

Secrets must automatically be redacted from logs.

---

# 54. Audit Logging

Audit all administrative operations:

```text
Connector created

Connector deleted

Credential changed

Parser changed

Collector configuration changed

Certificate rotated

Response action requested

Response action executed

Collector upgraded
```

---

# 55. Agent Gateway

Endpoint agents connect:

```text
Agent
  │
  │ mTLS
  ▼
Collector Agent Gateway
```

Agent registration flow:

```text
Agent starts
    ↓
Bootstrap token
    ↓
Collector validation
    ↓
Certificate issued
    ↓
Agent identity registered
    ↓
Persistent mTLS communication
```

---

# 56. Agent Heartbeat

Example:

```json
{
  "agent_id": "agt-server-001",

  "hostname": "prod-web-01",

  "os": "linux",

  "version": "1.0.0",

  "status": "healthy",

  "last_seen": "..."
}
```

---

# 57. Response Command

Example command:

```json
{
  "command_id": "rsp-29182",

  "action": "quarantine_asset",

  "target": {
    "type": "agent",
    "id": "agt-server-001"
  },

  "requested_by": "user-101",

  "reason": "Suspected compromise",

  "expires_at": "...",

  "signature": "..."
}
```

---

# 58. Response Validation

Before execution:

```text
Verify command signature
        ↓
Verify tenant
        ↓
Verify Collector
        ↓
Verify target
        ↓
Verify response policy
        ↓
Verify user authorization
        ↓
Check expiry
        ↓
Audit
        ↓
Execute
```

---

# 59. Initial Response Actions

The initial response framework should support:

```text
Quarantine endpoint/workload

Kill process

Block network destination

Terminate network connection

Disable local account/session

Revoke cloud token

Disable cloud access key

Apply quarantine security group
```

The available actions depend on the connected agent/API capabilities.

---

# 60. SaaS Communication Model

Prefer a single outbound connection.

```text
Collector
    │
    ├── Telemetry
    ├── Health
    ├── Security Facts
    ├── Agent Status
    └── Response Status
          │
          ▼
     Overlook SaaS
```

The SaaS should not need to directly connect inbound to the Collector.

---

# 61. Ports

Recommended defaults:

```text
443/TCP
Collector → Overlook SaaS

4222/TCP
Collector internal → NATS

8222/TCP
NATS monitoring
LOCALHOST ONLY

6514/TCP
TLS Syslog

514/TCP
Optional Syslog TCP

514/UDP
Optional Syslog UDP

8443/TCP
Collector Local UI/API

9443/TCP
Agent Gateway
```

Ports should remain configurable.

---

# 62. Firewall Requirements

Minimum outbound:

```text
Collector → Overlook SaaS : TCP 443
```

Additional connector-specific traffic may include:

```text
AWS APIs : TCP 443

Azure APIs : TCP 443

GCP APIs : TCP 443

GitHub : TCP 443

FortiGate API : TCP 443

Databases : DB-specific ports
```

Inbound ports only need to be exposed inside the customer's trusted network when required.

---

# 63. Collector UI

UI should remain lightweight.

Primary sections:

```text
Dashboard

Connectors

Agents

Parsing

Queue

System

Security

Diagnostics
```

---

# 64. Dashboard

```text
OVERLOOK COLLECTOR

Collector: SG-COL-01

Status: HEALTHY

SaaS: CONNECTED

──────────────

Events/sec
12,451

Queue
4.2 GB

CPU
42%

RAM
51%

Disk
28%

──────────────

Connectors

AWS Production     HEALTHY

FortiGate          HEALTHY

CrowdStrike        HEALTHY

GitHub             WARNING
```

---

# 65. Connector UI

Display:

```text
Name

Type

Connection Status

Events/sec

Last Event

Parser

Error Rate

Last Sync
```

Actions:

```text
Configure

Test

Disable

Restart

View Diagnostics
```

---

# 66. Deployment Layout

Recommended filesystem:

```text
/opt/overlook/
│
├── bin/
│   └── overlook-collector
│
├── web/
│
├── parsers/
│
└── scripts/


/etc/overlook/
│
├── collector.yaml
│
├── certificates/
│
└── secrets/


/var/lib/overlook/
│
├── state/
│
├── nats/
│
├── spool/
│
└── updates/


/var/log/overlook/
```

---

# 67. Linux Permissions

Use a dedicated service account:

```text
overlook
```

Example:

```text
/etc/overlook/secrets

Owner:
overlook

Permissions:
700
```

Individual secrets:

```text
600
```

---

# 68. systemd Service

Conceptually:

```ini
[Unit]
Description=Overlook Edge Collector
After=network-online.target nats.service

[Service]
User=overlook

ExecStart=/opt/overlook/bin/overlook-collector

Restart=always

RestartSec=5

NoNewPrivileges=true

PrivateTmp=true

ProtectSystem=strict

[Install]
WantedBy=multi-user.target
```

---

# 69. Configuration

Example:

```yaml
collector:

  id: col-sg-01

  name: Singapore-Collector

  tenant: acme


saas:

  endpoint: https://collector.overlook.example

  mtls: true


queue:

  max_disk_gb: 300


processing:

  parser_workers: 8

  normalization_workers: 8

  fact_workers: 4


forwarding:

  max_batch_events: 500

  max_batch_mb: 4

  flush_seconds: 2


retention:

  raw_hours: 24

  deadletter_days: 7
```

---

# 70. Local Storage Policy

Recommended initial behavior:

```text
Raw Queue
6–24 hours

Parsed Queue
6–24 hours

Security Facts
72 hours or until acknowledged

Dead Letter
7 days

Operational Logs
30 days
```

All disk-backed sensitive data should be encrypted.

---

# 71. Collector Sizing

## Small

```text
Up to ~2,000 EPS

4 vCPU

16 GB RAM

250 GB SSD
```

## Medium

```text
~2,000–5,000 EPS

8 vCPU

32 GB RAM

500 GB SSD
```

## Large

```text
~5,000–10,000+ EPS

12 vCPU

64 GB RAM

1 TB SSD
```

Actual supported EPS must ultimately be determined through benchmark testing rather than treated as a guaranteed figure.

---

# 72. Failure Matrix

| Failure | Collector Behaviour |
|---|---|
| SaaS unavailable | Continue locally |
| Connector unavailable | Mark degraded |
| Parser failure | Dead-letter |
| NATS unavailable | Collector degraded/fail processing |
| State DB unavailable | Collector fail-safe |
| Disk >90% | Apply pressure controls |
| Credential expired | Connector failed |
| Certificate near expiry | Raise warning |
| Agent offline | Mark offline |
| Enrichment unavailable | Continue without enrichment |

---

# 73. Graceful Shutdown

On:

```text
systemctl stop overlook-collector
```

Collector should:

```text
Stop accepting new local work
       ↓
Stop connector polling
       ↓
Drain workers
       ↓
ACK completed events
       ↓
Persist checkpoints
       ↓
Close SaaS connection
       ↓
Close NATS connection
       ↓
Close DB
       ↓
Exit
```

---

# 74. Upgrade Mechanism

```text
Overlook SaaS
      │
      ▼
Version notification
      │
      ▼
Collector updater
      │
      ▼
Download signed package
      │
      ▼
Verify signature
      │
      ▼
Backup configuration
      │
      ▼
Install
      │
      ▼
Restart
      │
      ▼
Health check
```

If health check fails:

```text
Rollback
```

---

# 75. Security Controls

Minimum Collector controls:

```text
mTLS

TLS 1.3

AES-256-GCM

Certificate rotation

Encrypted local secrets

Signed Collector releases

Signed updates

RBAC

Audit logging

Tamper-resistant logs

Credential isolation

No shell execution through SaaS

Response policy validation

Replay protection

Nonce / command expiration

Rate limiting

Local firewall hardening
```

---

# 76. Main Data Flow Sequence

```text
FortiGate
   │
   │ Syslog TLS
   ▼
Collector Ingestion
   │
   ▼
NATS RAW
   │
   ▼
FortiGate Parser
   │
   ▼
Normalized Schema
   │
   ▼
Asset + Network Enrichment
   │
   ▼
Security Fact Engine
   │
   ▼
Privacy Filter
   │
   ▼
ZSTD
   │
   ▼
AES-GCM
   │
   │ mTLS
   ▼
Overlook SaaS
   │
   ▼
NATS
   │
   ├── ClickHouse
   ├── Postgres
   └── Graph Processing
```

---

# 77. AWS CSPM Flow

```text
AWS API
   │
   ▼
AWS Connector
   │
   ▼
Inventory
   │
   ▼
Normalize
   │
   ▼
Evaluate posture
   │
   ▼
Finding

"EC2 publicly exposed"
   │
   ▼
Security Fact
   │
   ▼
SaaS
   │
   ▼
CSPM
```

---

# 78. DSPM Flow

```text
Database / S3
      │
      ▼
Metadata Connector
      │
      ▼
Schema / Object Metadata
      │
      ▼
Classification
      │
      ▼
PII Finding
      │
      ▼
Security Fact
      │
      ▼
SaaS
      │
      ▼
DSPM
```

Raw customer records should not be uploaded merely to perform SaaS-side visualization.

---

# 79. ASPM Flow

```text
GitHub / GitLab
      │
      ▼
Repository Connector
      │
      ▼
Repo Metadata
      │
      ├── Branch
      ├── Dependency
      ├── Secret finding
      ├── Vulnerability
      └── Pipeline
              │
              ▼
        Security Facts
              │
              ▼
             ASPM
```

---

# 80. Network Flow

```text
Firewall
Switch
Flow Log
Cloud Network
      │
      ▼
Collector
      │
      ▼
Normalize
      │
      ▼
Network Relationships
      │
      ▼
Security Graph
```

Example:

```text
Internet
   ↓
ALB
   ↓
EC2
   ↓
Database
```

---

# 81. Unified Graph Output

The Collector should help generate relationships such as:

```text
Identity

    CAN_ASSUME

Role

    CAN_ACCESS

Application

    RUNS_ON

EC2

    CONNECTED_TO

Database

    CONTAINS

PII
```

This allows Overlook to correlate findings across CSPM, DSPM, ASPM, Network and Identity.

---

# 82. MVP Components

Collector MVP should contain:

```text
Collector identity

mTLS SaaS connection

Connector framework

AWS connector

Generic REST connector

Syslog connector

NATS JetStream

Parser framework

Normalization

Security Fact format

Privacy filtering

ZSTD compression

Encrypted forwarding

Local state DB

Health API

Basic React UI
```

---

# 83. Phase 2

```text
Auto parser

Advanced enrichment

Dead-letter management UI

Dynamic worker scaling

Agent gateway

Local credential vault improvements

Certificate auto-rotation

Advanced flow control
```

---

# 84. Phase 3

```text
Response Engine

Cloud response actions

Endpoint response actions

Local analytics

Advanced graph extraction

OT collectors

High availability

Collector clustering
```

---

# 85. What Should NOT Be in Collector V1

Avoid introducing:

```text
Kafka

Kubernetes

Redis cluster

Elasticsearch

Full ClickHouse cluster

Full PostgreSQL cluster

Large ML models

Full SIEM correlation engine

Full SaaS dashboard

20+ microservices
```

These will make deployment and troubleshooting significantly harder.

---

# 86. Recommended V1 Stack

```text
                     COLLECTOR

                    Go Binary

                       +

                NATS JetStream

                       +

                     SQLite

                       +

              React Management UI
```

Optional later:

```text
Redis

Local analytical database

ML inference
```

---

# 87. Final Collector Component Map

```text
╔════════════════════════════════════════════════════╗
║             OVERLOOK EDGE COLLECTOR               ║
║                                                    ║
║ CONNECTORS                                         ║
║ AWS | Azure | GCP | Firewall | EDR | Git | DB    ║
║ Agent | Syslog | API                               ║
║                        │                           ║
║                        ▼                           ║
║              INGESTION GATEWAY                     ║
║                        │                           ║
║                        ▼                           ║
║                NATS JETSTREAM                      ║
║                        │                           ║
║                        ▼                           ║
║                  PARSER ENGINE                     ║
║                        │                           ║
║                        ▼                           ║
║               NORMALIZATION ENGINE                 ║
║                        │                           ║
║                        ▼                           ║
║                 ENRICHMENT ENGINE                  ║
║                        │                           ║
║                        ▼                           ║
║                SECURITY FACT ENGINE                ║
║                        │                           ║
║                        ▼                           ║
║                  PRIVACY ENGINE                    ║
║                        │                           ║
║                        ▼                           ║
║                ZSTD COMPRESSION                    ║
║                        │                           ║
║                        ▼                           ║
║                AES-GCM ENCRYPTION                  ║
║                        │                           ║
║                        ▼                           ║
║                METADATA FORWARDER                  ║
║                        │                           ║
║                       mTLS                         ║
╚════════════════════════╪═══════════════════════════╝
                         │
                         ▼
                OVERLOOK SAAS PLATFORM
```

---

# 88. Final Design Decision

The Collector should **not be treated as a log-forwarding appliance**.

Its architecture should make it a:

> **Customer-side security intelligence processing node that converts raw security telemetry into normalized, privacy-minimized Security Facts, Entities and Relationships before securely forwarding those facts to the Overlook SaaS platform.**

The central object connecting all Overlook modules should therefore be:

```text
OVERLOOK SECURITY FACT
        +
ENTITY
        +
RELATIONSHIP
```

rather than the original raw log.

This provides one common foundation for:

```text
CSPM

DSPM

ASPM

Network Security

Identity Security

Attack Paths

Shadow AI

Risk Correlation

Exposure Management

Response
```

while keeping the heavy/raw customer telemetry under customer control.
# System Architecture

> Status: DRAFT
> Phase: Architect
> Input: docs/PRD.md · docs/REQUIREMENTS.md · ADR-001 through ADR-008
> Last updated: 2026-10-05

---

## 1. System Overview

The agent-security-toolkit is a governance platform for autonomous agents and Non-Human Identities (NHIs). It enforces identity, runtime authorization, pre-execution policy, behavioral anomaly detection, data loss prevention, and tamper-evident auditability — without building or hosting agents itself.

The system is organized into two strictly separated planes:

```
┌──────────────────────────────────────────────────────────────────────┐
│  MANAGEMENT PLANE  (human operators + MFA only)                      │
│                                                                      │
│   Management API (FastAPI)          CLI (Python)                     │
│   ├─ Cedar policy CRUD + versioning  ├─ Policy management            │
│   ├─ Agent identity admin            ├─ Audit log query              │
│   ├─ System configuration            ├─ Compliance export            │
│   ├─ Audit log access                └─ Agent status                 │
│   └─ Kill switch configuration                                       │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  NO agent credential crosses this boundary
┌───────────────────────────▼──────────────────────────────────────────┐
│  DATA PLANE  (agent JWTs only)                                       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │            GOVERNANCE SERVICES  (Go · gRPC + REST)           │   │
│  │                                                              │   │
│  │  Authorization    Audit         Behavioral      Alert        │   │
│  │  Service          Service       Monitoring      Service      │   │
│  │                                 Service                      │   │
│  │  Policy Sync      Session Monitor / Kill Switch              │   │
│  │  Service          Identity / Token Service                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────┐     ┌────────────────────────────────────┐ │
│  │    Python SDK       │     │         Sidecar  (Go)              │ │
│  │  (embedded in agent)│     │  (K8s / Docker deployment mode)   │ │
│  └─────────────────────┘     └────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Components

### 2.1 Python SDK
**Language:** Python 3.11+
**Package:** `agent-security` (PyPI, Apache 2.0)
**Role:** Primary developer interface; embeds in agent code; handles runtime authorization, pre-execution policy, DLP, behavioral event emission, and kill switch listener.

**Internal modules:**
| Module | Responsibility |
|---|---|
| `identity` | JWT credential management, OIDC token exchange, credential refresh |
| `cedar` | Local Cedar policy cache and in-process evaluation (`cedar-policy` package) |
| `authorizer` | Per-action authorization — Cedar evaluation + fallback to Authorization Service |
| `policy_sync` | WebSocket listener for Cedar policy push updates from Policy Sync Service |
| `dlp` | FPE tokenization (FF1/FF3-1), pattern-based masking, content inspection |
| `behavioral` | Async behavioral event emitter — fire-and-forget to Monitoring Service |
| `monitor` | Kill switch listener (SSE/WebSocket), graduated enforcement state machine |
| `audit` | Audit log writer — batched async writes to Audit Service |
| `adapters` | LangChain and OpenAI Agents SDK framework adapters |

**Cedar evaluation flow:**
```
Agent action → SDK authorizer
    → Cedar local cache hit? → evaluate in-process (<1ms) → allow/deny
    → Cache miss or session-aware decision → Authorization Service (<50ms)
    → Policy Sync Service pushes updates → cache invalidated + refreshed
```

---

### 2.2 Sidecar
**Language:** Go 1.22+
**Role:** Language-agnostic deployment mode for K8s/Docker. Runs as a container sidecar, intercepts all agent HTTP/gRPC traffic, and enforces governance without code changes in the agent.

**Capabilities:**
- Intercepts outbound agent requests via iptables/transparent proxy
- Calls Authorization Service for each intercepted action
- Applies DLP inspection and FPE tokenization on egress traffic
- Emits behavioral events to Monitoring Service
- Listens for kill switch signals from Session Monitor
- Injects governance headers (correlation ID, session ID) into forwarded requests

**Deployment:**
```yaml
# K8s pod spec (abbreviated)
containers:
  - name: agent
    image: my-agent:latest
  - name: agent-security-sidecar
    image: agent-security/sidecar:latest
    env:
      - name: GOVERNANCE_API_ENDPOINT
        value: "governance-svc:50051"
      - name: AGENT_IDENTITY_TOKEN
        valueFrom:
          secretKeyRef: ...
```

---

### 2.3 Governance Services
**Language:** Go 1.22+
**Protocol:** gRPC (primary) + REST/JSON gateway (via grpc-gateway)
**Deployment:** Separate Go binary per service, independently scalable

#### Authorization Service
Evaluates Cedar authorization requests requiring central state (session-aware decisions, cross-agent context). Responds in <50ms p99.
- Receives `AuthorizeRequest{principal, action, resource, context}`
- Fetches session state from Redis
- Evaluates against Cedar policies (authoritative copy in PostgreSQL)
- Returns `AuthorizeResponse{decision: ALLOW|DENY, reason, risk_score}`
- Writes decision to Audit Service (async)

#### Audit Service
Append-only, cryptographically hash-chained log writer.
- Accepts audit events from SDK, sidecar, Authorization Service
- Computes SHA-256 hash of `(event_payload + previous_entry_hash)`
- Writes to append-only PostgreSQL table (insert-only trigger, no UPDATE/DELETE)
- Signs each entry with the system's audit signing key (RS256)
- Exposes query API (read-only) to Management API

#### Behavioral Monitoring Service
Processes behavioral event streams and runs anomaly detection.
- Consumes behavioral events from Redis Streams
- Maintains per-agent baseline models in PostgreSQL
- Runs anomaly detection: statistical drift (>2σ), known attack patterns (MITRE ATLAS library), bulk data access (>3σ volume)
- Publishes anomaly alerts to Alert Service
- Exposes live session state to Session Monitor

#### Alert Service
Routes security events to configured incident response destinations.
- Receives alerts from Behavioral Monitoring Service and Authorization Service
- Applies routing rules (severity, event type, agent identity, resource)
- Delivers to: webhook, Splunk HEC, PagerDuty Events API, Slack, CEF/LEEF SIEM
- Retries with exponential backoff on delivery failure
- Logs all delivery attempts in audit trail

#### Policy Sync Service
Pushes Cedar policy updates to all connected SDK instances and sidecars.
- Watches PostgreSQL Cedar policy table for changes (LISTEN/NOTIFY)
- Fans out policy diffs to all connected clients via WebSocket
- Clients apply partial updates to their local Cedar cache
- Handles reconnection and full-resync on connection loss

#### Session Monitor / Kill Switch
Maintains live agent session state and enforces kill commands.
- Tracks active sessions in Redis (TTL-based, refreshed on each action)
- Exposes SSE stream to SDK and sidecar for real-time kill switch signals
- Receives kill commands from Management API
- Broadcasts graduated enforcement signals: THROTTLE → WARN → SUSPEND → TERMINATE
- On TERMINATE: publishes credential revocation to Identity/Token Service

#### Identity / Token Service
Issues and validates agent JWTs; handles OIDC federation.
- OAuth 2.0 Client Credentials grant → issues short-lived JWT (RS256, default 15min TTL)
- OIDC token exchange: accepts SPIFFE SVID, AWS IAM, GCP Workload Identity, Azure Managed Identity, Vault OIDC tokens → issues scoped system JWT
- JWKS endpoint for offline JWT verification
- Credential revocation via Redis revocation list (checked on every authorization decision)

---

### 2.4 Management API
**Language:** Python 3.11+ / FastAPI
**Auth:** Human identity + MFA (OIDC with MFA enforcement at IdP level)
**Role:** Management plane only — accepts no agent JWT credentials at any endpoint.

**Endpoints:**
| Resource | Operations |
|---|---|
| `/policies` | CRUD Cedar policies, list versions, diff versions, rollback |
| `/identities` | Register agents, revoke credentials, view identity records |
| `/config` | Read/write system configuration (thresholds, FPE key references, enforcement modes) |
| `/audit` | Query audit log, export compliance reports (OWASP/MITRE/NIST tagged) |
| `/sessions` | View live agent sessions, issue kill commands |
| `/baselines` | View behavioral baselines (read-only from management plane) |

---

### 2.5 CLI
**Language:** Python 3.11+
**Package:** `agent-security-cli` (PyPI, Apache 2.0)
**Role:** Thin wrapper over the Management API for human-facing operations.

```bash
agent-security policy list
agent-security policy apply ./policies/data-pipeline.cedar
agent-security policy rollback --policy-id p-123 --version 4
agent-security audit query --agent-id agent-xyz --range 7d
agent-security audit export --format csv --framework owasp
agent-security session list
agent-security session kill --agent-id agent-xyz --level terminate
agent-security compliance export --framework nist-ai-rmf
```

---

## 3. Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| SDK language | Python 3.11+ | ADR-003; primary agent developer language |
| Services language | Go 1.22+ | <50ms p99 latency, high concurrency, strong gRPC support |
| Management API | Python FastAPI | Consistency with SDK; less latency-sensitive than data plane |
| Policy engine | Cedar (`cedar-policy` Python + Go bindings) | ADR-006; PARC model, formally verified |
| API protocol | gRPC (primary) + REST/JSON via grpc-gateway | Performance for SDK/sidecar; REST for CLI and tooling |
| Identity / Auth | Custom OIDC server (Go) + Keycloak for enterprise IdP federation | JWT issuance + OIDC federation (ADR-005) |
| Primary database | PostgreSQL 16 | Policies, audit log, identities, baselines, sessions |
| In-memory / session | Redis 7 | Session state, kill switch signals, policy sync pub/sub, revocation list |
| Event streaming | Redis Streams | Behavioral event pipeline from SDK/sidecar to Monitoring Service |
| FPE tokenization | `pyffx` / `ff3` (Python SDK) + Go FF1/FF3-1 library (sidecar) | ADR-007; vaultless tokenization |
| Key management | HashiCorp Vault / AWS KMS / GCP KMS | ADR-005, ADR-007; FPE keys and JWT signing keys |
| Containerization | Docker + Helm chart | Standard K8s deployment |
| Framework adapters | LangChain callback hooks + OpenAI Agents SDK hooks | ADR-001 |
| Audit signing | RS256 (JWT-style signing per log entry) | FR-5.2 cryptographic tamper-evidence |

---

## 4. Data Models

### Agent Identity
```
AgentIdentity {
  agent_id:        string (UUID)
  name:            string
  declared_scope:  []string         // resource patterns allowed
  declared_purpose: string
  framework:       enum(langchain, openai_agents, ...)
  client_id:       string           // OAuth 2.0 client identifier
  client_secret_hash: string        // bcrypt hash, never stored in plain
  status:          enum(active, suspended, revoked)
  created_at:      timestamp
  created_by:      string           // human operator identity
  tenant_id:       string           // NFR-12 tenant isolation
}
```

### Cedar Policy (Versioned)
```
PolicyVersion {
  policy_id:       string (UUID)
  version:         int              // monotonically increasing
  name:            string
  cedar_source:    text             // raw Cedar policy text
  schema_version:  string           // Cedar entity schema version
  status:          enum(active, inactive, draft)
  created_at:      timestamp
  created_by:      string           // human operator (management plane only)
  change_reason:   string
  diff_from_prev:  text             // unified diff from previous version
  tenant_id:       string
}
```

### Audit Log Entry
```
AuditLogEntry {
  entry_id:        string (UUID)
  sequence_num:    int              // monotonically increasing per tenant
  timestamp:       timestamp (ns)
  session_id:      string
  agent_id:        string
  action_type:     string           // e.g. "tool_call", "api_request", "data_access"
  resource:        string           // resource identifier
  outcome:         enum(allow, deny, redact, block, tokenize)
  policy_matched:  string           // Cedar policy ID or rule name
  risk_score:      enum(low, medium, high, critical)
  owasp_category:  string           // e.g. "LLM01"
  mitre_technique: string           // e.g. "AML.T0051"
  nist_function:   string           // e.g. "DETECT"
  prev_entry_hash: string           // SHA-256 of previous entry
  entry_hash:      string           // SHA-256(payload + prev_hash)
  signature:       string           // RS256 signature of entry_hash
  tenant_id:       string
}
```

### Behavioral Baseline
```
BehavioralBaseline {
  agent_id:           string
  computed_at:        timestamp
  observation_window: duration
  action_type_freq:   map[string]float   // actions per minute by type
  resource_access_patterns: []string     // typical resource patterns
  avg_session_duration: duration
  typical_data_volume:  float            // bytes/session, p95
  known_tool_sequences: [][]string       // common action sequences
  anomaly_threshold:    float            // σ multiplier (default 2.0)
  tenant_id:            string
}
```

### Session State
```
SessionState {
  session_id:       string
  agent_id:         string
  started_at:       timestamp
  last_action_at:   timestamp
  action_count:     int
  current_action:   string
  resources_accessed: []string
  running_risk_score: enum(low, medium, high, critical)
  anomaly_flags:    []string
  enforcement_level: enum(normal, throttle, warn, suspend, terminate)
  tenant_id:        string
  ttl:              int             // Redis TTL in seconds
}
```

---

## 5. API Contracts

### Authorization Service (gRPC)
```protobuf
service AuthorizationService {
  rpc Authorize(AuthorizeRequest) returns (AuthorizeResponse);
  rpc AuthorizeBatch(AuthorizeBatchRequest) returns (AuthorizeBatchResponse);
}

message AuthorizeRequest {
  string agent_id    = 1;
  string session_id  = 2;
  string action      = 3;  // Cedar action
  string resource    = 4;  // Cedar resource
  map<string, string> context = 5;  // Cedar context attributes
}

message AuthorizeResponse {
  enum Decision { ALLOW = 0; DENY = 1; }
  Decision decision  = 1;
  string   reason    = 2;  // matched Cedar policy or denial reason
  string   risk_score = 3; // low | medium | high | critical
  string   entry_id  = 4;  // audit log entry ID for this decision
}
```

### Audit Service (gRPC)
```protobuf
service AuditService {
  rpc WriteEvent(AuditEvent) returns (WriteEventResponse);
  rpc WriteBatch(AuditBatchRequest) returns (WriteEventResponse);
}
```

### Session Monitor / Kill Switch (gRPC + SSE)
```protobuf
service SessionMonitor {
  rpc GetActiveSessions(GetSessionsRequest) returns (stream SessionState);
  rpc IssueKillCommand(KillCommand) returns (KillCommandResponse);
}

message KillCommand {
  enum Scope    { INSTANCE = 0; AGENT_TYPE = 1; SESSION = 2; TENANT = 3; }
  enum Level    { THROTTLE = 0; WARN = 1; SUSPEND = 2; TERMINATE = 3; }
  string target_id = 1;
  Scope  scope     = 2;
  Level  level     = 3;
  string reason    = 4;
  string issued_by = 5;
}
```

### SDK Interface (Python)
```python
from agent_security import GovernanceClient

client = GovernanceClient(
    agent_id="agent-xyz",
    credential_path="~/.agent-security/credentials",
    governance_endpoint="https://governance.internal:50051",
)

# Runtime authorization
decision = client.authorize(
    action="tool:database_query",
    resource="db:customers",
    context={"classification": "PII", "environment": "production"}
)
# decision.allowed → bool
# decision.risk_score → "low" | "medium" | "high" | "critical"
# decision.reason → str

# Pre-execution check (raises PolicyViolationError if blocked)
client.pre_execute(action=proposed_action, context=session_context)

# DLP — tokenize sensitive output before returning
safe_output = client.dlp.tokenize(output_text, data_types=["PII", "credentials"])

# Behavioral monitoring (fire-and-forget)
client.emit_event(action="tool_call", resource="api:payment", metadata={...})

# Full session monitoring context manager
with client.session(name="data-pipeline-run") as session:
    result = my_agent.run(task)
```

---

## 6. Cedar Integration

### Entity Schema
```cedar
entity Agent {
  scope: Set<String>,
  purpose: String,
  framework: String,
  tenant: Tenant,
};

entity Resource {
  classification: String,  // "PII" | "credentials" | "confidential" | "internal" | "public"
  owner: String,
  tenant: Tenant,
};

entity Action
  memberOf ActionGroup;

entity Tenant;
```

### Example Policies
```cedar
// Allow data pipeline agents to read non-PII resources during business hours
permit(
  principal in Agent::"data-pipeline",
  action in [Action::"read", Action::"list"],
  resource is Resource
)
when {
  resource.classification != "PII" &&
  resource.classification != "credentials" &&
  context.time_hour >= 6 &&
  context.time_hour <= 22 &&
  context.environment == "production"
};

// Deny all agents from accessing credential resources
forbid(
  principal,
  action,
  resource is Resource
)
when {
  resource.classification == "credentials" &&
  !principal.scope.contains("credentials:read")
};
```

### Hybrid Cache Architecture
```
Policy Sync Service (Go)
    │ LISTEN/NOTIFY on policy table
    │ WebSocket fan-out to all SDK instances
    ▼
SDK cedar module
    ├─ In-memory Cedar policy set (authoritative for low-risk decisions)
    ├─ Policy version tracked → stale detection
    └─ On update: partial diff applied, full resync on version gap
```

---

## 7. Security Architecture

### Key Management
| Key | Purpose | Storage | Rotation |
|---|---|---|---|
| JWT signing key (RS256) | Sign agent JWTs | Vault / KMS | 90 days |
| Audit signing key (RS256) | Sign audit log entries | Vault / KMS | 180 days |
| FPE encryption key (AES-256) | FF1/FF3-1 tokenization | Vault / KMS | v2 (FD-6) |
| Management API TLS cert | mTLS for management plane | Vault PKI | 30 days |
| Service-to-service mTLS | Inter-service auth (data plane) | SPIFFE/SPIRE | 1 hour |

### Trust Boundaries
```
Internet / Agent
    │ Agent JWT (RS256, 15min TTL)
    ▼
Data Plane API (TLS 1.3)
    │ Service identity via mTLS (SPIFFE SVID)
    ▼
Governance Services
    │ Management plane calls require human OIDC token + MFA
    ▼
Management API (separate network segment, separate auth middleware)
    │ Vault/KMS token (short-lived)
    ▼
Key Management Service
```

### Management / Data Plane Separation
- Separate Go binaries, separate network namespaces in K8s
- Management API and Governance Services share no code paths for authentication middleware
- Management API rejects all requests with `Authorization: Bearer <agent-jwt>` header — validates only human OIDC tokens with MFA claim
- No shared database credentials — separate PostgreSQL roles with row-level security per tenant and per plane

---

## 8. Deployment Model

### v1 Self-Hosted (Docker Compose — local dev)
```yaml
services:
  governance-auth:      # Authorization + Identity Service
  governance-audit:     # Audit Service
  governance-monitor:   # Behavioral Monitoring + Session Monitor
  governance-alert:     # Alert Service
  governance-policy-sync: # Policy Sync Service
  management-api:       # Management API (separate service)
  postgres:             # Primary database
  redis:                # Session state + event streaming
  vault-dev:            # Dev Vault instance for key management
```

### v1 K8s (Helm chart)
```
agent-security/
├── charts/
│   ├── governance-services/   # All data plane services
│   ├── management-api/        # Management plane (separate namespace)
│   ├── sidecar-injector/      # Mutating webhook for auto-sidecar injection
│   └── postgres-redis/        # Dependencies
├── values.yaml
└── values.production.yaml
```

### Sidecar Injection (K8s)
A mutating admission webhook automatically injects the sidecar into pods annotated with:
```yaml
annotations:
  agent-security.io/inject: "true"
  agent-security.io/agent-id: "agent-xyz"
```

---

## 9. Framework Adapter Interface

All framework adapters implement a common interface so adding new frameworks (ADR-001) requires only a new adapter — no changes to the core SDK.

```python
class AgentFrameworkAdapter(ABC):
    """Base adapter — implement for each supported framework."""

    @abstractmethod
    def intercept_pre_execution(self, action: AgentAction) -> AdapterDecision:
        """Called before the agent executes any action."""
        ...

    @abstractmethod
    def intercept_post_execution(self, action: AgentAction, result: Any) -> Any:
        """Called after execution — apply DLP on output before returning."""
        ...

    @abstractmethod
    def emit_behavioral_event(self, action: AgentAction) -> None:
        """Fire-and-forget behavioral event to Monitoring Service."""
        ...


class LangChainAdapter(AgentFrameworkAdapter):
    """Hooks into LangChain's callback system."""
    # Implements BaseCallbackHandler — zero changes to agent code

class OpenAIAgentsAdapter(AgentFrameworkAdapter):
    """Hooks into OpenAI Agents SDK tool call lifecycle."""
    # Wraps tool definitions transparently
```

---

## 10. Open Architecture Questions

Items to resolve before implementation begins:

- [ ] Should the Authorization Service embed Cedar directly (Go bindings) or call out to the Python cedar-policy service? Go bindings give lower latency but require maintaining Cedar Go wrapper.
- [ ] PostgreSQL append-only enforcement: trigger-based (simpler) or separate write-once table role (stronger)? 
- [ ] Redis Streams vs. Kafka for behavioral event pipeline — Redis Streams sufficient for v1 scale (1,000+ agents per NFR-2), revisit for v2.
- [ ] OIDC server: build minimal custom issuer (Go) or deploy Keycloak? Keycloak adds operational overhead but handles SAML federation out of the box (FR-1.3).

# Requirements

> Status: DRAFT
> Phase: BA
> Input: docs/PRD.md
> Last updated: 2026-10-04

---

## Functional Requirements

### FR-1: Agent & NHI Identity and Authentication

**FR-1.1 — Identity Registration**
- Given an autonomous agent or NHI is being onboarded,
- When an administrator registers it in the system,
- Then the system issues a scoped, cryptographically signed **JWT** (via OAuth 2.0 Client Credentials grant) bound to that agent's declared purpose, permissions, and expiry — usable as the universal baseline identity across any environment.

**FR-1.2 — Short-Lived Credentials**
- Given an agent has a registered identity,
- When it requests access to an enterprise resource,
- Then the system issues a short-lived JWT (configurable TTL, default 15 minutes) rather than a static long-lived secret; the agent is responsible for requesting a new token before expiry using its client credentials.

**FR-1.3 — Identity Federation**
- Given an enterprise uses an existing workload identity system,
- When an agent authenticates using any of the following:
  - **SPIFFE/SPIRE SVID** (X.509 or JWT-SVID)
  - **Cloud Workload Identity** — AWS IAM role token, GCP Workload Identity token, Azure Managed Identity token
  - **SAML 2.0** — via enterprise IdP (Okta, ADFS) that bridges SAML to an OIDC endpoint; the system accepts the resulting OIDC token, not raw SAML assertions
  - **HashiCorp Vault** — via Vault AppRole auth or a Vault-issued OIDC token from Vault's identity engine
- Then the system accepts the external token via OIDC token exchange and issues a scoped system JWT in return — no additional secrets or credential stores required.

**FR-1.4 — Identity Revocation**
- Given an agent identity is compromised or decommissioned,
- When an administrator revokes it,
- Then all active credentials are immediately invalidated and future authentication attempts are denied.

---

### FR-2: Runtime Authorization

**FR-2.1 — Per-Action Authorization**
- Given an authenticated agent attempts to execute an action (API call, data access, tool use),
- When the action is evaluated at runtime,
- Then the system checks the agent's current context, permissions, and policy before allowing or denying the action — not just at provisioning time.

**FR-2.2 — Context-Aware Decisions**
- Given an agent requests an action,
- When the authorization engine evaluates the request,
- Then it considers dynamic context: time of day, source environment, data sensitivity, prior actions in the session, and declared agent intent.

**FR-2.3 — Least-Privilege Enforcement**
- Given an agent's identity scope,
- When it requests access beyond its declared scope,
- Then the system denies the request and logs the violation, even if the agent holds a valid credential.

**FR-2.4 — Authorization Decision Logging**
- Given any authorization decision (allow or deny),
- When it is made,
- Then it is recorded in the audit log with full context: agent identity, action, resource, policy matched, decision, and timestamp.

---

### FR-3: Pre-Execution Policy Controls

**FR-3.1 — Policy Definition**
- Given a security or platform team,
- When they define agent behavior policies,
- Then the system provides a **Cedar**-based policy language to express rules using the PARC model (Principal = agent identity, Action = tool call / API request, Resource = data / service / system, Context = session state, time, risk score) — supporting conditions such as: allowed actions, forbidden resource patterns, rate limits, required approvals, and risk thresholds.

**FR-3.2 — Intent Evaluation**
- Given an agent is about to execute an action,
- When the pre-execution control layer evaluates the action,
- Then it submits a Cedar authorization request `(principal, action, resource, context)` to the policy engine, receives an `Allow` or `Deny` decision in <50ms, and assigns a risk score before execution proceeds.

**FR-3.3 — Action Blocking**
- Given an agent's intended action violates policy or exceeds the risk threshold,
- When the pre-execution check runs,
- Then the action is blocked and the agent receives a structured denial response with the reason.

**FR-3.4 — Human-in-the-Loop Escalation**
- Given an agent's intended action is high-risk but not explicitly forbidden,
- When the pre-execution check runs,
- Then the action is held pending human approval, the agent is notified, and an alert is sent to the designated approver.

**FR-3.5 — Policy Simulation / Dry Run**
- Given a team wants to test a new policy before enforcing it,
- When they run the system in simulation mode,
- Then it evaluates real agent actions against the new policy and reports what would have been blocked — without actually blocking anything.

---

### FR-4: Behavioral Anomaly Detection

**FR-4.1 — Behavioral Baseline**
- Given an agent has been operating in the system,
- When sufficient behavioral data is collected,
- Then the system builds a baseline model of normal behavior per agent: typical actions, access patterns, frequency, and data touched.

**FR-4.2 — Drift Detection**
- Given a behavioral baseline exists for an agent,
- When the agent's current behavior deviates significantly from baseline,
- Then the system flags the anomaly with a severity score and triggers an alert.

**FR-4.3 — Known Attack Pattern Detection**
- Given an agent's action sequence matches a known attack pattern (e.g. privilege escalation, lateral movement, data exfiltration),
- When the detection engine evaluates the sequence,
- Then it raises a high-severity alert immediately regardless of baseline.

**FR-4.4 — Anomaly Investigation View**
- Given a security engineer receives an anomaly alert,
- When they open the investigation view,
- Then they see a timeline of the agent's actions, the specific deviation that triggered the alert, and recommended remediation steps.

**FR-4.5 — Automated Response**
- Given an anomaly exceeds a configurable severity threshold,
- When the response engine triggers,
- Then the system can automatically suspend the agent's credentials pending human review.

---

### FR-5: Tamper-Evident Audit Logging

**FR-5.1 — Append-Only Log**
- Given any agent action occurs (authentication, authorization decision, execution, anomaly),
- When it is recorded,
- Then it is written to an append-only log that cannot be modified or deleted by any agent or automated process.

**FR-5.2 — Cryptographic Signing**
- Given a log entry is written,
- When it is committed,
- Then it is cryptographically signed so that any tampering with the log is detectable.

**FR-5.3 — Structured Log Format**
- Given any log entry,
- When it is written,
- Then it includes: timestamp, agent identity, action type, resource, outcome, policy matched, session ID, and a hash chain linking to the prior entry.

**FR-5.4 — Log Querying**
- Given a security engineer or auditor,
- When they query the audit log,
- Then they can filter by agent, action type, resource, time range, outcome, and anomaly flag.

**FR-5.5 — Compliance Export**
- Given an organization requires compliance reporting,
- When they export the audit log,
- Then the system produces a report with findings tagged by OWASP LLM Top 10 category, MITRE ATLAS technique ID, and NIST AI RMF function, exported in CSV and JSON formats.

---

### FR-6: Data Exfiltration Prevention and DLP

**FR-6.1 — Input Content Inspection**
- Given a user uploads a document, pastes content, or shares data with an agent,
- When the data enters the agent interaction,
- Then the system scans the input for sensitive content (PII, credentials, confidential classification markers) and flags or blocks it according to the configured ingress policy before the agent processes it.

**FR-6.2 — Output Content Inspection**
- Given an agent is about to produce an output (API response, message, file write, external API call),
- When the output is evaluated pre-transmission,
- Then the system scans the content for sensitive data patterns and applies the configured egress policy: allow, redact, or block before the output leaves the system boundary.

**FR-6.3 — Egress Policy Enforcement**
- Given a security team has defined data egress policies in Cedar,
- When an agent attempts to transmit data to an external system, another agent, or an end user,
- Then the Cedar policy engine evaluates the transmission against declared rules — specifying what data classifications can go to which destinations — and blocks or redacts the transmission if it violates policy.

**FR-6.4 — Sensitive Data Classification**
- Given a resource or data object is registered in the system,
- When it is tagged with a sensitivity classification (e.g. PII, credentials, confidential, internal),
- Then all downstream authorization decisions, egress policy evaluations, and audit log entries reference that classification — ensuring consistent enforcement across the system.

**FR-6.5 — Vaultless Tokenization as the Primary Enforcement Action**
- Given an agent output or inter-agent message contains sensitive data,
- When the egress policy enforcement layer processes the content,
- Then the system applies **Format-Preserving Encryption (FPE)** using NIST FF1/FF3-1 to tokenize sensitive values — producing tokens that are the same format and length as the original (e.g. a credit card number tokenizes to a different but valid-looking card number) — so downstream systems and agents continue to function without receiving real sensitive data.
- The encryption key is retrieved from a configured secrets manager (HashiCorp Vault, AWS KMS, or GCP KMS) — never stored in code or config.
- The same input + key always produces the same token, preserving referential integrity across joins and lookups.
- If FPE tokenization is not applicable for a given data type (e.g. unstructured free text), the system falls back to **pattern-based masking** (replace with `[TYPE]` placeholder) or **full suppression** (remove the sensitive span entirely), depending on policy configuration.
- All tokenization, masking, and suppression events are recorded in the audit log with: data type detected, enforcement action applied, destination, and session ID — but never the original sensitive value.

**FR-6.6 — Agent-to-Agent Propagation Controls**
- Given an agent receives sensitive data in one interaction,
- When it attempts to pass that data to another agent, tool, or downstream system,
- Then the system evaluates the propagation against the egress policy for the originating data classification and blocks transmission to any destination not explicitly permitted.

---

### FR-7: Policy Versioning and Rollback

**FR-7.1 — Immutable Policy Versions**
- Given a security or platform team modifies a Cedar policy,
- When the change is saved,
- Then the system creates a new immutable policy version with: version number, full policy diff, author identity, timestamp, and change reason — and the prior version remains intact and queryable.

**FR-7.2 — Policy Change Audit Log**
- Given any Cedar policy is created, modified, or deleted,
- When the change occurs,
- Then it is recorded in the audit log with the same tamper-evident guarantees as agent action logs — including author identity, before/after state, and timestamp.

**FR-7.3 — Policy Rollback**
- Given a policy change causes unintended behavior or a security incident,
- When an administrator initiates a rollback,
- Then the system restores the selected prior policy version as the active version within 5 seconds, and records the rollback event in the audit log.

**FR-7.4 — Policy Version History Query**
- Given an administrator or auditor,
- When they query the policy version history,
- Then they can view the full chronological history of changes for any policy, diff any two versions, and identify who made each change and when.

---

### FR-8: Incident Response Integration

**FR-8.1 — Configurable Alert Delivery**
- Given an anomaly, policy violation, or high-severity security event is detected,
- When the alert is triggered,
- Then the system delivers the alert to one or more configured destinations: webhook (universal), Splunk HEC, PagerDuty Events API, or Slack — based on the organization's incident response configuration.

**FR-8.2 — Structured Alert Payload**
- Given an alert is delivered to any destination,
- When it is transmitted,
- Then the payload includes: severity, event type, agent identity, action, resource, policy matched or anomaly description, session ID, timestamp, and a deep link to the audit log entry.

**FR-8.3 — SIEM Integration via CEF/LEEF**
- Given an organization uses a SIEM platform (Splunk, QRadar, ArcSight),
- When security events are forwarded,
- Then the system emits events in Common Event Format (CEF) and Log Event Extended Format (LEEF) compatible with standard SIEM ingestion pipelines.

**FR-8.4 — Alert Routing Rules**
- Given different event types require different response teams,
- When an alert is generated,
- Then routing rules determine which destination receives it based on: severity, event type, agent identity, and affected resource — so high-severity events reach on-call responders and low-severity events go to a monitoring queue.

---

### FR-9: Agent Monitoring and Kill Switch

**FR-9.1 — Real-Time Session Monitoring**
- Given one or more agents are actively executing,
- When an operator opens the monitoring view,
- Then they see a live dashboard of all active sessions updated in near real-time, showing: agent identity, current action, session duration, resources being accessed, running risk score, and any active anomaly flags.

**FR-9.2 — Manual Kill Switch**
- Given an operator determines an agent must be stopped,
- When they issue a kill command,
- Then the system immediately stops the targeted scope — selectable as: a specific agent instance, all instances of an agent type, all agents within a session or pipeline, or a tenant-wide emergency stop — independent of whether an anomaly was detected.

**FR-9.3 — In-Flight Action Termination**
- Given a kill command is issued while an agent action is currently executing,
- When the termination signal is sent,
- Then the system aborts the in-flight action, rolls back its effects where possible, records the incomplete state and abort reason in the audit log, and prevents the agent from initiating any further actions.

**FR-9.4 — Graduated Enforcement**
- Given an agent's behavior or risk score warrants intervention,
- When an enforcement action is triggered (manually or automatically),
- Then the system applies one of four graduated levels — configurable per agent identity and per Cedar policy:
  - **Throttle** — rate-limit the agent's actions without stopping execution; alert operators
  - **Warn** — flag the session and alert operators; agent continues unrestricted
  - **Suspend** — block all new actions and hold the session state; agent cannot proceed until a human releases the suspension
  - **Terminate** — full stop; abort in-flight actions, invalidate credentials, clean up session state

**FR-9.5 — Post-Kill Recovery**
- Given an agent has been suspended or terminated,
- When an operator initiates recovery,
- Then the system presents a structured recovery workflow: a summary of what the agent did, what actions were aborted, what state was left behind, and any resources that may require remediation — followed by an explicit re-authorization step that must be completed before the agent can resume execution.

---

### FR-10: System Self-Protection and Integrity

**FR-10.1 — Management / Data Plane Separation**
- Given an agent holds a valid JWT credential,
- When it attempts to call any management plane endpoint (policy management, identity administration, system configuration, audit log access, baseline management, kill switch configuration),
- Then the request is denied with a 403, the attempt is logged as a critical security event, and a high-severity alert is raised — the agent JWT grants access to data plane APIs only.

**FR-10.2 — Policy Write Protection**
- Given any request to create, modify, or delete a Cedar policy,
- When the request is received,
- Then the system accepts it only from an authenticated human identity via the management plane with MFA verified; any attempt originating from an agent credential is denied, logged, and alerted regardless of the agent's declared scope or permissions.

**FR-10.3 — Behavioral Baseline Integrity**
- Given an agent is generating behavioral data,
- When that data is used to compute or update the agent's behavioral baseline,
- Then the baseline is computed entirely by the system's internal engine from observed actions; no agent-provided input, API call, or external data source can directly modify, reset, or delete a baseline — the baseline store is write-protected from all agent credentials.

**FR-10.4 — System Configuration Immutability**
- Given any request to modify system-wide configuration (enforcement thresholds, anomaly detection rules, FPE encryption keys, kill switch settings, monitoring parameters, rate limit values),
- When the request arrives,
- Then it is accepted only from an authenticated human identity via the management plane; automated processes and agent credentials are denied unconditionally, and all configuration change attempts are logged in the tamper-evident audit trail.

**FR-10.5 — Governance API Rate Limiting and Abuse Prevention**
- Given an agent is making requests to the authorization or policy evaluation API,
- When the request rate from that agent's identity exceeds a configurable threshold,
- Then the system automatically throttles the agent, logs the abuse pattern as a high-severity anomaly, and alerts operators — preventing a compromised agent from flooding the governance service to force fail-open behavior or exhaust system resources.

**FR-10.6 — Prompt Injection Defense for Governance Endpoints**
- Given an agent's input or output contains instructions targeting governance management endpoints (e.g. injected instructions to call policy APIs, modify configurations, or disable monitoring),
- When the pre-execution inspection layer evaluates the content,
- Then the injected instructions are detected, the action is blocked, the event is logged as a critical security violation, and the agent session is escalated for human review.

---

## Compliance Requirements

> Technically feasible controls derived from OWASP LLM Top 10, MITRE ATLAS, and NIST AI RMF.
> Non-technical, organizational, and process requirements from these frameworks are out of MVP scope.

### CR-1: Verifiable, Scoped Identity Tokens
- Given an agent is registered and issued a JWT identity token,
- When the token is presented to a resource or evaluated by the authorization engine,
- Then it must be verifiable offline using the issuer's public key (RS256 or ES256), contain claims for: agent ID, declared scope, purpose, issuing system, and expiry — and must be rejected if any claim is missing or the signature is invalid.
- **Maps to:** MITRE ATLAS: Initial Access · NIST AI RMF: GOVERN

---

### CR-2: Agent Action Rate Limiting
- Given an authenticated agent is executing actions,
- When it exceeds the configured action rate for its identity scope,
- Then further actions are blocked until the rate window resets, and the event is logged as a policy violation.
- **Maps to:** OWASP LLM10: Unbounded Consumption · MITRE ATLAS: Impact

---

### CR-3: Sensitive Resource Access Control
- Given an agent requests access to a resource,
- When that resource is tagged as sensitive (PII, credentials, internal system context),
- Then access is denied unless explicitly permitted in the agent's declared scope, regardless of credential validity.
- **Maps to:** OWASP LLM02: Sensitive Information Disclosure · MITRE ATLAS: Collection

---

### CR-4: Prompt Injection Detection
- Given an agent receives an input before execution,
- When the pre-execution layer evaluates the input,
- Then it scans for instruction override patterns (e.g. "ignore previous instructions", role switching, delimiter injection) and blocks execution if detected.
- **Maps to:** OWASP LLM01: Prompt Injection · MITRE ATLAS: Execution

---

### CR-5: System Prompt Leakage Prevention
- Given an agent is about to execute an action,
- When that action would expose internal system context, instructions, or configuration to an external system or user,
- Then the action is blocked and flagged as a high-severity policy violation.
- **Maps to:** OWASP LLM07: System Prompt Leakage · MITRE ATLAS: Exfiltration

---

### CR-6: Per-Action Risk Scoring
- Given an agent's action is being evaluated pre-execution,
- When the policy engine assesses the action,
- Then it assigns a risk score (low / medium / high / critical) based on: action type, resource sensitivity classification, session context, and deviation from declared agent intent.
- **Maps to:** NIST AI RMF: MEASURE · OWASP LLM06: Excessive Agency

---

### CR-7: Reconnaissance Pattern Detection
- Given an agent is executing a sequence of actions within a session,
- When that sequence matches known reconnaissance patterns (repeated capability probing, systematic API enumeration, parameter fuzzing),
- Then the system raises a high-severity anomaly alert and logs the full sequence.
- **Maps to:** MITRE ATLAS: Reconnaissance · MITRE ATLAS: Discovery

---

### CR-8: Bulk Data Access Detection
- Given an agent is accessing data resources,
- When the volume or breadth of data retrieved within a session significantly exceeds the agent's baseline,
- Then the system flags the session as a potential exfiltration event and triggers an alert.
- **Maps to:** MITRE ATLAS: Collection · OWASP LLM02: Sensitive Information Disclosure

---

### CR-9: Privilege Escalation Detection
- Given an agent has established initial access,
- When it subsequently attempts to access resources outside its originally declared scope,
- Then the system denies the request, flags it as a privilege escalation attempt, and raises a high-severity alert.
- **Maps to:** MITRE ATLAS: Privilege Escalation · OWASP LLM06: Excessive Agency

---

### CR-10: Compliance-Tagged Audit Log Entries
- Given any security-relevant event is recorded in the audit log,
- When the event matches one or more compliance framework controls,
- Then the log entry is automatically tagged with: the applicable OWASP LLM Top 10 category, MITRE ATLAS tactic and technique ID, and NIST AI RMF function — enabling filtered compliance exports without manual mapping.
- **Maps to:** OWASP LLM Top 10 · MITRE ATLAS · NIST AI RMF: MANAGE

---

## Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Performance | Authorization decisions must complete in <50ms p99 to avoid blocking agent execution |
| NFR-2 | Scalability | Must support concurrent authorization of 1,000+ agents without degradation |
| NFR-3 | Availability | Authorization and policy enforcement services must target 99.9% uptime |
| NFR-4 | Security | All credentials issued by the system must use industry-standard cryptography (RS256, ES256) |
| NFR-5 | Security | The governance system must enforce strict separation between the management plane (human operators, MFA-authenticated) and the data plane (agent JWTs); agent credentials grant zero access to management plane APIs at any privilege level; management and data plane services must run as separate processes with independent authentication middleware so a data plane compromise cannot propagate to the management plane |
| NFR-6 | Interoperability | Must integrate with OIDC, SAML 2.0, AWS IAM, Azure AD, GCP Workload Identity, and HashiCorp Vault |
| NFR-7 | Auditability | Audit logs must be retained for a minimum of 12 months with tamper evidence intact |
| NFR-8 | Usability | A basic scan/integration must be achievable with a single command and zero custom config |
| NFR-9 | Extensibility | Policy engine (Cedar) must support custom rules and entity schemas without modifying core system code; new resource types and actions must be registerable via Cedar schema extensions |
| NFR-10 | Observability | All system components must emit structured logs and metrics consumable by standard SIEM/monitoring tools |
| NFR-11 | Resilience | When the authorization or policy service is unavailable, the system must **fail closed** by default — all agent actions are denied until service recovers; an explicit **audit-only mode** is configurable per deployment for production-critical workloads where operators accept the availability-over-security tradeoff |
| NFR-12 | Tenant Isolation | All agent identities, Cedar policies, audit logs, behavioral baselines, and FPE keys must be scoped to a single tenant; no cross-tenant data access is permitted at any layer; v1 is single-tenant (self-hosted), but the data model must support tenant partitioning from day one to enable v2 multi-tenant SaaS without a schema rewrite |

---

## Acceptance Criteria Summary

| Requirement | Acceptance Criteria |
|---|---|
| FR-1.1 | Agent registered → scoped identity issued within 2s; identity is verifiable via public key |
| FR-1.2 | Credentials expire per configured TTL; rotation occurs automatically with zero agent downtime |
| FR-1.4 | Revoked identity: all tokens invalidated within 5s; subsequent auth attempts denied with 401 |
| FR-2.1 | 100% of agent actions pass through the authorization engine; no action executes without a logged decision |
| FR-2.3 | Out-of-scope access requests denied 100% of the time; zero false negatives in test suite |
| FR-3.3 | Policy-violating actions blocked before execution in all test cases; agent receives structured denial |
| FR-3.4 | High-risk actions held within 200ms; approver notified within 30s |
| FR-4.2 | Anomaly alerts triggered for behavioral deviations >2σ from baseline |
| FR-4.3 | Known attack patterns detected with <1% false negative rate against reference attack library |
| FR-5.2 | Any modification to a log entry detectable via hash chain verification |
| FR-5.4 | Log queries over 12-month dataset return results in <5s |
| CR-1 | JWT verified offline (no network call) in <5ms using RS256/ES256; missing claims or invalid signature rejected with 401 |
| CR-2 | Actions beyond rate limit blocked immediately; rate window and limit configurable per agent identity |
| CR-4 | Prompt injection patterns detected with <2% false negative rate against OWASP LLM01 test suite |
| CR-5 | System prompt leakage attempts blocked 100% of the time in test suite; zero false negatives |
| CR-6 | Risk score assigned to 100% of actions pre-execution; score visible in audit log and denial response |
| CR-7 | Reconnaissance sequences detected within 5 actions of pattern onset; alert raised within 10s |
| CR-8 | Bulk access anomalies flagged when data volume exceeds 3σ above agent session baseline |
| CR-9 | Privilege escalation attempts denied and alerted within 200ms of detection |
| CR-10 | 100% of security events tagged with applicable OWASP / MITRE ATLAS / NIST AI RMF identifiers |
| FR-6.1 | PII and credential patterns detected in agent inputs with >95% accuracy; flagged inputs blocked or held within 100ms |
| FR-6.2 | Sensitive content in agent outputs detected before transmission; redaction or block applied before data crosses system boundary |
| FR-6.3 | 100% of agent data transmissions evaluated against Cedar egress policy; zero transmissions bypass policy evaluation |
| FR-6.5 | FPE tokens are format-identical to original values; no original sensitive value appears in output; tokenization completes in <10ms per value; all enforcement events logged |
| FR-6.6 | Agent-to-agent data propagation blocked for any destination not in the originating data classification's permitted list |
| FR-7.1 | Every policy save creates a new version with full diff and author identity; prior versions remain intact and queryable |
| FR-7.3 | Policy rollback completes within 5s; rollback event recorded in audit log with author and restored version number |
| FR-8.1 | Alerts delivered to all configured destinations within 30s of event detection |
| FR-8.3 | CEF/LEEF output validated against Splunk and QRadar ingestion requirements |
| NFR-11 | Service unavailability triggers fail-closed within 500ms; all denied actions logged; audit-only mode toggled via config without restart |
| NFR-12 | Zero cross-tenant data leakage in isolation test suite; tenant partitioning verified at identity, policy, audit log, and key layers |
| FR-9.1 | Live session dashboard refreshes within 2s of any state change; all active sessions visible with no polling required by operator |
| FR-9.2 | Kill command executed within 500ms of operator action; all targeted agents stopped regardless of current execution state |
| FR-9.3 | In-flight actions aborted within 1s of kill signal; incomplete state and abort reason recorded in audit log |
| FR-9.4 | All four enforcement levels (throttle/warn/suspend/terminate) configurable per agent identity and per Cedar policy; transitions between levels logged |
| FR-9.5 | Recovery workflow presented within 5s of operator initiating recovery; re-authorization required before agent resumes; no agent restarts without explicit human approval |
| FR-10.1 | 100% of agent JWT requests to management plane endpoints rejected with 403; zero false negatives in penetration test suite; all attempts logged as critical security events |
| FR-10.2 | Policy write operations rejected for all agent credentials in 100% of test cases; MFA verification required for every management plane policy change |
| FR-10.3 | No agent API call, payload, or injected instruction can produce a write to the baseline store; verified via dedicated attack simulation suite |
| FR-10.4 | System configuration unchanged after any agent-initiated request sequence in attack simulation; all modification attempts logged |
| FR-10.5 | Governance API abuse throttling activates within 3s of rate threshold breach; fail-closed maintained under sustained 10x normal load |
| FR-10.6 | Injected governance endpoint instructions detected and blocked in 100% of test cases from OWASP LLM01 attack library targeting management APIs |

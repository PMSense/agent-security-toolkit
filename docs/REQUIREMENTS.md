# Requirements

> Status: DRAFT
> Phase: BA
> Input: docs/PRD.md
> Last updated: 2026-09-21

---

## Functional Requirements

### FR-1: Agent & NHI Identity and Authentication

**FR-1.1 — Identity Registration**
- Given an autonomous agent or NHI is being onboarded,
- When an administrator registers it in the system,
- Then the system issues a scoped, verifiable identity (e.g. workload identity token or short-lived credential) bound to that agent's declared purpose and permissions.

**FR-1.2 — Short-Lived Credentials**
- Given an agent has a registered identity,
- When it requests access to an enterprise resource,
- Then the system issues a short-lived credential (TTL-bound) rather than a static long-lived secret, and rotates it automatically upon expiry.

**FR-1.3 — Identity Federation**
- Given an enterprise uses an existing IAM provider (OIDC, SAML, cloud IAM),
- When an agent authenticates,
- Then the system federates with the enterprise identity provider and does not require a separate credential store.

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
- Then the system provides a policy language or config format to express rules such as: allowed actions, forbidden resource patterns, rate limits, required approvals, and risk thresholds.

**FR-3.2 — Intent Evaluation**
- Given an agent is about to execute an action,
- When the pre-execution control layer evaluates the action,
- Then it assesses the action against defined policies and assigns a risk score before execution proceeds.

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
- Then the system produces a report mapped to standard frameworks (OWASP LLM Top 10, SOC 2, ISO 27001) in CSV and JSON formats.

---

## Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Performance | Authorization decisions must complete in <50ms p99 to avoid blocking agent execution |
| NFR-2 | Scalability | Must support concurrent authorization of 1,000+ agents without degradation |
| NFR-3 | Availability | Authorization and policy enforcement services must target 99.9% uptime |
| NFR-4 | Security | All credentials issued by the system must use industry-standard cryptography (RS256, ES256) |
| NFR-5 | Security | The system itself must not be a single point of compromise — agent credentials must be scoped and isolated |
| NFR-6 | Interoperability | Must integrate with OIDC, SAML 2.0, AWS IAM, Azure AD, GCP Workload Identity, and HashiCorp Vault |
| NFR-7 | Auditability | Audit logs must be retained for a minimum of 12 months with tamper evidence intact |
| NFR-8 | Usability | A basic scan/integration must be achievable with a single command and zero custom config |
| NFR-9 | Extensibility | Policy engine must support custom rules without modifying core system code |
| NFR-10 | Observability | All system components must emit structured logs and metrics consumable by standard SIEM/monitoring tools |

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

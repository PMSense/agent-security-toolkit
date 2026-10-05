# Product Requirements Document (PRD)

> Status: DRAFT
> Phase: PM
> Last updated: 2026-10-04

---

## Problem Statement

Autonomous agents and Non-Human Identities (NHIs) — including AI agents, bots, service accounts, and automated pipelines — are proliferating across enterprise environments. Unlike human users, they operate at machine speed, across multiple systems simultaneously, and often with broad permissions. Yet enterprises lack a unified system to govern them.

The core problem: **there is no standard system that governs how autonomous agents and NHIs authenticate, access enterprise systems and data, and execute actions** — with the controls, visibility, and accountability that enterprises require.

This creates five specific gaps:

1. **No runtime authorization** — Agents operate with static, pre-granted permissions rather than dynamic, context-aware authorization at the moment of action.
2. **No pre-execution policy controls** — There is no enforcement layer that evaluates whether an agent *should* take an action before it executes, based on policy, context, and risk.
3. **No behavioral anomaly detection** — Deviations from expected agent behavior (unusual access patterns, unexpected tool calls, privilege escalation) go undetected until damage is done.
4. **No tamper-evident auditability** — Agent actions are either unlogged or logged in ways that can be altered, making forensic investigation and compliance reporting unreliable.
5. **No data egress controls or DLP for agent interactions** — AI-driven agents operate with access to enterprise systems and data, and users are actively uploading and sharing sensitive information with these capabilities. Once data enters an agent interaction — whether accessed by the agent directly or provided by a user — there are no enforced controls over where it goes next. An agent with legitimate read access to customer records can still exfiltrate that data by embedding it in an API call, passing it to another agent, or including it in a response. Users sharing confidential documents with an agent have no guardrails preventing that content from being propagated beyond approved system boundaries. There is no inspection layer that examines what data is leaving the enterprise — whether through deliberate agent action, inadvertent inclusion in outputs, or user-initiated disclosure — making unauthorized data loss undetectable until after the fact.

---

## Target Users

| User | Context |
|---|---|
| **Security Engineers** | Responsible for securing AI systems; need tools to detect and remediate agent vulnerabilities |
| **AI/ML Developers** | Building agent systems; need security guardrails that integrate naturally into their workflow |
| **Product / Platform Teams** | Deploying AI-powered products; need compliance, safety, and auditability built in |
| **DevSecOps / CI Teams** | Running automated pipelines; need security checks that run automatically at every stage |

---

## Goals

1. **Govern agent identity** — Provide a standard way for autonomous agents and NHIs to authenticate to enterprise systems with verifiable, scoped identities.
2. **Enforce runtime authorization** — Make every agent action subject to dynamic, context-aware authorization at the moment of execution — not just at provisioning time.
3. **Apply pre-execution policy controls** — Evaluate agent intent against policy before any action is carried out, and block or escalate when policy is violated.
4. **Detect behavioral anomalies** — Continuously monitor agent behavior and surface deviations that indicate compromise, misconfiguration, or misuse.
5. **Guarantee tamper-evident auditability** — Produce cryptographically verifiable, immutable logs of every agent action for forensics, compliance, and accountability.
6. **Map to industry compliance frameworks** — Tag findings and audit exports against OWASP LLM Top 10, MITRE ATLAS, and NIST AI RMF so enterprises can demonstrate governance posture without manual mapping.
7. **Prevent data exfiltration and enforce data loss prevention** — Inspect data entering and leaving agent interactions, enforce egress policies over what data can cross system boundaries, and redact or block sensitive content before it is propagated beyond approved scope.
8. **Monitor and control agents in real time** — Provide live visibility into active agent sessions and a graduated kill switch so operators can throttle, suspend, or immediately terminate agents exhibiting bad behavior — with full state audit and safe recovery.
9. **Protect the governance system itself** — Enforce strict separation between the management plane (human operators) and the data plane (agents), ensuring no agent credential can ever modify policies, baselines, system configuration, or the kill switch — and that the governance system cannot be weaponized or circumvented by the agents it governs.

---

## Scope

### In Scope
- Agent and NHI identity and authentication (short-lived credentials, identity federation, workload identity)
- Runtime authorization engine (policy evaluation at point of action, not just at provisioning)
- Pre-execution policy controls (intent analysis, risk scoring, action blocking / human-in-the-loop escalation)
- Behavioral anomaly detection (baseline modeling, drift detection, alerting)
- Tamper-evident audit logging (append-only, cryptographically signed action logs)
- Integration with enterprise identity systems (IAM, OIDC, SAML, secrets managers)
- Support for **LangChain** and **OpenAI Agents SDK** (v1); AutoGen, Claude SDK, CrewAI via adapter pattern in v2
- Compliance mapping to OWASP LLM Top 10, MITRE ATLAS, and NIST AI RMF
- Data egress controls — input and output content inspection for sensitive data (PII, credentials, confidential IP)
- Egress policy enforcement via Cedar — define what data can leave, to which destinations, in what form
- Sensitive data classification tagging on resources and agent-produced outputs
- Vaultless Format-Preserving Encryption (FPE/FF1/FF3-1) tokenization — tokens are same format and length as original values; key managed via secrets manager; pattern-based masking and suppression as fallback for unstructured text
- Agent-to-agent data propagation controls — prevent sensitive data received in one agent interaction from being passed to unauthorized agents or systems
- Fail-safe behavior — fail closed by default when the governance service is unavailable; configurable audit-only mode for production-critical workloads
- Policy versioning and rollback — immutable Cedar policy version history with diff, author, timestamp, and one-click rollback
- Incident response integration — configurable alert delivery to webhook, Splunk HEC, PagerDuty, Slack, and generic SIEM (CEF/LEEF)
- Single-tenant (self-hosted) for v1; multi-tenant SaaS isolation architecture designed into the data model from day one for v2 enterprise tier
- Real-time agent session monitoring — live view of active sessions, current actions, running risk scores, and anomaly flags
- System self-protection — strict management/data plane separation; agent credentials grant zero access to policy management, identity admin, system configuration, baselines, or kill switch settings; governance API rate limiting and abuse prevention; prompt injection defense for governance endpoints
- Manual kill switch — operator-initiated immediate stop at instance, agent-type, session/pipeline, or tenant-wide scope
- In-flight action termination — abort currently executing actions, not just block future ones
- Graduated enforcement — throttle → warn → suspend → terminate, configurable per agent and per policy
- Post-kill recovery workflow — audit of aborted state, cleanup, and explicit re-authorization before agent resumes

### Out of Scope
- Building or hosting AI agents (this governs agents, it does not build them)
- General-purpose SIEM or observability platform
- Non-agent/NHI identity management (human IAM is covered by existing enterprise tools)

---

## Success Metrics

| Metric | Target |
|---|---|
| Vulnerability detection rate | Catches >90% of known agent attack vectors in test suite |
| Developer adoption friction | Runnable with a single command, zero config required for basic scan |
| CI integration | Integrates with GitHub Actions, GitLab CI in <30 min |
| Audit coverage | Generates complete audit trail for every agent action scanned |
| Community traction | 500+ GitHub stars within 6 months of launch |
| Data loss prevention | >95% detection rate for PII and credential patterns in agent inputs and outputs |

---

## Open Questions

- [x] **Which AI agent frameworks should be supported first?**
  **Decision:** Launch with **LangChain** and **OpenAI Agents SDK** — they cover the largest share of enterprise agent deployments today. The system will be built around a clean adapter interface so remaining frameworks can be added in v2 without redesigning the core.
  **v2 backlog:** AutoGen, Claude SDK, CrewAI (in that order — AutoGen has strong Microsoft/enterprise backing; Claude SDK is growing with Anthropic enterprise adoption; CrewAI has smaller enterprise footprint today).

- [x] **Should access control be policy-as-code (e.g. OPA/Rego) or a simpler config file format?**
  **Decision (updated):** Use **Cedar** as the policy engine from v1. Cedar is purpose-built for authorization decisions, uses the PARC model (Principal, Action, Resource, Context) that maps directly to agent governance, is formally verified, type-safe, and simpler to write than OPA/Rego. Eliminates the YAML → OPA migration path from the original decision. See ADR-006.
- [x] **What is the recommended form factor — CLI, Python SDK, or both?**
  **Decision:** Ship v1 with a **Python SDK** as the core (required for runtime authorization, pre-execution policy enforcement, and behavioral monitoring). Follow with a **CLI** as a thin wrapper over the same library for policy management, audit querying, and CI/CD integration. The CLI is ~1 additional week of effort and serves security engineers and DevSecOps who won't embed Python.
- [x] **Are there compliance frameworks (SOC2, OWASP LLM Top 10) we should explicitly map findings to?**
  **Decision:** Map v1 findings and audit exports to **OWASP LLM Top 10**, **MITRE ATLAS**, and **NIST AI RMF** — the three most technically relevant frameworks for this system. SOC 2 and ISO 27001 are deferred to the enterprise tier (v2). 10 specific technical controls (CR-1 through CR-10) derived from these frameworks have been added to REQUIREMENTS.md.
- [x] **What is the licensing model — open source, commercial, or dual?**
  **Decision:** **Dual licensing (Open Core).** Core SDK and CLI are open source (Apache 2.0) — free to use, self-host, and contribute to. Enterprise tier is commercial — adds managed hosting, centralized audit logging with compliance export, advanced policy controls, all framework adapters, and SLA-backed support. Open core drives developer adoption and creates the top-of-funnel pipeline for enterprise deals.

---

## Future Deliverables

Items identified during requirements review that are out of v1 scope but must be addressed in future releases.

| # | Item | Target | Notes |
|---|---|---|---|
| FD-1 | FPE key scoping (per-classification, per-tenant, per-agent) | v2 | Key granularity affects blast radius of a compromised key; v1 uses a single key per deployment |
| FD-2 | SDK total latency overhead NFR | v2 | Define max acceptable end-to-end latency tax the SDK adds to agent execution; blocked on real-world v1 benchmarks |
| FD-3 | Agent versioning — re-registration, baseline reset, history preservation on agent update | v2 | Requires agent lifecycle management design; v1 treats each registration as independent |
| FD-4 | Human-in-the-loop alert delivery channels (email, Slack, PagerDuty, webhook) | v1.1 | FR-3.4 currently specifies alerting without delivery mechanism; configurable channels needed before enterprise adoption |
| FD-5 | Multi-tenant SaaS isolation (per-tenant keys, data partitioning, tenant admin) | v2 enterprise | Data model designed for tenant isolation in v1 but full SaaS multi-tenancy ships with enterprise tier |
| FD-6 | FPE key rotation with re-tokenization | v2 | Key rotation invalidates existing tokens without re-tokenization; requires coordinated migration tooling |
| FD-7 | NER-based detection for free text tokenization | v2 | v1 covers structured data types only; unstructured free text requires Named Entity Recognition |
| FD-8 | SOC 2 Type II and ISO 27001 compliance mapping | v2 enterprise | Deferred from compliance framework decision; required for enterprise procurement |

# Product Requirements Document (PRD)

> Status: DRAFT
> Phase: PM
> Last updated: 2026-09-27

---

## Problem Statement

Autonomous agents and Non-Human Identities (NHIs) — including AI agents, bots, service accounts, and automated pipelines — are proliferating across enterprise environments. Unlike human users, they operate at machine speed, across multiple systems simultaneously, and often with broad permissions. Yet enterprises lack a unified system to govern them.

The core problem: **there is no standard system that governs how autonomous agents and NHIs authenticate, access enterprise systems and data, and execute actions** — with the controls, visibility, and accountability that enterprises require.

This creates four specific gaps:

1. **No runtime authorization** — Agents operate with static, pre-granted permissions rather than dynamic, context-aware authorization at the moment of action.
2. **No pre-execution policy controls** — There is no enforcement layer that evaluates whether an agent *should* take an action before it executes, based on policy, context, and risk.
3. **No behavioral anomaly detection** — Deviations from expected agent behavior (unusual access patterns, unexpected tool calls, privilege escalation) go undetected until damage is done.
4. **No tamper-evident auditability** — Agent actions are either unlogged or logged in ways that can be altered, making forensic investigation and compliance reporting unreliable.

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

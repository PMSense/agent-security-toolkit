# Architecture Decision Records (ADRs)

> Log of key decisions made during the project, with context and rationale.

## ADR-001: Initial Agent Framework Support

- **Date:** 2026-09-21
- **Status:** Accepted
- **Context:** The system needs to integrate with AI agent frameworks to intercept and govern agent actions. Five frameworks were evaluated: LangChain, OpenAI Agents SDK, AutoGen, Claude SDK, CrewAI. Supporting all five at launch would delay time-to-market and risk shallow integrations before the adapter pattern is proven.
- **Decision:** Launch with **LangChain** and **OpenAI Agents SDK**. Build the integration layer around a clean adapter interface from day one. Remaining frameworks (AutoGen → Claude SDK → CrewAI) are queued for v2 in order of enterprise footprint.
- **Consequences:** Faster v1 launch with deeper integrations. Enterprises on AutoGen, Claude SDK, or CrewAI cannot use the system until v2. Adapter pattern adds upfront design cost but prevents core rewrites as new frameworks are added.

---

## ADR-002: Policy Engine — Simple Config for v1, OPA/Rego for v2

- **Date:** 2026-09-21
- **Status:** Superseded by ADR-006
- **Context:** The policy engine needs to evaluate agent actions against access control rules. Two options were evaluated: OPA/Rego (expressive, industry-standard, but +1.5-2 weeks delay, steep learning curve, operational overhead) and a simple YAML/JSON config format (low barrier, fast to ship, sufficient for 80% of use cases). A layered approach (config compiling to Rego) is the long-term target but adds +3-4 weeks to v1.
- **Decision:** Ship v1 with a **simple YAML/JSON config format**. Design the schema to map cleanly to Rego from the start so the v2 transpiler (config → Rego) doesn't require breaking changes. OPA/Rego becomes the advanced layer in v2.
- **Consequences:** Superseded — Cedar (ADR-006) eliminates the YAML → OPA migration path entirely.

---

## ADR-006: Policy Engine — Cedar

- **Date:** 2026-10-03
- **Status:** Accepted (supersedes ADR-002)
- **Context:** ADR-002 chose simple YAML/JSON config for v1 with a planned OPA/Rego migration in v2 via a transpiler. This required maintaining two engines and a complex migration path. Cedar was evaluated as an alternative: it is purpose-built for authorization decisions, uses the PARC model (Principal, Action, Resource, Context) that maps directly to agent governance, is formally verified (policies are mathematically provable), type-safe, fast (<5ms evaluation), and open source (Apache 2.0). Cedar is the engine behind Amazon Verified Permissions. It is more expressive than YAML but simpler to write than Rego.
- **Decision:** Use **Cedar** as the policy engine from v1. Policies are written in Cedar's native language. The SDK calls Cedar's evaluation engine with `(principal, action, resource, context)` tuples. The YAML/JSON config path and OPA/Rego migration are dropped entirely.
- **Consequences:** Single policy engine for the lifetime of the product — no v2 migration. Cedar has a learning curve (small compared to Rego). Enterprises on AWS can use Amazon Verified Permissions as a managed Cedar policy store. Cedar's formal verification is a differentiator for a security-critical system. Python SDK uses the `cedar-policy` PyPI package.

---

## ADR-003: Form Factor — Python SDK Core, CLI Fast Follow

- **Date:** 2026-09-22
- **Status:** Accepted
- **Context:** Three form factors were evaluated: CLI only, Python SDK only, and both. Runtime authorization, pre-execution policy enforcement, and behavioral monitoring require in-process integration — a CLI cannot intercept agent actions mid-execution. CLI-only misses 3 of 5 core requirements. SDK-only leaves security engineers and DevSecOps without a usable interface. Both together serve all four user types with ~1 week of additional effort since the CLI is a thin wrapper over the shared core library.
- **Decision:** **Python SDK** as the v1 core. **CLI** ships as a fast follow (same sprint or next), built as a thin wrapper over the same underlying library.
- **Consequences:** All runtime governance features require developers to embed the SDK. Security engineers and ops teams get CLI access for audit/management without writing Python. Shared core library means no duplicated logic between the two interfaces.

---

## ADR-004: Licensing Model — Dual (Open Core)

- **Date:** 2026-09-22
- **Status:** Accepted
- **Context:** Three models were evaluated: open source (fast adoption, indirect revenue), commercial (direct revenue, slow adoption), and dual/open core (fast adoption + direct enterprise revenue). Target users include developers (want free, low-friction tools) and enterprises (need compliance, SLAs, managed hosting). The security tooling space defaults to open-source trust signals — pure commercial would slow adoption significantly.
- **Decision:** **Dual licensing (Open Core).** Core SDK and CLI under Apache 2.0. Enterprise tier (managed hosting, centralized audit + compliance export, advanced policy controls, all framework adapters, SLA support) is commercial.
- **Consequences:** Open core creates developer adoption flywheel that feeds enterprise pipeline. Core can be forked, but enterprise features stay proprietary. Requires discipline to keep the right features in each tier — too much in the commercial tier kills adoption; too little leaves revenue on the table.

---

## ADR-005: Short-Lived Agent Identity — JWT Baseline + Federation Layer

- **Date:** 2026-10-02
- **Status:** Accepted
- **Context:** Three mechanisms were evaluated for short-lived agent/NHI identity: JWT via OAuth 2.0 Client Credentials (universal, low complexity, developer-friendly), SPIFFE/SPIRE SVIDs (purpose-built for NHI, auto-rotating, higher infrastructure cost), and Cloud Workload Identity (AWS/GCP/Azure — cloud-native, zero secrets, but environment-specific). No single mechanism covers all deployment scenarios.
- **Decision:** **Two-tier approach.** JWT via OAuth 2.0 Client Credentials is the universal baseline — SDK issues JWTs by default, works in any environment, TTL configurable (default 15 min, RS256/ES256 signed). SPIFFE/SPIRE SVIDs and Cloud Workload Identity tokens (AWS IAM, GCP Workload Identity, Azure Managed Identity) are accepted as federation sources via OIDC token exchange — the system validates the external token and issues a scoped system JWT in return. No long-lived secrets stored anywhere in the federation path.
- **Consequences:** Developers get started immediately with JWT; enterprises with existing SPIRE or cloud IAM infrastructure need no new credential stores. Agent is responsible for refreshing JWTs before expiry (no auto-rotation in the baseline tier — SPIFFE handles auto-rotation natively in the federation path). JWT claim schema must be stable — breaking changes require a versioned migration.

---

## ADR-007: Data Enforcement Action — Vaultless FPE Tokenization

- **Date:** 2026-10-03
- **Status:** Accepted
- **Context:** FR-6.5 required a data enforcement action for sensitive content detected in agent outputs and inter-agent messages. Options evaluated: (1) classic redaction/masking — simple but breaks downstream systems that expect valid data formats; (2) traditional vault-based tokenization — referential integrity preserved but introduces vault infrastructure, scalability bottleneck, and a new attack surface; (3) vaultless Format-Preserving Encryption (FPE) using NIST FF1/FF3-1 — no vault required, tokens are same format/length as originals, deterministic (same input + key = same token), reversible by authorized systems, key managed via existing secrets manager.
- **Decision:** **Vaultless FPE tokenization** (FF1/FF3-1) as the primary enforcement action for structured sensitive data (PII, credentials, card numbers, SSNs, emails, phone numbers, dates, API keys). Encryption key stored in secrets manager (HashiCorp Vault, AWS KMS, or GCP KMS). Pattern-based masking and full suppression are fallbacks for unstructured free text where FPE is not applicable. v1 covers structured data types; NER-based detection for free text spans is v2.
- **Consequences:** Agents and downstream systems process tokens that look like real data — no breakage. Exfiltrated tokens are useless without the encryption key. Key rotation requires re-tokenization (handled in v2). Referential integrity maintained across joins/lookups via deterministic token generation. Key management becomes the critical security dependency — must be integrated with a secrets manager from day one.

---

## Template
### ADR-XXX: [Decision Title]
- **Date:** YYYY-MM-DD
- **Status:** Proposed | Accepted | Deprecated
- **Context:** _Why was this decision needed?_
- **Decision:** _What was decided?_
- **Consequences:** _What are the tradeoffs?_

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
- **Status:** Accepted
- **Context:** The policy engine needs to evaluate agent actions against access control rules. Two options were evaluated: OPA/Rego (expressive, industry-standard, but +1.5-2 weeks delay, steep learning curve, operational overhead) and a simple YAML/JSON config format (low barrier, fast to ship, sufficient for 80% of use cases). A layered approach (config compiling to Rego) is the long-term target but adds +3-4 weeks to v1.
- **Decision:** Ship v1 with a **simple YAML/JSON config format**. Design the schema to map cleanly to Rego from the start so the v2 transpiler (config → Rego) doesn't require breaking changes. OPA/Rego becomes the advanced layer in v2.
- **Consequences:** Faster v1 launch. Enterprises needing complex conditional policies (multi-attribute, composable rules) must wait for v2. Config schema design is critical — poor schema design in v1 will make the v2 OPA upgrade harder.

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

## Template
### ADR-XXX: [Decision Title]
- **Date:** YYYY-MM-DD
- **Status:** Proposed | Accepted | Deprecated
- **Context:** _Why was this decision needed?_
- **Decision:** _What was decided?_
- **Consequences:** _What are the tradeoffs?_

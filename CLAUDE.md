# agent-security-toolkit

## Project Overview
Security toolkit for AI agents — covers the full SDLC from intent to deployment.

## Roles & Docs
| Role | Input | Output |
|---|---|---|
| Product Manager | Goal / intent | docs/PRD.md |
| Business Analyst | docs/PRD.md | docs/REQUIREMENTS.md |
| Architect | docs/REQUIREMENTS.md | docs/ARCHITECTURE.md |
| Developer | docs/ARCHITECTURE.md | src/ |
| QA Engineer | docs/REQUIREMENTS.md + src/ | tests/ + docs/TEST_PLAN.md |

## Conventions
- Language: TBD (update once stack is chosen)
- All docs live in `docs/`
- Source code lives in `src/`
- Tests live in `tests/`
- Use clear, descriptive commit messages per phase (e.g. `[PM] Add PRD`, `[DEV] Implement auth module`)

## Commands
- Run tests: TBD
- Build: TBD
- Lint: TBD

## Phase Workflow
1. Define intent → docs/PRD.md
2. Generate requirements → docs/REQUIREMENTS.md
3. Design system → docs/ARCHITECTURE.md
4. Implement → src/
5. Test → tests/ + docs/TEST_PLAN.md
6. Review → /code-review, /security-review
7. Deploy → CI/CD config

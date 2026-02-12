# ai-mortgage-quickstart

> **AI-powered mortgage loan processing reference implementation for the Red Hat AI Quickstart template, demonstrating multi-agent orchestration patterns for regulated industries.**

## Project Context

| Attribute | Value |
|-----------|-------|
| Maturity | `mvp` |
| Domain | Fintech — Mortgage Lending |
| Primary Users | AI Developers / Solutions Architects, Loan Officers, Compliance Officers |
| Compliance | Fair lending (ECOA, Fair Housing Act) — reference patterns, not production-certified |

### Maturity Expectations

**Important:** Maturity level governs **implementation quality** — test coverage, error handling depth, documentation thoroughness, infrastructure complexity. It does **not** govern **workflow phases**. An MVP still follows the full plan-review-build-verify sequence when SDD criteria are met (see `workflow-patterns` skill). The artifacts may be lighter, but they are not skipped.

| Concern | MVP |
|---------|-----|
| Testing | Happy path + critical edges |
| Error handling | Basic error responses |
| Security | Auth + input validation |
| Documentation | README + API basics |
| Performance | Profile obvious bottlenecks |
| Code review | Light review |
| Infrastructure | Basic CI + single deploy target |

## Goals

1. Demonstrate multi-agent AI patterns (supervisor-worker orchestration, human-in-the-loop workflows, confidence-based escalation) for regulated industries using LangGraph
2. Provide a self-contained developer quickstart for the Red Hat AI Quickstart template, deployable locally in under 30 minutes with `make setup && make dev`
3. Show compliance-first design with complete, immutable audit trails and explainable AI reasoning for every agent decision
4. Serve as a teaching tool with production-quality code patterns for mortgage lending — not a production loan origination system

## Non-Goals

- Production regulatory certification — demonstrates patterns, not certified for real lending
- End-user authentication — no user registration, password management, or OAuth; API key auth only
- Real credit bureau integration — mocked with synthetic data
- Payment processing — application lifecycle ends at approval/denial
- Mobile application — web only, desktop/tablet
- Multi-tenancy
- Custom ML model training or fine-tuning — uses off-the-shelf LLMs via API
- Real-time collaboration — UI polls for updates, no WebSockets for app status
- Internationalization — English only, US mortgage regulations only
- High-availability deployment — basic deployment for demo/dev

## Constraints

- Red Hat ecosystem required: OpenShift for deployment, Podman for containers, Helm for orchestration
- Must extend the existing Red Hat AI Quickstart template (Turborepo monorepo with React 19, FastAPI, PostgreSQL, Helm charts)
- Agent orchestration must use LangGraph with persistent checkpointing (PostgresSaver)
- Hybrid LLM strategy: Claude for reasoning, GPT-4 Vision for document analysis, optional LlamaStack for local/data-residency
- PostgreSQL + pgvector as single database for application data and RAG embeddings (no separate vector DB)
- Self-contained quickstart: `make setup && make dev` must get to a working system
- Complete audit trails: every agent decision, human action, and workflow transition recorded immutably

## Stakeholder Preferences

| Preference Area | Observed Pattern |
|-----------------|-----------------|
| Security posture | Upgrade, don't defer — real API key auth from day one, image redaction before LLM calls, separate DB roles from Phase 1, global rate limits before public access |
| Feature richness | Prefers impressive over minimal — include fraud detection, denial coaching, PDF metadata examination, sentiment analysis. More agents and richer demos preferred. |
| Scope decisions | Feature-rich but scoped — adds extras within the agent/demo domain (e.g., cross-session context, expanded FRED series, BatchData integration) |
| Risk tolerance | Industry-standard patterns preferred — use well-known approaches (e.g., confidence threshold locking with audit trail) over simpler custom alternatives |
| Permission model | Three roles (loan_officer, senior_underwriter, reviewer) — explicitly chose meaningful permission hierarchy over simpler two-role model |
| Conflict resolution | All agent conflicts escalate to human review — no automated tie-breaking allowed |
| Communication style | Detailed product brief provided up front; expects agents to extract and follow requirements without re-asking |

## Red Hat AI Compliance

All AI-assisted work in this project must comply with Red Hat's internal AI policies. The full machine-enforceable rules are in `.claude/rules/ai-compliance.md`. Summary of obligations:

1. **Human-in-the-Loop** — All AI-generated code must be reviewed, tested, and validated by a human before merge
2. **Sensitive Data Prohibition** — Never input confidential data, PII, credentials, or internal hostnames into AI tools
3. **AI Marking** — Include `// This project was developed with assistance from AI tools.` (or language equivalent) at the top of AI-assisted files, and use `Assisted-by:` / `Generated-by:` commit trailers
4. **Copyright & Licensing** — Verify generated code doesn't reproduce copyrighted implementations; all dependencies must use [Fedora Allowed Licenses](https://docs.fedoraproject.org/en-US/legal/allowed-licenses/)
5. **Upstream Contributions** — Check upstream project AI policies before contributing AI-generated code; default to disclosure
6. **Security Review** — Treat AI-generated code with the same or higher scrutiny as human-written code, especially for auth, crypto, and input handling

See `docs/ai-compliance-checklist.md` for the developer quick-reference checklist.

## Key Decisions

- **Language:** TypeScript 5.x (frontend), Python 3.11+ (backend)
- **Backend:** FastAPI (async)
- **Frontend:** React 19 + Vite + TanStack Router + TanStack Query + Tailwind CSS + shadcn/ui
- **Database:** PostgreSQL + pgvector (application data + RAG embeddings)
- **ORM:** SQLAlchemy 2.0 (async) + Alembic migrations
- **Testing:** Vitest + Playwright (frontend), Pytest (backend)
- **Package Manager:** pnpm (frontend), uv (backend)
- **Build System:** Turborepo
- **Agent Framework:** LangGraph + LangChain (orchestration, state management, checkpointing)
- **LLM Observability:** LangFuse
- **Caching:** Redis (RAG queries, sessions, external API responses, rate limiting)
- **Object Storage:** MinIO (S3-compatible, document uploads)
- **Containers:** Podman
- **Deployment:** Helm charts on OpenShift

---

## Agent System

This project uses a multi-agent system with specialized Claude Code agents. The main session handles routing and orchestration using the routing matrix in `.claude/CLAUDE.md`. Each agent has a defined role, model tier, and tool set optimized for its task.

### Quick Reference — "I need to..."

| Need | Agent | Command |
|------|-------|---------|
| Plan a feature or large task | **Main session** | Describe what you need; routing matrix and workflow-patterns skill guide orchestration |
| Shape a product idea into a plan | **Product Manager** | `@product-manager` |
| Gather requirements | **Requirements Analyst** | `@requirements-analyst` |
| Design system architecture | **Architect** | `@architect` |
| Design feature-level implementation approach | **Tech Lead** | `@tech-lead` |
| Break work into epics & stories | **Project Manager** | `@project-manager` |
| Write backend/API code | **Backend Developer** | `@backend-developer` |
| Build UI components | **Frontend Developer** | `@frontend-developer` |
| Design database schema | **Database Engineer** | `@database-engineer` |
| Design API contracts | **API Designer** | `@api-designer` |
| Review code quality | **Code Reviewer** | `@code-reviewer` |
| Write or fix tests | **Test Engineer** | `@test-engineer` |
| Audit security | **Security Engineer** | `@security-engineer` |
| Optimize performance | **Performance Engineer** | `@performance-engineer` |
| Set up CI/CD or infra | **DevOps Engineer** | `@devops-engineer` |
| Define SLOs & incident response | **SRE Engineer** | `@sre-engineer` |
| Debug a problem | **Debug Specialist** | `@debug-specialist` |
| Write documentation | **Technical Writer** | `@technical-writer` |

### How It Works

1. **Describe what you need** — for non-trivial tasks, the main session uses the routing matrix and workflow-patterns skill to select agents and sequence work.
2. **Use a specialist directly** when you know exactly which agent you need (e.g., `@backend-developer`).
3. **Rules files** enforce project conventions automatically — global rules are imported below, and path-scoped rules (API, UI, database development) load automatically for matching files.
4. **Spec-Driven Development** is the default for non-trivial features — plan review before code review, machine-verifiable exit conditions, and anti-rubber-stamping governance.
5. **Skills** provide workflow templates and project convention references.

## Project Conventions

@.claude/rules/ai-compliance.md
@.claude/rules/code-style.md
@.claude/rules/git-workflow.md
@.claude/rules/testing.md
@.claude/rules/security.md
@.claude/rules/error-handling.md
@.claude/rules/observability.md
@.claude/rules/api-conventions.md
@.claude/rules/agent-workflow.md
@.claude/rules/review-governance.md
@.claude/rules/architecture.md
@.claude/rules/domain.md

## Project Commands

```bash
make setup              # Install all dependencies (Node + Python)
make build              # Build all packages
make dev                # Start all dev servers (UI + API)
make test               # Run tests across all packages
make lint               # Run linters across all packages
pnpm type-check         # TypeScript type checking
make db-start           # Start database + Redis + MinIO containers
make db-upgrade         # Run database migrations (Alembic)
make containers-build   # Build all container images
make containers-up      # Start all services via compose
```

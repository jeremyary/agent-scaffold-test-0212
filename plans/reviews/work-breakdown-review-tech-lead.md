<!-- This project was developed with assistance from AI tools. -->

# Work Breakdown Review: Tech Lead

**Reviewer:** Tech Lead
**Document:** plans/work-breakdown.md v1.0
**Date:** 2026-02-12
**Verdict:** APPROVE WITH CONDITIONS

## Summary

The work breakdown is well-structured, with clean epic boundaries and generally faithful mapping of TD work units to stories. However, there are several dependency chain errors that would cause build failures if followed as-written, a handful of effort estimates that undercount complexity, and some exit conditions that test existence rather than behavior. The conditions below must be addressed before sprint execution begins -- most are dependency corrections and estimate adjustments, not structural rework.

## Findings

### TL-01: TD Dependency Graph Not Faithfully Mapped for Phase 1 API Core
**Severity:** Critical
**Location:** S-P1-08, S-P1-09, S-P1-10

**Finding:** The work breakdown lists S-P1-08 (Settings and DI) as depending on S-P1-06 and S-P1-07 (both repository WUs). The TD's dependency graph (lines 670-695) shows a different structure:

- P1-WU07 (Settings/DI, mapped to S-P1-08) depends on **P1-WU06a** (core repos), and is listed alongside P1-WU08 (Health) as parallel after P1-WU06a.
- P1-WU09 (Correlation ID, mapped to S-P1-10) depends on **P1-WU06a**, not on P1-WU08 (Health).
- The TD shows P1-WU07, P1-WU08, and P1-WU09 as three branches stemming from P1-WU06a, with P1-WU12 (CRUD routes) depending on P1-WU09.

The WB linearizes these into a strict chain S-P1-08 -> S-P1-09 -> S-P1-10, which is overly sequential. More importantly, S-P1-08 (Settings) depending on S-P1-07 (Supporting Repos) creates a false dependency -- Settings needs the repository _types_ for DI wiring, but those types come from P1-WU06a (core repos) where `__init__.py` is created. Supporting repos (P1-WU06b) can run in parallel.

**Recommendation:** Correct the dependency graph:
- S-P1-08 depends on S-P1-06 only (not S-P1-07).
- S-P1-09 (Health) and S-P1-10 (Correlation ID) can both depend on S-P1-08 and run in parallel.
- S-P1-07 (Supporting Repos) can run in parallel with S-P1-08 and S-P1-09.
- This unlocks parallelism in Sprint 2 and shortens the critical path.

---

### TL-02: Missing Dependency -- S-P1-10 Does Not Depend on Auth Middleware
**Severity:** Warning
**Location:** S-P1-10 (Correlation ID), Sprint 2 Parallelization Map

**Finding:** The TD dependency graph shows P1-WU10 (Auth, mapped to S-P1-11) depending on **P1-WU05 (Seed Data)**, because auth middleware needs seeded API keys to test against. The WB has S-P1-11 (Auth) depending on S-P1-10 (Correlation ID), which is correct, but is missing the dependency on S-P1-05 (Seed Data).

In Sprint 2, S-P1-11 is listed after the Sprint 1 stories complete, so in practice the seed data would exist. However, the story-level dependency is not declared, meaning a dependency-aware scheduler could theoretically start S-P1-11 before seed data exists.

**Recommendation:** Add S-P1-05 as an explicit dependency of S-P1-11 (Authentication Middleware). This is a correctness fix even if sprint boundaries mask it.

---

### TL-03: S-P2-01 Depends on S-P1-19, But TD Shows It Depending on P1-WU18
**Severity:** Warning
**Location:** S-P2-01 (PII Redaction Service)

**Finding:** The WB lists S-P2-01 as depending on S-P1-19 (Phase 1 Integration Test). The TD's Phase 2 dependency graph starts P2-WU01 at the top with no explicit cross-phase dependency, but it implicitly depends on Phase 1 completion. However, S-P1-20 (Developer Documentation) is the _last_ Phase 1 story, and S-P2-01 has no technical dependency on documentation.

The more accurate dependency is on S-P1-19 (Integration Test, mapped from P1-WU18), which is the last _code_ story in Phase 1 and creates `main.py` (the app factory). PII redaction depends on the API being runnable, not on the README.

**Recommendation:** Change S-P2-01 dependency from S-P1-19 to S-P1-19 (the integration test), which it already is. However, the WB Sprint 3 includes S-P1-20 _after_ S-P1-19, and S-P2-01 depends on S-P1-19, which is correct. The finding here is that the _critical path_ (Section 4) shows S-P1-18 -> S-P2-01, skipping S-P1-19 (integration test) and S-P1-20 entirely. The critical path should include S-P1-19 (integration test) before S-P2-01, since S-P2-01's stated dependency is on S-P1-19. The critical path appears to incorrectly reference S-P1-18 (Application Submit Endpoint) directly flowing to S-P2-01.

Wait -- on re-reading the critical path: `S-P1-18 (1h) -> S-P2-01 (1.5h)`. But S-P1-18 maps to P1-WU17 (Submit Endpoint), and S-P1-19 maps to P1-WU18 (Integration Test). The integration test (S-P1-19) is the gating story for Phase 2. So the critical path is missing one story. Corrected critical path should include S-P1-19 between S-P1-18 and S-P2-01, adding 1.5h to the critical path total.

**Recommendation:** Insert S-P1-19 (1.5h) into the critical path between S-P1-18 and S-P2-01. Update critical path total from 38.5h to 40h.

---

### TL-04: S-P1-17 Dependency Is Incorrect
**Severity:** Critical
**Location:** S-P1-17 (LangGraph Graph Definition)

**Finding:** The WB lists S-P1-17 as depending on S-P1-16 (Audit Trail Route). The TD's dependency graph shows P1-WU16 (Graph Definition, mapped to S-P1-17) depending on the chain P1-WU15 -> P1-WU13 -> P1-WU12 -> P1-WU09. However, looking more carefully at the TD dependency tree:

```
P1-WU15 (audit route) -> P1-WU16 (graph stubs)
```

And S-P1-16 maps to P1-WU15 (Audit Trail Route), while S-P1-17 maps to P1-WU16 (Graph Definition). So the dependency S-P1-17 -> S-P1-16 is technically correct per the TD graph. This finding is retracted on closer inspection.

However, the graph definition (P1-WU16) creates 5 files (`state.py`, `loan_processing.py`, `supervisor.py`, `stubs.py`, `checkpointer.py`), and the TD describes it as requiring both repository access and route context. The dependency on the audit trail route specifically exists because the supervisor node records audit events, which requires the audit trail patterns to be established. This is correct.

**Recommendation:** No change needed -- retracted. The dependency chain is faithful.

---

### TL-05: S-P2-08 Depends on S-P2-07, But TD Shows P2-WU08 Depending on P2-WU07
**Severity:** Positive
**Location:** S-P2-08

**Finding:** The dependency is correctly mapped. The PM correctly identified that the fraud sensitivity endpoint should be added to the existing config routes file (created in P2-WU06/S-P2-06 for thresholds), after the review queue is built.

---

### TL-06: Effort Estimate for S-P1-05 (Seed Data) Is Low
**Severity:** Warning
**Location:** S-P1-05

**Finding:** S-P1-05 is estimated at S (0.5h). The TD's P1-WU05 description requires: creating 3 users with different roles, generating 3 API keys with SHA-256 hashing and prefix extraction, making keys deterministic in dev mode with a fixed seed, inserting 3 configuration rows (thresholds, fraud sensitivity, rate limits), providing a downgrade path, and printing keys to stdout. The exit condition involves an asyncpg-based Python verification script.

This is not a simple 30-minute task. The deterministic key generation alone requires careful implementation to ensure reproducibility.

**Recommendation:** Upgrade to M (1h).

---

### TL-07: Effort Estimate for S-P1-17 (LangGraph Graph Definition) Is Reasonable But Tight
**Severity:** Suggestion
**Location:** S-P1-17

**Finding:** S-P1-17 is estimated at L (2h) and creates 5 files: `state.py`, `loan_processing.py`, `supervisor.py`, `stubs.py`, `checkpointer.py`. The TD describes this as the most complex single WU in Phase 1 -- it includes the full graph definition with conditional edges, 3 routing functions, a checkpoint configuration with a separate connection pool, an aggregation node, and 6 stub implementations. Additionally, it requires tests for graph compilation, stub shapes, and routing logic.

At 2h this is achievable for a developer familiar with LangGraph, but borderline. Given this is on the critical path, any underrun compounds.

**Recommendation:** Maintain L (2h) but flag as a sprint risk. If the implementing agent is not pre-loaded with LangGraph context, consider 2.5h.

---

### TL-08: Effort Estimate for S-P2-02 (Document Processing Agent) Is Low
**Severity:** Warning
**Location:** S-P2-02

**Finding:** S-P2-02 is estimated at M (1.5h). The TD's P2-WU02 description involves: creating `llm.py` (LLM wrapper with retry, provider selection, callback integration), implementing the document processing node (iterate documents, fetch from MinIO, call PII redaction, store redacted version, call vision model for classification, call vision model for extraction, validate LLM response structure, retry on malformed responses, extract PDF metadata, store results via repository, create agent decision records, create audit events). This is two significant components (`llm.py` + the agent node) with substantial error handling.

The `llm.py` module alone (retry logic, multiple model providers, LangFuse callback integration, token tracking) is a non-trivial piece.

**Recommendation:** Upgrade to L (2h), or split `llm.py` creation into its own S (0.5h) sub-task.

---

### TL-09: S-P2-09 (UI Build Infrastructure) Depends on S-P2-07, Not On Backend At All
**Severity:** Warning
**Location:** S-P2-09

**Finding:** S-P2-09 (UI Build Infrastructure) is listed as depending on S-P2-07 (Review Queue). The TD's P2-WU09a has no explicit backend dependency -- it is pure UI scaffolding (Vite config, TypeScript config, entry point). It only depends on `packages/ui/package.json` which was created in P1-WU01. The TD's Phase 2 dependency graph shows P2-WU09a as an independent chain from the backend WUs.

This false dependency means the entire UI track is blocked by the backend review queue implementation, preventing any UI parallelism with backend Phase 2 work.

**Recommendation:** Remove the S-P2-07 dependency from S-P2-09. Instead, S-P2-09 depends only on S-P1-01 (Project Scaffolding, which creates `packages/ui/package.json`). This allows Sprint 6 (UI) to overlap with Sprint 5 (backend financial analysis and routing), significantly improving total elapsed time. The UI auth/routing story (S-P2-10) also does not depend on backend review queue -- it can work with any authenticated API endpoint, which exists by end of Sprint 2.

---

### TL-10: S-P3a-01 (RAG Infrastructure) Depends on S-P2-13, Blocking Unnecessarily
**Severity:** Warning
**Location:** S-P3a-01

**Finding:** S-P3a-01 (Knowledge Base and RAG Infrastructure) depends on S-P2-13 (Phase 2 Integration Test). The RAG infrastructure creates an embedding service, knowledge search tool, and embedding repository. It has no technical dependency on the UI integration test. Its actual dependency is on:
- The database being set up (embeddings table needs migrations) -- satisfied by Phase 1
- The Redis service for caching -- satisfied by Phase 1
- The `packages/db/src/repositories/` pattern -- satisfied by Phase 1

The RAG infrastructure could start as early as the end of Phase 1, overlapping with Phase 2 backend work.

**Recommendation:** Change S-P3a-01 dependency to S-P1-19 (Phase 1 Integration Test). This allows RAG infrastructure to begin during Sprint 5 or 6, overlapping with Phase 2 UI work. The compliance agent (S-P3a-02) still depends on S-P3a-01, so the ordering within Phase 3a is preserved.

---

### TL-11: S-P3b-01 Depends on S-P3a-05, but FRED Client Has No Phase 3a Dependency
**Severity:** Warning
**Location:** S-P3b-01 (FRED API Client)

**Finding:** The FRED API client creates a service that calls an external API, caches results in Redis, and provides a LangGraph tool. It has no dependency on compliance checking, fraud detection, or denial coaching (Phase 3a). Its actual dependencies are:
- Redis service (Phase 1)
- The services/ directory pattern (Phase 1)
- The LangGraph tools/ pattern (Phase 2, when tools like calculator start)

Phase 3b stories could overlap with late Phase 3a work, but S-P3b-01 through S-P3b-02 (FRED + calculator API) could start earlier.

**Recommendation:** Change S-P3b-01 dependency to S-P2-08 (end of Phase 2 backend). This allows the FRED client and calculator API to begin while Phase 3a agents are being implemented, reducing total elapsed time.

---

### TL-12: Exit Condition for S-P1-20 (Developer Documentation) Tests Existence, Not Behavior
**Severity:** Warning
**Location:** S-P1-20

**Finding:** The exit condition is:
```
test -f README.md && test -f .env.example && grep -q REDIS_PASSWORD .env.example && grep -q "quickstart" README.md
```

This tests that files exist and contain specific strings, but does not verify the documentation is actually useful. The TD's P1-WU19 exit condition is similarly lightweight:
```
test -f .env.example && grep -c "DATABASE_URL" .env.example && grep -c "make setup" README.md
```

For a documentation task, file existence is a reasonable minimum. However, the TD specifies that `.env.example` should include "all environment variables from architecture Appendix A" -- verifying one variable (`REDIS_PASSWORD` or `DATABASE_URL`) does not catch a half-complete `.env.example`.

**Recommendation:** Strengthen the exit condition to verify at least the critical variables exist in `.env.example`:
```bash
test -f .env.example && \
grep -q DATABASE_URL .env.example && \
grep -q REDIS_PASSWORD .env.example && \
grep -q SSN_ENCRYPTION_KEY .env.example && \
grep -q ANTHROPIC_API_KEY .env.example && \
test -f README.md && grep -q "make setup" README.md
```

---

### TL-13: Exit Condition for S-P2-09 (UI Build Infrastructure) Uses `pnpm preview`
**Severity:** Warning
**Location:** S-P2-09

**Finding:** The exit condition is `cd packages/ui && pnpm build && pnpm preview`. The `pnpm preview` command starts a server and blocks -- it does not exit. This is not a machine-verifiable pass/fail command.

The TD's P2-WU09a exit condition is `cd packages/ui && npx tsc --noEmit`, which is a clean pass/fail check.

**Recommendation:** Change exit condition to match the TD: `cd packages/ui && pnpm build && npx tsc --noEmit`.

---

### TL-14: Exit Conditions for P4 UI Stories Are Weak
**Severity:** Warning
**Location:** S-P4-03, S-P4-04

**Finding:** S-P4-03 (Admin Configuration UI) and S-P4-04 (Portfolio Analytics Dashboard) have exit conditions of `cd packages/ui && pnpm test -- admin-config.test.tsx` and `cd packages/ui && pnpm test -- analytics.test.tsx` respectively. These assume test files will be created as part of the story, which is good.

However, the TD's Phase 4 WU descriptions note: "Phase 4 WUs will be refined with stronger exit conditions before implementation begins" and "UI WUs should include at least one vitest component test in addition to type checking." The current WB exit conditions only reference tests, not type checking.

**Recommendation:** Add `&& npx tsc --noEmit` to all UI story exit conditions in Phase 4, to match the TD's guidance.

---

### TL-15: S-P1-03 Exit Condition Diverges From TD
**Severity:** Warning
**Location:** S-P1-03 (ORM Models)

**Finding:** The WB exit condition is `cd packages/db && pytest tests/unit/test_models.py -v`. The TD's P1-WU03 exit condition is:
```bash
cd packages/db && python -c "from src.models import Application, Document, AuditEvent, User, ApiKey, AgentDecision, Configuration, Embedding; print('All models imported successfully')"
```

The TD uses a simple import check (no test file). The WB introduces a test file that doesn't exist in the TD's file manifest for this WU. Either the WB is adding scope (write model tests) or the exit condition references a nonexistent file.

**Recommendation:** Either (a) match the TD's import-check exit condition, or (b) explicitly state in the story description that a `tests/unit/test_models.py` file must be created. Option (b) is preferable -- model import tests are cheap and catch syntax errors -- but the scope addition should be deliberate.

---

### TL-16: WU Count Discrepancy
**Severity:** Suggestion
**Location:** Section 1 Overview

**Finding:** The overview states "55 work units from the technical design" and 55 stories. The TD defines the following WUs:
- Phase 1: P1-WU01 through P1-WU19 (19 WUs, with WU06 split into 06a/06b = 20, and WU09 is not split in P1)
- Phase 2: P2-WU01 through P2-WU12 (12 WUs, with WU09 split into 09a/09b = 13)
- Phase 3a: P3a-WU01 through P3a-WU05 (5 WUs)
- Phase 3b: P3b-WU01 through P3b-WU07 (7 WUs)
- Phase 4: P4-WU01 through P4-WU10 (10 WUs)

Total: 20 + 13 + 5 + 7 + 10 = 55 WUs, mapping to 55 stories. The count is correct.

---

### TL-17: Sprint 3 Load Is High -- 8 Stories, 10 Hours
**Severity:** Warning
**Location:** Sprint 3

**Finding:** Sprint 3 packs 8 stories totaling 10 hours, including the most complex story in the project (S-P1-17, LangGraph Graph Definition, 2h) and an integration test (S-P1-19, 1.5h). The parallelization map assumes significant overlap between application routes and agent framework, but S-P1-17 through S-P1-20 form a strict sequential chain of 5.5h.

The maximum parallelization is:
- Parallel pair 1: S-P1-13 (1.5h) and S-P1-15 (1h) = 1.5h elapsed
- Parallel pair 2: S-P1-14 (1h) and S-P1-16 (1h) = 1h elapsed
- Sequential tail: S-P1-17 (2h) -> S-P1-18 (1h) -> S-P1-19 (1.5h) -> S-P1-20 (1h) = 5.5h

Total minimum elapsed: 1.5 + 1 + 5.5 = 8h. This is the most loaded sprint.

**Recommendation:** Consider splitting Sprint 3 into two sub-sprints: Sprint 3a (Application Routes: S-P1-13 through S-P1-16, ~4.5h with parallelism ~2.5h) and Sprint 3b (Agent Framework: S-P1-17 through S-P1-20, ~5.5h sequential). This gives smaller review surfaces and earlier feedback on application routes before investing in the graph definition.

---

### TL-18: Sprint 9 Has Too Many Independent Stories for Meaningful Review
**Severity:** Suggestion
**Location:** Sprint 9

**Finding:** Sprint 9 includes 10 stories across 5 parallel groups. While the parallelism is technically sound (most stories are independent), a 10-story sprint review covering LangFuse integration, admin UI, analytics dashboard, deployment manifests, CI pipeline, key lifecycle, and documentation polish would be a chaotic review session. The review-governance rules target ~400 lines per PR.

**Recommendation:** Accept this as-is given Phase 4's lower priority (P1/P2), but note that each story should be its own PR and reviewed individually, not batched into one sprint review.

---

### TL-19: Scope Drift in S-P2-06 Description
**Severity:** Warning
**Location:** S-P2-06 (Confidence-Based Routing)

**Finding:** The story description says "Replace routing stub with real confidence-based decision logic." The TD's P2-WU06 actually does three things: (1) update `aggregation_node`, (2) update `routing_decision_node`, and (3) create a new `routes/config.py` file for threshold management endpoints. Creating a new API route module with GET and PUT endpoints is a different concern than updating agent graph logic.

The story description mentions "Handle agent disagreements" and the exit condition references `tests/integration/test_config.py`, so the config routes are implicitly included. However, the story description should explicitly call out the threshold management endpoints, since this adds an API surface.

**Recommendation:** Update the S-P2-06 description to explicitly mention: "Also implements GET/PUT /v1/config/thresholds endpoints for threshold management (reviewer role required)." This matches the TD's P2-WU06 scope.

---

### TL-20: Critical Path Is Missing S-P1-19 and Over-Strict in Phase 2 UI
**Severity:** Warning
**Location:** Section 4 (Critical Path Analysis)

**Finding:** Two issues with the critical path:

1. As noted in TL-03, the critical path jumps from S-P1-18 directly to S-P2-01, but S-P2-01 depends on S-P1-19 (integration test). The critical path should include S-P1-19 (1.5h).

2. The critical path routes through S-P2-09 -> S-P2-10 -> S-P2-11 (UI stories), but per TL-09, the UI track has no dependency on the backend Phase 2 stories. After S-P2-07, the critical path should continue directly to S-P3a-01 (if the Phase 3a dependency on Phase 2 integration test is corrected per TL-10). The UI stories are a parallel track.

With corrected dependencies, the longest path through Phase 2 would be the backend path: S-P2-01 -> S-P2-02 -> S-P2-03 -> S-P2-04 -> S-P2-06 -> S-P2-07 -> S-P2-13 (integration test), totaling 9.5h. The UI path (S-P2-09 -> S-P2-10 -> S-P2-11 -> S-P2-13 = 5.5h) runs in parallel.

**Recommendation:** Recalculate the critical path after applying dependency corrections from TL-01, TL-09, TL-10, and TL-11. The total will shift, and some stories currently on the critical path will move off it.

---

### TL-21: Risk Register Missing -- LLM Response Validation Complexity
**Severity:** Suggestion
**Location:** Section 5 (Risk Register)

**Finding:** The risk register does not mention the risk of LLM response validation complexity. The TD's P2-WU02 describes: "Validate LLM response structure: Classification must return a valid DocumentType enum value. Extraction must return expected fields for the document type. If malformed, retry up to 3 times; on exhaustion, escalate to human review." This is a significant implementation challenge -- LLM outputs are non-deterministic, and structured output validation is a common source of production bugs.

**Recommendation:** Add risk entry for Sprint 4:
| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P2-02 | LLM response validation logic is complex; non-deterministic outputs make testing fragile | High | Medium | Use Pydantic models for response validation; mock LLM responses in tests with known-good and known-bad shapes |

---

### TL-22: User Story Mapping Drift for S-P1-09 and S-P1-12
**Severity:** Suggestion
**Location:** S-P1-09, S-P1-12

**Finding:** S-P1-09 (Health Checks and Error Handling) maps to US-065, but the TD's P1-WU08 maps to US-067. S-P1-12 (RBAC Middleware) maps to US-060, matching the TD's P1-WU11 -- this is correct.

For S-P1-09: the TD says P1-WU08 satisfies US-067. The WB says S-P1-09 satisfies US-065. Looking at the broader mapping, US-065 appears to be about health checks (correct for this story), while US-067 appears in the TD for this WU. Without access to requirements.md, one of these is likely a typo.

**Recommendation:** Verify the user story numbers against `plans/requirements.md` for S-P1-09 and align.

## Effort Estimate Audit

| Story | PM Estimate | TL Assessment | Rationale |
|-------|-------------|---------------|-----------|
| S-P1-05 (Seed Data) | S (0.5h) | M (1h) | Deterministic key generation, 3 users, 3 API keys with SHA-256, 3 config rows, downgrade path, asyncpg verification script. Not a simple 30-minute task. |
| S-P2-02 (Doc Processing Agent) | M (1.5h) | L (2h) | Creates `llm.py` (retry, multi-provider, callbacks, token tracking) AND the document processing node (iterate docs, fetch MinIO, PII redaction, vision model classification, extraction, response validation, metadata extraction, repository writes, audit events). Two significant components. |
| S-P1-17 (LangGraph Graph) | L (2h) | L (2h) -- tight | 5 files, full graph with conditional edges, 3 routing functions, separate checkpoint pool, 6 stub implementations, tests. Achievable but on the boundary. Flag as sprint risk. |
| S-P3a-01 (RAG Infrastructure) | L (2h) | L (2h) -- accurate | Embedding service, knowledge search tool, embedding repository, seed script for regulatory content. Well-scoped for 2h. |
| S-P2-06 (Routing + Config) | M (1.5h) | M-L (1.5-2h) | Updates 2 graph nodes AND creates a new route module with 2 endpoints. Borderline, but achievable if config routes are straightforward. |
| S-P4-10 (Documentation Polish) | M (1h) | M (1h) -- if truly polish | If this is polish of existing docs, 1h is fine. If it is writing API reference, deployment guide, and troubleshooting from scratch, it should be L (2h). The WB description says "polish" which aligns with 1h. |

**Net impact:** +1.5h total if adjustments applied (S-P1-05: +0.5h, S-P2-02: +0.5h, S-P2-06: +0.5h potential). Sprint 1 would go from 7.5h to 8h, Sprint 4 from 3h to 3.5h, Sprint 5 from 7.5h to potentially 8h.

## Dependency Chain Audit

| Issue | Current Dependency | Correct Dependency | Impact |
|-------|-------------------|-------------------|--------|
| S-P1-08 (Settings/DI) | S-P1-06, S-P1-07 | S-P1-06 only | Removes false dependency on supporting repos; enables Sprint 2 parallelism |
| S-P1-11 (Auth Middleware) | S-P1-10 | S-P1-10 AND S-P1-05 | Auth tests need seeded API keys; dependency currently implicit via sprint ordering |
| S-P2-09 (UI Build Infra) | S-P2-07 | S-P1-01 | **Major unlock**: enables UI track to start during Sprint 4-5, overlapping with backend Phase 2 |
| S-P3a-01 (RAG Infra) | S-P2-13 | S-P1-19 | Enables RAG infrastructure during Sprint 5-6 |
| S-P3b-01 (FRED Client) | S-P3a-05 | S-P2-08 or S-P1-19 | Enables FRED client during Phase 3a |
| Critical path S-P1-18 -> S-P2-01 | Missing S-P1-19 | S-P1-18 -> S-P1-19 -> S-P2-01 | Adds 1.5h to critical path |

## Sprint Load Assessment

| Sprint | Stories | Estimated Hours | Assessment |
|--------|---------|----------------|------------|
| Sprint 1 | 7 | 7.5h (should be 8h per TL-06) | Tight but feasible with S-P1-06/S-P1-07 parallelism. Adjust S-P1-05 estimate. |
| Sprint 2 | 5 | 5.5h | Reasonable. Could unlock parallelism per TL-01. |
| Sprint 3 | 8 | 10h | **Overloaded.** 5.5h sequential tail on critical path. Consider splitting per TL-17. |
| Sprint 4 | 2 | 3h (should be 3.5h per TL-08) | Light sprint. Good recovery point. |
| Sprint 5 | 6 | 7.5h (potentially 8h per TL-08 effect) | Well-balanced with good parallelism after S-P2-03. |
| Sprint 6 | 5 | 7h | Reasonable. Could start earlier per TL-09. |
| Sprint 7 | 5 | 7.5h | Good parallelism after S-P3a-01. |
| Sprint 8 | 7 | 9.5h | Moderate load with good parallelism. |
| Sprint 9 | 10 | 13h | High count but high parallelism. Each story should be its own PR. |

**Overall:** Total effort shifts from ~70h to ~72.5h with estimate adjustments. Critical path shifts from 38.5h to ~40h with S-P1-19 correction. With dependency corrections (especially TL-09 and TL-10), the achievable _elapsed_ time could decrease by 5-8h through UI/backend parallelism.

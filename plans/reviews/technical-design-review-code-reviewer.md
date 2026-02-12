<!-- This project was developed with assistance from AI tools. -->

# Technical Design Review -- Code Reviewer

**Document:** `plans/technical-design.md` v1.0
**Reviewer:** Code Reviewer (Opus)
**Date:** 2026-02-12
**Upstream references:** `plans/architecture.md` v1.0, `plans/requirements.md`, `.claude/rules/review-governance.md`, `.claude/rules/agent-workflow.md`

---

## 1. Technical Design Review Checklist

| # | Check | Verdict | Notes |
|---|-------|---------|-------|
| 1 | Contracts are concrete | **PASS** | Pydantic schemas (Section 2.2), repository method signatures (Section 2.3), agent state TypedDicts (Section 2.4), middleware contracts (Section 2.6), service protocols (Section 2.7), and error format (Section 2.8) all contain actual type definitions, field names, and JSON shapes. This is an unusually concrete TD -- code-level definitions are provided rather than prose descriptions. |
| 2 | Data flow covers error paths | **PASS** | Error paths are defined at each agent boundary (architecture Section 3.4 error table is referenced), RFC 7807 error hierarchy is specified (P1-WU08), and per-boundary failure behavior is documented. The TD relies on the architecture for the error-path table but correctly references it. |
| 3 | Exit conditions are machine-verifiable | **PASS WITH EXCEPTIONS** | All WUs in Phases 1-3 have bash commands. However, several Phase 4 exit conditions are weak (see W-3). Additionally, some exit conditions could pass while the WU is incomplete (see W-4). |
| 4 | File structure maps to actual codebase | **PASS WITH EXCEPTIONS** | The `packages/` directory does not yet exist (correct -- this is a greenfield build). The TD's proposed file paths are internally consistent with architecture Section 9. However, the TD introduces files not present in the architecture file tree (see W-5). |
| 5 | No TBDs in binding contracts | **PASS** | OQ-1 (knowledge base content) and OQ-9 (PII redaction approach) are explicitly flagged as open questions. No binding interface contract contains "TBD". The `thresholds` and other untyped `dict` fields are a quality concern (see W-1) but not strictly "TBD" -- they have specified shapes in comments. |

---

## 2. Findings

### Critical

#### C-1: P1-WU11 modifies `main.py` before P1-WU18 creates it

**[technical-design.md:1115-1116]**

P1-WU11 (RBAC Middleware) lists under "Files to modify": `packages/api/src/main.py` (add startup event). But `main.py` is listed as "Files to create" in P1-WU18 (Phase 1 Integration Test and FastAPI App Factory). The dependency graph shows P1-WU11 depends on P1-WU10 -> P1-WU05 -> P1-WU04 -> P1-WU03. P1-WU18 depends on P1-WU17 -> P1-WU16 -> P1-WU15. There is no dependency link from P1-WU11 to P1-WU18 or any earlier WU that creates `main.py`.

This means an agent executing P1-WU11 will attempt to modify a file that does not exist.

**Suggestion:** Either (a) move the `main.py` creation to an earlier WU (e.g., P1-WU07 or P1-WU08 which establishes the API core), or (b) split the startup event logic from P1-WU11 and defer it to P1-WU18 when `main.py` is created, or (c) have P1-WU11 create a standalone startup module (e.g., `packages/api/src/startup.py`) that P1-WU18 imports.

---

### Warning

#### W-1: Untyped `dict` fields persist in binding contracts from architecture

**[technical-design.md:201, 255, 269, 335, 339, 444, 515, 517, 527]**

The architecture review (see `plans/reviews/architecture-review-code-reviewer.md`) flagged untyped `dict` fields in the agent state TypedDicts as a concern. The TD reproduces them verbatim without strengthening the types. Specific instances:

- `AgentAnalysisSummary.findings: dict | None` (line 201) -- no inner type
- `ReviewQueueItem.agent_summary: dict | None` (line 255) -- no inner type
- `AuditEventResponse.details: dict` (line 269) -- no inner type
- `DataEnvelope.data: dict | list` (line 335) -- fully untyped
- `PaginatedEnvelope.data: list` (line 339) -- untyped list
- `DocumentResult.extracted_data: dict[str, Any]` (line 444) -- Any inner type
- `LoanProcessingState.thresholds: dict` (line 515) -- should be `ThresholdsSnapshot` TypedDict
- `LoanProcessingState.previous_analyses: list[dict]` (line 517) -- no inner type
- `IntakeState.tool_results: list[dict]` and `citations: list[dict]` (line 527) -- no inner type

Since these are binding contracts between WUs, two agents implementing adjacent WUs may produce incompatible `dict` shapes. The `thresholds` field is particularly concerning because it is read by the routing decision node (critical business logic) -- if the shape is inconsistent, routing fails at runtime.

**Suggestion:** Define concrete TypedDicts for at least the high-traffic contract fields: `ThresholdsSnapshot` (with `auto_approve: float`, `escalation: float`, `denial: float`), `AgentSummarySnapshot` (for `ReviewQueueItem.agent_summary`), and `PreviousAnalysis` (for `previous_analyses`). The `DataEnvelope` and `PaginatedEnvelope` should use generics: `DataEnvelope[T]` / `PaginatedEnvelope[T]`. This was flagged in the architecture review and remains unresolved.

#### W-2: P1-WU06 creates 7 files -- exceeds chunking heuristic

**[technical-design.md:893-899]**

P1-WU06 (Database Repositories) lists 7 files to create:

1. `packages/db/src/repositories/__init__.py`
2. `packages/db/src/repositories/application.py`
3. `packages/db/src/repositories/document.py`
4. `packages/db/src/repositories/audit.py`
5. `packages/db/src/repositories/agent_decision.py`
6. `packages/db/src/repositories/user.py`
7. `packages/db/src/repositories/configuration.py`

Plus the WU description says tests must be written as part of the WU (line 923: "The test file `tests/test_repositories.py` must be written as part of this WU"), making it 8 files total.

The chunking heuristic in `agent-workflow.md` says 3-5 files per task. At 7-8 files touching repository logic plus tests, this WU is likely to take longer than one hour and increases compound error probability.

**Suggestion:** Split into two WUs. WU06a: `ApplicationRepository`, `DocumentRepository`, `AuditRepository` + `__init__.py` (core CRUD -- 4 files). WU06b: `AgentDecisionRepository`, `UserRepository`, `ConfigurationRepository` (supporting repositories -- 3 files). Each includes its own test file.

#### W-3: Phase 4 exit conditions are weak or non-verifiable

**[technical-design.md:2525-2579]**

Several Phase 4 WU exit conditions are effectively "grep for a string" rather than verifiable behavior:

- P4-WU10 (Documentation Polish): `grep -c "walkthrough\|pattern\|checkpoint" README.md # >= 3 occurrences` -- this checks that words exist, not that documentation is useful or correct.
- P4-WU03 (Admin Configuration UI): `cd packages/ui && npx tsc --noEmit` -- type-checking alone does not verify that the UI renders correctly or connects to the right endpoints.
- P4-WU04 (Portfolio Analytics Dashboard): same `npx tsc --noEmit` -- no functional verification.

The agent-workflow rule states that exit conditions must be "machine-verifiable" and that "Implementation is complete" is not acceptable. While `tsc --noEmit` is technically machine-verifiable, it only proves type correctness, not functional correctness.

**Suggestion:** Add at least one Vitest component test per UI WU (e.g., "renders threshold controls" or "renders analytics chart with mock data"). For P4-WU10, add a structural check such as verifying specific section headings exist in the README. Acknowledge that Phase 4 is further out and WU definitions will be refined before implementation begins.

#### W-4: Some exit conditions could pass while the WU is incomplete

**[technical-design.md:706-710, 982-984]**

- P1-WU01 exit condition: `cd packages/api && uv sync --frozen 2>&1 | tail -1` -- This tests dependency resolution but not that all required files actually exist. If `__init__.py` files are missing, `uv sync` will still succeed but the packages will not be importable.
- P1-WU07 exit condition: `python -c "from src.settings import Settings; s = Settings(_env_file='.env.test'); print(f'Settings loaded: {s.API_PORT}')"` -- This only tests that `Settings` can be instantiated. The dependency injection functions (`get_application_repo`, etc.) are not tested at all.

**Suggestion:** For P1-WU01, add `python -c "import src; print('api package importable')"` for each package. For P1-WU07, add a check that exercises at least one dependency function, or add a requirement for a minimal test file.

#### W-5: TD introduces files not present in architecture Section 9 file tree

**[technical-design.md:996, 1112]**

The TD creates files that are not listed in the architecture's file structure (Section 9):

- `packages/api/src/errors.py` (P1-WU08, line 996) -- not in architecture file tree
- `packages/api/src/middleware/authorization.py` (P1-WU11, line 1112) -- not in architecture file tree; the architecture describes RBAC as a `require_role()` dependency pattern (Section 6.2) but does not give it a dedicated middleware file

These are reasonable additions (the error hierarchy and authorization logic need to live somewhere), but the deviation from the architecture file tree should be explicitly documented as a TD design decision.

**Suggestion:** Add a brief note in the TD overview or in each WU acknowledging that these files are additions to the architecture's file tree, with rationale (e.g., "RFC 7807 error handling warrants a dedicated module not shown in the architecture file tree").

#### W-6: DocumentResult type values diverge from architecture

**[technical-design.md:442]**

The TD's `DocumentResult.type` comment lists: `W2, PAY_STUB, TAX_RETURN, BANK_STATEMENT, APPRAISAL, PHOTO, UNKNOWN` (line 442). The architecture's `DocumentResult.type` comment (architecture line 1189) lists: `W2, PAYSTUB, TAX_RETURN, BANK_STATEMENT, ID, OTHER`.

Differences:
- `PAY_STUB` (TD, underscore) vs `PAYSTUB` (architecture, no underscore)
- `APPRAISAL, PHOTO, UNKNOWN` (TD) vs `ID, OTHER` (architecture)

The TD's `DocumentType` enum (line 98-105) uses `PAY_STUB`, `APPRAISAL`, `PHOTO`, `UNKNOWN`. The architecture's database DDL (line 387) uses `PAY_STUB`. So the TD is internally consistent with its own enum and with the architecture's DDL, but diverges from the architecture's `DocumentResult` TypedDict comment.

This is not blocking because the TD's enum is the authoritative contract, and the architecture's TypedDict comment was an example. However, the inconsistency could confuse an implementer who reads both documents.

**Suggestion:** Note in Section 2.4 that the `DocumentResult.type` values are governed by the `DocumentType` enum in Section 2.1, superseding the architecture's example values.

#### W-7: P2-WU09 (UI Shell) creates 9 files -- nearly double the chunking limit

**[technical-design.md:1886-1895]**

P2-WU09 creates 9 files for the React app shell. This significantly exceeds the 3-5 file chunking heuristic.

**Suggestion:** Split into two WUs. WU09a: Build infrastructure (`vite.config.ts`, `tsconfig.json`, `index.html`, `main.tsx`) -- 4 files. WU09b: Auth and routing (`__root.tsx`, `index.tsx`, `dashboard.tsx`, `use-auth.ts`, `api-client.ts`) -- 5 files. WU09b depends on WU09a.

#### W-8: Income discrepancy thresholds differ between risk assessment and fraud detection without clear justification

**[technical-design.md:1731, 2146, 2259]**

P2-WU05 (Risk Assessment) says "Cross-validate income across sources (10% tolerance per US-016)" (line 1731). P3a-WU03 (Fraud Detection) says "Flag > 15% variance" (line 2146). The Phase 3a context package acknowledges this: "15% income variance threshold (vs 10% in risk assessment)" (line 2259).

While having different thresholds in different agents may be intentional (risk assessment is stricter than fraud detection, which seems backwards), the TD does not explain why these differ. An implementer may question whether this is a design decision or an error.

**Suggestion:** Add a brief note in P3a-WU03 or the Phase 3a context package explaining the rationale for the different thresholds. If intentional: the risk agent flags at 10% as a risk factor (impacting confidence), while the fraud agent flags at 15% as a potential fraud indicator (impacting the `has_fraud_flags` state). If unintentional, align them.

---

### Suggestion

#### S-1: P0 user stories US-071 (Non-Functional Performance Targets) and US-072 (Graceful Degradation) are not mapped to any WU

US-071 (P0, All Phases) defines performance targets: application submission < 30 seconds, document upload < 5 seconds, etc. US-072 (P0, All Phases) defines graceful degradation requirements. Neither story appears in any WU's "Satisfies" field.

These are cross-cutting concerns that should be addressed as acceptance criteria on integration test WUs (P1-WU18, P2-WU12) or as dedicated NFR verification WUs.

**Suggestion:** Add US-071 as an acceptance criterion on integration test WUs (P1-WU18, P2-WU12) with specific timing assertions. Add US-072 to the WU that implements Redis fallback (P3b-WU04) and to P1-WU08 (health/readiness checks).

#### S-2: `DataEnvelope` and `PaginatedEnvelope` should be generic types

**[technical-design.md:334-340]**

```python
class DataEnvelope(BaseModel):
    data: dict | list

class PaginatedEnvelope(BaseModel):
    data: list
    pagination: PaginationMeta
```

These are untyped wrappers that provide no compile-time safety. Pydantic v2 supports generic models:

```python
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

class DataEnvelope(BaseModel, Generic[T]):
    data: T

class PaginatedEnvelope(BaseModel, Generic[T]):
    data: list[T]
    pagination: PaginationMeta
```

This would allow route handlers to specify `DataEnvelope[ApplicationSummary]` and get proper OpenAPI schema generation and runtime validation.

#### S-3: `comparison` calculation type from architecture not mentioned in TD

The architecture (Section 4.2, line 956) lists `"comparison"` as a supported calculation type: `"monthly_payment"`, `"total_interest"`, `"dti_preview"`, `"affordability"`, `"amortization"`, `"comparison"`. This maps to US-050 (Scenario Comparison Mode, P1 priority, Phase 3b).

P3b-WU02 mentions "Comparison mode: side-by-side scenarios A and B" in the UI description (line 2329), but the calculator tool definition in P3a-WU04 (line 2181-2186) does not mention the `comparison` type. Ensure the tool implementation includes this.

#### S-4: Phase 1 dependency graph has a diamond merge that may cause confusion

**[technical-design.md:634-671]**

P1-WU04 (migrations) feeds into both P1-WU05 (seed data) and P1-WU06 (repositories). P1-WU05 feeds into P1-WU10 (auth middleware). P1-WU06 feeds into P1-WU09 (correlation ID middleware) and P1-WU07 (settings). Then P1-WU10 and P1-WU09 converge at P1-WU12 (application CRUD routes).

This diamond pattern is fine for dependency resolution, but the "Context Package" definitions should be explicit about which earlier WU outputs are needed at each convergence point, to prevent an implementer from missing a required import. The Phase 1 Context Package section (lines 1479-1514) does cover this at a work-area level, which is sufficient.

#### S-5: Consider adding `__init__.py` to middleware directory in P1-WU09

**[technical-design.md:1032]**

P1-WU09 creates `packages/api/src/middleware/__init__.py` but this is listed alongside the correlation_id and logging middleware files. The P1-WU10 (auth) and P1-WU11 (RBAC) WUs do not list `__init__.py` creation because P1-WU09 already creates it. This is correct, but the dependency on P1-WU09 for the `__init__.py` is implicit.

**Suggestion:** Add a note in P1-WU10 and P1-WU11 that they depend on P1-WU09 having created the middleware package `__init__.py`.

---

### Positive

#### P-1: Exceptionally concrete interface contracts

The interface contracts in Section 2 are remarkably detailed for a TD. Actual Python code with complete Pydantic models, field validators, alias configurations, and model_config settings are provided. This level of concreteness means two agents working on adjacent WUs (e.g., the backend developer implementing routes and the database engineer implementing repositories) can produce compatible code without coordination. This is the highest quality TD contract section I have reviewed.

#### P-2: Agent state TypedDicts are fully typed (compared to architecture)

The architecture used untyped `dict` for many agent result fields. The TD has defined concrete TypedDicts for all six agent result types (`DocumentResult`, `CreditAnalysisResult`, `RiskAssessmentResult`, `ComplianceCheckResult`, `FraudDetectionResult`, `AggregatedResults`, `CoachingOutput`). This is a significant improvement from the architecture and directly addresses a concern from the architecture review.

#### P-3: Context Packages are well-structured

Each phase includes a "Context Package" section (e.g., lines 1479-1514 for Phase 1) that explicitly lists which files to read, which binding contracts apply, which key decisions are relevant, and the scope boundary. This directly supports the context engineering guidance in `agent-workflow.md` and prevents agents from speculatively loading irrelevant files.

#### P-4: Cross-task dependency map is comprehensive

Section 8 (Cross-Task Dependencies, lines 2585-2602) provides a clear "Produces / Consumed By / Contract" table that traces every major output through the system. This enables a project manager to parallelize work correctly and catch missing dependencies.

#### P-5: Validation logic in Pydantic schemas is thorough

The `ThresholdUpdateRequest` validator (lines 283-296) enforces ordering (`auto_approve > escalation > denial`). The `ReviewRequest` validator (lines 226-233) conditionally requires `requestedDocuments` when the decision is `REQUEST_MORE_DOCUMENTS`. These are the kind of business rules that are often missed in TDs and discovered during implementation.

---

## 3. Summary Table

| ID | Severity | Location | Description |
|----|----------|----------|-------------|
| C-1 | Critical | P1-WU11 / P1-WU18 | `main.py` modified before it is created; dependency graph broken |
| W-1 | Warning | Section 2.2, 2.4 | Untyped `dict` fields in binding contracts (carried from architecture) |
| W-2 | Warning | P1-WU06 | 7-8 files -- exceeds chunking heuristic |
| W-3 | Warning | Phase 4 WUs | Weak exit conditions (grep/tsc only) |
| W-4 | Warning | P1-WU01, P1-WU07 | Exit conditions could pass with incomplete WU |
| W-5 | Warning | P1-WU08, P1-WU11 | Files not in architecture Section 9 tree, no documented justification |
| W-6 | Warning | Section 2.4 | `DocumentResult.type` values diverge from architecture TypedDict |
| W-7 | Warning | P2-WU09 | 9 files -- nearly double chunking limit |
| W-8 | Warning | P2-WU05, P3a-WU03 | Income discrepancy thresholds differ (10% vs 15%) without rationale |
| S-1 | Suggestion | Cross-cutting | US-071 and US-072 (P0) not mapped to any WU |
| S-2 | Suggestion | Section 2.2 | `DataEnvelope` / `PaginatedEnvelope` should be generic |
| S-3 | Suggestion | P3a-WU04, P3b-WU02 | `comparison` calculation type not mentioned in calculator tool WU |
| S-4 | Suggestion | Phase 1 graph | Diamond dependency merge -- context propagation is adequate but could be more explicit |
| S-5 | Suggestion | P1-WU10, P1-WU11 | Implicit dependency on P1-WU09 for `__init__.py` creation |
| P-1 | Positive | Section 2 | Exceptionally concrete interface contracts with actual code |
| P-2 | Positive | Section 2.4 | Agent state TypedDicts improved from architecture |
| P-3 | Positive | Phase context packages | Well-structured context engineering support |
| P-4 | Positive | Section 8 | Comprehensive cross-task dependency map |
| P-5 | Positive | Section 2.2 | Business rule validators in Pydantic schemas |

---

## 4. Verdict

**REQUEST CHANGES**

The TD is high quality overall -- the interface contracts, WU scoping, and context packages are well above average. However, C-1 (dependency graph broken: P1-WU11 modifies `main.py` before P1-WU18 creates it) is a blocking issue that will cause a concrete implementation failure. An agent assigned to P1-WU11 will attempt to modify a file that does not exist yet.

### Required before implementation begins:

1. **Fix C-1** -- resolve the `main.py` creation/modification ordering conflict between P1-WU11 and P1-WU18.
2. **Address W-2 and W-7** -- split P1-WU06 (7-8 files) and P2-WU09 (9 files) to comply with the 3-5 file chunking heuristic.
3. **Address W-1** -- define `ThresholdsSnapshot` TypedDict for the `thresholds` field in `LoanProcessingState`. This field is read by business-critical routing logic and must not be an untyped `dict`.

### Recommended before Phase 2 begins:

4. **Address W-8** -- document the rationale for different income discrepancy thresholds (10% vs 15%) between risk assessment and fraud detection.
5. **Address W-5** -- document the file additions (`errors.py`, `authorization.py`) as TD design decisions.
6. **Address S-1** -- map US-071 and US-072 to integration test WUs.

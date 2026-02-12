<!-- This project was developed with assistance from AI tools. -->

# Technical Design: AI Mortgage Quickstart

**Version:** 1.0
**Date:** 2026-02-12
**Status:** Proposed

---

## Table of Contents

1. [Overview](#1-overview)
2. [Interface Contracts](#2-interface-contracts)
3. [Phase 1: Foundation](#3-phase-1-foundation)
4. [Phase 2: Core AI Agents](#4-phase-2-core-ai-agents)
5. [Phase 3a: Full Agent Suite](#5-phase-3a-full-agent-suite)
6. [Phase 3b: Public Access and External Data](#6-phase-3b-public-access-and-external-data)
7. [Phase 4: Polish and Operability](#7-phase-4-polish-and-operability)
8. [Cross-Task Dependencies](#8-cross-task-dependencies)
9. [Risks and Open Questions](#9-risks-and-open-questions)

---

## 1. Overview

This document translates the system architecture into implementable work units (WUs) organized by delivery phase. Each WU is scoped to touch 3-5 files, completable in roughly one hour by a single agent, and has a machine-verifiable exit condition.

**Upstream inputs:**
- Product plan (`plans/product-plan.md`) -- features, personas, phasing
- Requirements (`plans/requirements.md`) -- 81 user stories with acceptance criteria
- Architecture (`plans/architecture.md`) -- system design, schema, API shapes, agent graphs

**Conventions:**
- WU identifiers: `P{phase}-WU{number}` (e.g., `P1-WU01`)
- Agent assignments: `backend-developer`, `frontend-developer`, `database-engineer`, `devops-engineer`, `test-engineer`
- All file paths are relative to the project root
- All monetary values are integer cents; all timestamps are UTC
- Python code follows PEP 8 with Ruff; TypeScript follows project style guide with ESLint

---

## 2. Interface Contracts

These are the binding contracts that multiple WUs must agree on. Implementers must conform to these exactly.

### 2.1 Pydantic Enums (`packages/api/src/models/enums.py`)

```python
# This project was developed with assistance from AI tools.
from enum import IntEnum, StrEnum


class ApplicationStatus(StrEnum):
    DRAFT = "DRAFT"
    PROCESSING = "PROCESSING"
    DOCUMENT_PROCESSING = "DOCUMENT_PROCESSING"
    PARALLEL_ANALYSIS = "PARALLEL_ANALYSIS"
    AGGREGATED = "AGGREGATED"
    AUTO_APPROVED = "AUTO_APPROVED"
    ESCALATED = "ESCALATED"
    DENIED = "DENIED"
    PROCESSING_FAILED = "PROCESSING_FAILED"
    AWAITING_DOCUMENTS = "AWAITING_DOCUMENTS"
    HUMAN_APPROVED = "HUMAN_APPROVED"
    HUMAN_DENIED = "HUMAN_DENIED"


class Role(IntEnum):
    LOAN_OFFICER = 1
    SENIOR_UNDERWRITER = 2
    REVIEWER = 3


class AgentName(StrEnum):
    SUPERVISOR = "supervisor"
    DOCUMENT_PROCESSING = "document_processing"
    CREDIT_ANALYSIS = "credit_analysis"
    RISK_ASSESSMENT = "risk_assessment"
    COMPLIANCE = "compliance"
    FRAUD_DETECTION = "fraud_detection"
    DENIAL_COACHING = "denial_coaching"
    INTAKE = "intake"


class DecisionType(StrEnum):
    CLASSIFICATION = "classification"
    EXTRACTION = "extraction"
    CREDIT_ASSESSMENT = "credit_assessment"
    RISK_SCORE = "risk_score"
    COMPLIANCE_CHECK = "compliance_check"
    FRAUD_CHECK = "fraud_check"
    ROUTING_DECISION = "routing_decision"
    COACHING_OUTPUT = "coaching_output"
    AGGREGATION = "aggregation"


class DocumentType(StrEnum):
    W2 = "W2"
    PAY_STUB = "PAY_STUB"
    TAX_RETURN = "TAX_RETURN"
    BANK_STATEMENT = "BANK_STATEMENT"
    APPRAISAL = "APPRAISAL"
    PHOTO = "PHOTO"
    UNKNOWN = "UNKNOWN"


class ReviewDecision(StrEnum):
    APPROVE = "APPROVE"
    DENY = "DENY"
    REQUEST_MORE_DOCUMENTS = "REQUEST_MORE_DOCUMENTS"


class AuditEventType(StrEnum):
    WORKFLOW_INITIALIZED = "workflow_initialized"
    AGENT_DISPATCHED = "agent_dispatched"
    AGENT_COMPLETED = "agent_completed"
    AGGREGATION_COMPLETED = "aggregation_completed"
    ROUTING_DECISION = "routing_decision"
    ESCALATED = "escalated"
    AUTO_APPROVED = "auto_approved"
    DENIED = "denied"
    HUMAN_REVIEW_STARTED = "human_review_started"
    HUMAN_APPROVED = "human_approved"
    HUMAN_DENIED = "human_denied"
    DOCUMENTS_REQUESTED = "documents_requested"
    DOCUMENTS_UPLOADED = "documents_uploaded"
    THRESHOLD_CHANGED = "threshold_changed"
    SENSITIVITY_CHANGED = "sensitivity_changed"
    AUDIT_EXPORTED = "audit_exported"
    APPLICATION_CREATED = "application_created"
    KEY_GENERATED = "key_generated"
    KEY_REVOKED = "key_revoked"


class FraudSensitivity(StrEnum):
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"
```

### 2.2 Pydantic Request/Response Schemas (`packages/api/src/models/schemas.py`)

```python
# This project was developed with assistance from AI tools.
from datetime import datetime
from decimal import Decimal
from uuid import UUID

from pydantic import BaseModel, Field, field_validator

from .enums import (
    ApplicationStatus,
    DocumentType,
    ReviewDecision,
    FraudSensitivity,
)


# --- Application ---

class CreateApplicationRequest(BaseModel):
    borrower_name: str = Field(..., alias="borrowerName", min_length=1, max_length=255)
    ssn: str = Field(..., pattern=r"^\d{3}-?\d{2}-?\d{4}$")
    loan_amount_cents: int = Field(..., alias="loanAmountCents", gt=0)
    property_address: str = Field(..., alias="propertyAddress", min_length=1)
    loan_term_months: int = Field(..., alias="loanTermMonths", gt=0, le=600)

    model_config = {"populate_by_name": True}


class ApplicationSummary(BaseModel):
    id: UUID
    borrower_name: str = Field(..., alias="borrowerName")
    loan_amount_cents: int = Field(..., alias="loanAmountCents")
    property_address: str = Field(..., alias="propertyAddress")
    loan_term_months: int = Field(..., alias="loanTermMonths")
    interest_rate_bps: int | None = Field(None, alias="interestRateBps")
    status: ApplicationStatus
    created_at: datetime = Field(..., alias="createdAt")
    updated_at: datetime = Field(..., alias="updatedAt")

    model_config = {"populate_by_name": True, "from_attributes": True}


class DocumentSummary(BaseModel):
    id: UUID
    filename: str
    document_type: DocumentType | None = Field(None, alias="documentType")
    classification_confidence: float | None = Field(None, alias="classificationConfidence")
    uploaded_at: datetime = Field(..., alias="uploadedAt")

    model_config = {"populate_by_name": True, "from_attributes": True}


class AgentAnalysisSummary(BaseModel):
    status: str
    confidence: float | None = None
    recommendation: str | None = None
    summary: str | None = None
    findings: list[str] | None = None
    completed_at: datetime | None = Field(None, alias="completedAt")

    model_config = {"populate_by_name": True}


class ApplicationDetail(ApplicationSummary):
    escalation_reason: str | None = Field(None, alias="escalationReason")
    required_reviewer_role: str | None = Field(None, alias="requiredReviewerRole")
    documents: list[DocumentSummary] = []
    agent_analysis: dict[str, AgentAnalysisSummary | None] | None = Field(
        None, alias="agentAnalysis"
    )
    workflow_thread_id: str | None = Field(None, alias="workflowThreadId")


# --- Review ---

class ReviewRequest(BaseModel):
    decision: ReviewDecision
    rationale: str = Field(..., min_length=10)
    requested_documents: str | None = Field(
        None, alias="requestedDocuments", min_length=10
    )

    @field_validator("requested_documents")
    @classmethod
    def require_docs_description(cls, v: str | None, info) -> str | None:
        if info.data.get("decision") == ReviewDecision.REQUEST_MORE_DOCUMENTS and not v:
            raise ValueError(
                "requestedDocuments is required when decision is REQUEST_MORE_DOCUMENTS"
            )
        return v

    model_config = {"populate_by_name": True}


class ReviewResponse(BaseModel):
    application_id: UUID = Field(..., alias="applicationId")
    status: ApplicationStatus
    reviewed_by: str = Field(..., alias="reviewedBy")
    reviewed_at: datetime = Field(..., alias="reviewedAt")
    decision: ReviewDecision
    rationale: str

    model_config = {"populate_by_name": True}


class ReviewQueueItem(BaseModel):
    application_id: UUID = Field(..., alias="applicationId")
    borrower_name: str = Field(..., alias="borrowerName")
    escalation_reason: str | None = Field(None, alias="escalationReason")
    required_reviewer_role: str = Field(..., alias="requiredReviewerRole")
    escalated_at: datetime = Field(..., alias="escalatedAt")
    agent_summary: dict | None = Field(None, alias="agentSummary")

    model_config = {"populate_by_name": True}


# --- Audit ---

class AuditEventResponse(BaseModel):
    id: UUID
    application_id: UUID | None = Field(None, alias="applicationId")
    event_type: str = Field(..., alias="eventType")
    actor_type: str = Field(..., alias="actorType")
    actor_id: str = Field(..., alias="actorId")
    actor_role: str | None = Field(None, alias="actorRole")
    details: dict
    correlation_id: UUID | None = Field(None, alias="correlationId")
    created_at: datetime = Field(..., alias="createdAt")

    model_config = {"populate_by_name": True, "from_attributes": True}


# --- Configuration ---

class ThresholdUpdateRequest(BaseModel):
    auto_approve: float = Field(..., alias="autoApprove", gt=0, le=1)
    escalation: float = Field(..., alias="escalation", gt=0, le=1)
    denial: float = Field(..., alias="denial", gt=0, le=1)

    @field_validator("auto_approve")
    @classmethod
    def validate_ordering(cls, v: float, info) -> float:
        escalation = info.data.get("escalation")
        denial = info.data.get("denial")
        if escalation is not None and v <= escalation:
            raise ValueError(
                "Auto-approve threshold must be greater than escalation threshold"
            )
        if denial is not None and escalation is not None and escalation <= denial:
            raise ValueError(
                "Escalation threshold must be greater than denial threshold"
            )
        return v

    model_config = {"populate_by_name": True}


class FraudSensitivityUpdateRequest(BaseModel):
    level: FraudSensitivity


# --- Pagination ---

class PaginationMeta(BaseModel):
    next_cursor: str | None = Field(None, alias="nextCursor")
    has_more: bool = Field(..., alias="hasMore")

    model_config = {"populate_by_name": True}


# --- Health ---

class DependencyHealth(BaseModel):
    status: str
    latency_ms: int | None = Field(None, alias="latencyMs")

    model_config = {"populate_by_name": True}


class HealthResponse(BaseModel):
    status: str
    timestamp: datetime


class ReadinessResponse(HealthResponse):
    dependencies: dict[str, DependencyHealth]


# --- Data Envelope ---

from typing import Generic, TypeVar
T = TypeVar("T")


class DataEnvelope(BaseModel, Generic[T]):
    data: T


class PaginatedEnvelope(BaseModel, Generic[T]):
    data: list[T]
    pagination: PaginationMeta
```

### 2.3 Repository Interfaces (`packages/db/src/repositories/`)

Each repository is an async class injected via FastAPI `Depends`. Method signatures below are the contracts.

```python
# packages/db/src/repositories/application.py
class ApplicationRepository:
    async def create(
        self, *, borrower_name: str, ssn_encrypted: bytes, ssn_hash: str,
        loan_amount_cents: int, property_address: str, loan_term_months: int,
        created_by: UUID,
    ) -> Application: ...

    async def get_by_id(self, application_id: UUID) -> Application | None: ...

    async def list_applications(
        self, *, status: str | None = None, cursor: str | None = None, limit: int = 20,
    ) -> tuple[list[Application], str | None]: ...

    async def update_status(
        self, application_id: UUID, *, status: str,
        escalation_reason: str | None = None,
        required_reviewer_role: str | None = None,
        workflow_thread_id: UUID | None = None,
    ) -> Application: ...


# packages/db/src/repositories/document.py
class DocumentRepository:
    async def create(
        self, *, application_id: UUID, filename: str, content_type: str,
        file_size_bytes: int, storage_key: str, uploaded_by: UUID,
    ) -> Document: ...

    async def get_by_id(self, document_id: UUID) -> Document | None: ...

    async def list_by_application(self, application_id: UUID) -> list[Document]: ...

    async def update_classification(
        self, document_id: UUID, *, document_type: str, confidence: float,
    ) -> Document: ...

    async def update_extraction(
        self, document_id: UUID, *, extracted_data: dict, extraction_confidence: dict,
    ) -> Document: ...


# packages/db/src/repositories/audit.py
class AuditRepository:
    """Append-only. No update or delete methods exist."""
    async def create(
        self, *, application_id: UUID | None, event_type: str,
        actor_type: str, actor_id: str, actor_role: str | None = None,
        details: dict | None = None, correlation_id: UUID | None = None,
    ) -> AuditEvent: ...

    async def list_by_application(
        self, application_id: UUID, *, cursor: str | None = None, limit: int = 50,
    ) -> tuple[list[AuditEvent], str | None]: ...


# packages/db/src/repositories/agent_decision.py
class AgentDecisionRepository:
    async def create(
        self, *, application_id: UUID, agent_name: str, decision_type: str,
        confidence_score: float | None, recommendation: str | None,
        reasoning: str, findings: dict, model_used: str | None = None,
        prompt_tokens: int | None = None, completion_tokens: int | None = None,
        latency_ms: int | None = None, correlation_id: UUID,
        parent_step_id: UUID | None = None,
    ) -> AgentDecision: ...

    async def list_by_application(self, application_id: UUID) -> list[AgentDecision]: ...


# packages/db/src/repositories/user.py
class UserRepository:
    async def get_by_api_key_hash(self, key_hash: str) -> tuple[User, ApiKey] | None: ...
    async def get_by_id(self, user_id: UUID) -> User | None: ...


# packages/db/src/repositories/configuration.py
class ConfigurationRepository:
    async def get(self, key: str) -> dict | None: ...
    async def update(self, key: str, value: dict, updated_by: UUID) -> None: ...
```

### 2.4 Agent State TypedDicts (`packages/api/src/agents/state.py`)

These are defined in the architecture (Section 5.1) and reproduced here as the binding contract for all agent node implementations.

```python
# This project was developed with assistance from AI tools.
from typing import Any, TypedDict, Annotated
from langgraph.graph import add_messages


class ThresholdsSnapshot(TypedDict):
    auto_approve: float
    escalation: float
    denial: float


class DocumentResult(TypedDict):
    """Note: The `type` field values are governed by the `DocumentType` enum in
    Section 2.1, which is the authoritative contract. The architecture's TypedDict
    comment listed example values that have been superseded by this enum."""
    doc_id: str
    type: str  # Must be a valid DocumentType enum value (Section 2.1)
    confidence: float
    extracted_data: dict[str, Any]
    metadata: dict[str, Any]


class CreditAnalysisResult(TypedDict):
    confidence: float
    recommendation: str  # APPROVE, DENY, REVIEW
    credit_score: int
    debt_to_income: float
    findings: list[str]


class RiskAssessmentResult(TypedDict):
    confidence: float
    recommendation: str  # APPROVE, DENY, REVIEW
    risk_score: float
    risk_factors: list[str]
    ltv_ratio: float


class ComplianceCheckResult(TypedDict):
    confidence: float
    recommendation: str  # APPROVE, DENY, REVIEW
    violations: list[str]
    regulatory_references: list[str]


class FraudDetectionResult(TypedDict):
    confidence: float
    recommendation: str  # APPROVE, DENY, REVIEW
    flags: list[str]
    severity: str  # LOW, MEDIUM, HIGH


class AggregatedResults(TypedDict):
    final_recommendation: str  # APPROVE, DENY, REVIEW
    confidence: float
    agent_recommendations: dict[str, str]
    explanation: str


class CoachingOutput(TypedDict):
    suggestions: list[str]
    missing_documents: list[str]
    next_steps: list[str]


class PreviousAnalysis(TypedDict):
    resubmission_number: int
    credit_analysis: CreditAnalysisResult | None
    risk_assessment: RiskAssessmentResult | None
    compliance_check: ComplianceCheckResult | None
    fraud_detection: FraudDetectionResult | None


class LoanProcessingState(TypedDict):
    application_id: str
    correlation_id: str
    workflow_status: str

    documents: list[DocumentResult]
    document_processing_complete: bool

    credit_analysis: CreditAnalysisResult | None
    risk_assessment: RiskAssessmentResult | None
    compliance_check: ComplianceCheckResult | None
    fraud_detection: FraudDetectionResult | None

    aggregated_results: AggregatedResults | None
    consolidated_narrative: str | None
    has_disagreements: bool
    has_fraud_flags: bool

    routing_decision: str | None
    routing_rationale: str | None
    required_reviewer_role: str | None

    coaching_output: CoachingOutput | None

    thresholds: ThresholdsSnapshot
    resubmission_count: int
    previous_analyses: list[PreviousAnalysis]

    messages: Annotated[list, add_messages]


class IntakeState(TypedDict):
    session_id: str
    messages: Annotated[list, add_messages]
    sentiment: str | None
    tool_results: list[dict]
    citations: list[dict]
```

### 2.5 Agent Node Function Signatures

Every agent node is an async function with this signature pattern:

```python
async def {node_name}_node(state: LoanProcessingState) -> dict:
    """Process state and return a partial state update dict."""
    ...
```

The return dict contains only the keys being updated. LangGraph merges it into the full state.

### 2.6 Middleware Contracts

```python
# packages/api/src/middleware/correlation_id.py
# Reads X-Request-ID header or generates UUID v4.
# Sets correlation_id_var context var.
# Adds X-Request-ID to response headers.
correlation_id_var: contextvars.ContextVar[str]

# packages/api/src/middleware/auth.py
# Skips /health, /ready, /v1/public/* paths.
# Extracts Bearer token from Authorization header.
# Looks up SHA-256(token) in api_keys table (with Redis cache).
# Returns 401 on missing/invalid token.
# Attaches AuthenticatedUser to request.state.user:
class AuthenticatedUser(BaseModel):
    user_id: UUID
    name: str
    role: Role

# packages/api/src/middleware/rate_limit.py
# Applies to /v1/public/* paths only.
# Uses Redis INCR with TTL for session and IP counters.
# Returns 429 with X-RateLimit-* headers on violation.
```

### 2.7 Service Protocol Contracts

```python
# packages/api/src/services/protocols.py
from typing import Protocol

class CreditReport(TypedDict):
    credit_score: int
    payment_history: list[dict]
    derogatory_marks: int
    total_accounts: int
    utilization_ratio: float
    inquiries_last_6_months: int

class PropertyData(TypedDict):
    address: str
    avm_value_cents: int
    last_sale_date: str | None
    last_sale_price_cents: int | None
    comparable_sales: list[dict]

class CreditBureauService(Protocol):
    async def get_credit_report(self, ssn_hash: str) -> CreditReport: ...

class PropertyDataService(Protocol):
    async def get_property_data(self, address: str) -> PropertyData: ...
```

### 2.8 Error Response Contract (RFC 7807)

All error responses use `ProblemDetail`:

```python
# packages/api/src/models/schemas.py (additional)
class ProblemDetail(BaseModel):
    type: str
    title: str
    status: int
    detail: str | None = None
    instance: str | None = None
    errors: list[dict] | None = None
```

Error type base URL: `https://mortgage-quickstart.example.com/errors/`

### 2.9 SQLAlchemy ORM Models (`packages/db/src/models.py`)

The ORM models map 1:1 to the DDL in architecture Section 3.1. The binding contract is the column names and types. All models use:
- `UUID` primary keys via `gen_random_uuid()`
- `TIMESTAMPTZ` for all timestamps
- `BIGINT` for monetary values (integer cents)
- `JSONB` for structured/flexible fields

The complete ORM model file is created in P1-WU03.

---

## 3. Phase 1: Foundation

**Note on file additions:** This TD introduces files not listed in the architecture's Section 9 file tree where the implementation requires dedicated modules: `packages/api/src/errors.py` (RFC 7807 error handling hierarchy), `packages/api/src/middleware/authorization.py` (RBAC dependency), `packages/api/src/middleware/request_limits.py` (request size enforcement). These are TD design decisions that extend the architecture's file structure.

**Goal:** Working application lifecycle with stub agents, real auth, real audit trail, complete graph structure.

**User stories covered:** US-001, US-002, US-006, US-008, US-021, US-053, US-054, US-058, US-059, US-060, US-064, US-065, US-067, US-068, US-069, US-073, US-081

### Dependency Graph

```
P1-WU01 (scaffolding)
  |
  +---> P1-WU02 (compose + Makefile)
  |       |
  |       +---> P1-WU03 (ORM models)
  |       |       |
  |       |       +---> P1-WU04 (migrations)
  |       |               |
  |       |               +---> P1-WU05 (seed data)
  |       |               |       |
  |       |               |       +---> P1-WU10 (auth middleware)
  |       |               |       |       |
  |       |               |       |       +---> P1-WU11 (RBAC)
  |       |               |       |
  |       |               +---> P1-WU06a (core DB repositories)
  |       |                       |
  |       |                       +---> P1-WU06b (supporting DB repositories)
  |       |                       |
  |       |                       +---> P1-WU09 (correlation ID + logging middleware)
  |       |                       |       |
  |       |                       |       +---> P1-WU12 (application CRUD routes)
  |       |                       |       |       |
  |       |                       |       |       +---> P1-WU13 (document upload route)
  |       |                       |       |       |       |
  |       |                       |       |       |       +---> P1-WU15 (audit trail route)
  |       |                       |       |       |               |
  |       |                       |       |       |               +---> P1-WU16 (LangGraph stubs)
  |       |                       |       |       |                       |
  |       |                       |       |       |                       +---> P1-WU17 (submit endpoint)
  |       |                       |       |       |                               |
  |       |                       |       |       |                               +---> P1-WU18 (Phase 1 integration test)
  |       |                       |       |
  |       |                       +---> P1-WU07 (settings + deps)
  |       |                       |
  |       |                       +---> P1-WU08 (health routes + error handling)
  |       |
  |       +---> P1-WU14 (MinIO client)
  |
  +---> P1-WU19 (README + .env.example)
```

### P1-WU01: Project Scaffolding -- Package Structure

**What to build:** Create the monorepo package skeleton for `packages/api`, `packages/ui`, and `packages/db` with their respective build configuration files.

**Satisfies:** US-068 (partial)

**Files to create:**
- `packages/api/pyproject.toml`
- `packages/api/src/__init__.py`
- `packages/db/pyproject.toml`
- `packages/db/src/__init__.py`
- `packages/ui/package.json`

**Files to read for context:**
- `plans/architecture.md` Section 9 (File Structure)
- `.claude/rules/architecture.md` (monorepo conventions)

**Task description:**

Create the three package directories with their build configuration:

1. `packages/api/pyproject.toml` -- Python package using `uv`, with dependencies: `fastapi>=0.115`, `uvicorn[standard]>=0.34`, `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.30`, `pydantic>=2.10`, `pydantic-settings>=2.7`, `langchain-core>=0.3`, `langchain-anthropic>=0.3`, `langchain-openai>=0.3`, `langgraph>=0.2`, `langgraph-checkpoint-postgres>=2.0`, `minio>=7.2`, `redis>=5.2`, `tenacity>=9.0`, `python-multipart>=0.0.18`, `cryptography>=44.0`. Dev dependencies: `pytest>=8.3`, `pytest-asyncio>=0.24`, `httpx>=0.28`, `ruff>=0.9`. Set `requires-python = ">=3.12"`. Package name: `mortgage-api`.

2. `packages/db/pyproject.toml` -- Python package using `uv`, with dependencies: `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.30`, `alembic>=1.14`, `pgvector>=0.3`. Dev dependencies: `pytest>=8.3`, `pytest-asyncio>=0.24`, `ruff>=0.9`. Package name: `mortgage-db`.

3. `packages/ui/package.json` -- React 19 app with Vite. Dependencies: `react@^19`, `react-dom@^19`, `@tanstack/react-router@^1`, `@tanstack/react-query@^5`, `tailwindcss@^4`. Dev dependencies: `vite@^6`, `@vitejs/plugin-react@^4`, `typescript@^5.7`, `vitest@^3`, `@playwright/test@^1.50`.

4. Create `packages/api/src/__init__.py` and `packages/db/src/__init__.py` with the AI compliance comment.

**Implementing agent:** `devops-engineer`

**Exit condition:**
```bash
cd packages/api && uv sync && python -c "import src; print('api importable')" && \
cd ../../packages/db && uv sync && python -c "import src; print('db importable')" && \
cd ../../packages/ui && pnpm install
```

---

### P1-WU02: Docker Compose, Makefile, and Turbo Config

**What to build:** Container composition for local development (PostgreSQL with pgvector, Redis, MinIO) plus the Makefile with standard commands and Turborepo config.

**Satisfies:** US-068 (partial)

**Files to create:**
- `compose.yml`
- `Makefile`
- `turbo.json`

**Files to read for context:**
- `plans/architecture.md` Section 7.2 (Docker Compose), Section 7.3 (Startup Order)
- `packages/api/pyproject.toml` (from P1-WU01)

**Task description:**

1. `compose.yml` -- Define services per architecture Section 7.2:
   - `postgres`: image `pgvector/pgvector:pg16`, port 5432, env `POSTGRES_DB=mortgage`, `POSTGRES_USER=postgres`, `POSTGRES_PASSWORD=postgres`, volume `pgdata`, healthcheck via `pg_isready`.
   - `redis`: image `redis:7-alpine`, port 6379, command `--requirepass ${REDIS_PASSWORD:-devpassword}`, healthcheck via `redis-cli -a ${REDIS_PASSWORD:-devpassword} ping`. Add `REDIS_PASSWORD` to `.env.example` in P1-WU19.
   - `minio`: image `minio/minio`, ports 9000/9001, env `MINIO_ROOT_USER=minioadmin`, `MINIO_ROOT_PASSWORD=minioadmin`, command `server /data --console-address ":9001"`, healthcheck via `mc ready local`.
   - Do NOT include `api` or `ui` services -- those run natively during development via `make dev`.

2. `Makefile` targets:
   - `setup`: runs `uv sync` in api and db packages, `pnpm install` in ui, `docker compose up -d` for infra, waits for health, runs `make db-upgrade`, runs `make db-seed`.
   - `dev`: starts API (`uvicorn`) and UI (`pnpm dev`) in parallel.
   - `test`: runs `pytest` in api and db, `pnpm test` in ui.
   - `lint`: runs `ruff check` in api and db, `pnpm lint` in ui.
   - `db-upgrade`: runs `alembic upgrade head` in db package.
   - `db-rollback`: runs `alembic downgrade -1` in db package.
   - `db-seed`: runs seed script (placeholder).
   - `containers-up`: `docker compose up -d`.
   - `containers-down`: `docker compose down`.

3. `turbo.json` -- Pipeline for `build`, `test`, `lint`, `dev` tasks.

**Implementing agent:** `devops-engineer`

**Exit condition:**
```bash
make containers-up && sleep 10 && docker compose ps --format '{{.Service}} {{.Status}}' | grep -c "healthy"
# Expected: 3 (postgres, redis, minio all healthy)
```

---

### P1-WU03: SQLAlchemy ORM Models

**What to build:** ORM model classes mapping to all tables defined in architecture Section 3.1.

**Satisfies:** US-073 (partial)

**Files to create:**
- `packages/db/src/models.py`
- `packages/db/src/engine.py`

**Files to read for context:**
- `plans/architecture.md` Section 3.1 (Database Schema)
- Interface contract Section 2.1 (Enums)

**Task description:**

1. `packages/db/src/engine.py`:
   - Create `create_engine` function that returns an `AsyncEngine` from `DATABASE_URL` env var with pool_size=5, max_overflow=15.
   - Create `async_session_factory` using `async_sessionmaker`.
   - Create `get_session` async generator for FastAPI dependency injection.

2. `packages/db/src/models.py`:
   - Define SQLAlchemy 2.0 declarative models for all tables: `Application`, `Document`, `AgentDecision`, `AuditEvent`, `User`, `ApiKey`, `Configuration`, `Embedding`.
   - Use `mapped_column` with type annotations.
   - All UUIDs use `postgresql.UUID` type with `server_default=func.gen_random_uuid()`.
   - All timestamps use `TIMESTAMP(timezone=True)` with `server_default=func.now()`.
   - Monetary values use `BigInteger`.
   - JSONB fields use `postgresql.JSONB`.
   - Define relationships: `Application.documents`, `Application.agent_decisions`, `Application.audit_events`, `User.api_keys`.
   - `AuditEvent` has no relationship back to allow updates -- it is designed for append-only access.

**Implementing agent:** `database-engineer`

**Exit condition:**
```bash
cd packages/db && python -c "from src.models import Application, Document, AuditEvent, User, ApiKey, AgentDecision, Configuration, Embedding; print('All models imported successfully')"
```

---

### P1-WU04: Alembic Migrations -- Initial Schema

**What to build:** Alembic configuration and the initial migration that creates all tables, indexes, and database roles.

**Satisfies:** US-073, US-021 (partial -- immutability grants)

**Files to create:**
- `packages/db/alembic.ini`
- `packages/db/alembic/env.py`
- `packages/db/alembic/versions/001_initial_schema.py`

**Files to read for context:**
- `packages/db/src/models.py` (from P1-WU03)
- `plans/architecture.md` Section 3.1 (DDL), Section 3.2 (Database Roles)

**Task description:**

1. `packages/db/alembic.ini` -- Standard config pointing to `alembic/` directory. `sqlalchemy.url` reads from `DATABASE_URL` env var.

2. `packages/db/alembic/env.py` -- Async Alembic env using `asyncpg`. Import `Base.metadata` from `models.py` for autogenerate support.

3. `packages/db/alembic/versions/001_initial_schema.py`:
   - `upgrade()`: Create all tables from architecture Section 3.1 DDL (applications, documents, agent_decisions, audit_events, users, api_keys, configuration, embeddings). Create all indexes. Create `app_role` and `migration_role` database roles. Apply grants: `REVOKE UPDATE, DELETE ON audit_events FROM app_role; GRANT SELECT, INSERT ON audit_events TO app_role;`. Grant app_role full CRUD on all other tables.
   - `downgrade()`: Drop all tables, drop roles.
   - Enable pgvector extension: `CREATE EXTENSION IF NOT EXISTS vector;`

**Implementing agent:** `database-engineer`

**Exit condition:**
```bash
cd packages/db && alembic upgrade head && alembic downgrade base && alembic upgrade head && echo "Migration idempotent: OK"
```

---

### P1-WU05: Seed Data Migration

**What to build:** Seed migration that creates default users, API keys, and configuration.

**Satisfies:** US-058 (partial -- default keys), US-060 (partial -- keys to detect)

**Files to create:**
- `packages/db/alembic/versions/002_seed_data.py`

**Files to read for context:**
- `plans/architecture.md` Section 4.1 (API Key Generation), Section 10.4
- `packages/db/src/models.py` (from P1-WU03)

**Task description:**

Create Alembic migration `002_seed_data.py`:

`upgrade()`:
1. Insert three users: `Demo Loan Officer` (role: `loan_officer`), `Demo Senior Underwriter` (role: `senior_underwriter`), `Demo Reviewer` (role: `reviewer`).
2. Generate three API keys using `secrets.token_urlsafe(32)` with prefix `mq_lo_`, `mq_su_`, `mq_rv_`. Store SHA-256 hash and 8-char prefix. Set `is_default=true`. Print full keys to stdout during migration (one-time display).
3. Insert default configuration rows:
   - `confidence_thresholds`: `{"auto_approve": 0.85, "escalation": 0.60, "denial": 0.40}`
   - `fraud_sensitivity`: `{"level": "MEDIUM"}`
   - `rate_limits`: `{"chat_per_hour": 20, "property_per_session": 5, "calc_per_hour": 100}`

`downgrade()`: Delete seed rows.

**Important:** The generated API keys must be deterministic in development (use a fixed seed for `secrets` in dev mode so keys are reproducible across `make setup` runs). Store the well-known dev keys in `.env.example` so developers can use them immediately.

**Implementing agent:** `database-engineer`

**Exit condition:**
```bash
cd packages/db && alembic upgrade head && python -c "
import asyncio, asyncpg, os
async def check():
    conn = await asyncpg.connect(os.environ.get('DATABASE_URL', 'postgresql://postgres:postgres@localhost:5432/mortgage'))
    users = await conn.fetch('SELECT name, role FROM users')
    keys = await conn.fetch('SELECT key_prefix, is_default FROM api_keys')
    config = await conn.fetch('SELECT key FROM configuration')
    assert len(users) == 3, f'Expected 3 users, got {len(users)}'
    assert len(keys) == 3, f'Expected 3 keys, got {len(keys)}'
    assert len(config) >= 2, f'Expected >= 2 config rows, got {len(config)}'
    print('Seed data verified: OK')
    await conn.close()
asyncio.run(check())
"
```

---

### P1-WU06a: Core Database Repositories

**What to build:** Core repository classes (application, document, audit) implementing the contracts from Section 2.3.

**Satisfies:** US-053 (partial), US-054 (partial), US-021 (partial)

**Files to create:**
- `packages/db/src/repositories/__init__.py`
- `packages/db/src/repositories/application.py`
- `packages/db/src/repositories/document.py`
- `packages/db/src/repositories/audit.py`

**Files to read for context:**
- Interface contract Section 2.3 (Repository Interfaces)
- `packages/db/src/models.py` (from P1-WU03)

**Task description:**

Implement core repository classes per the signatures in Section 2.3. Key implementation details:

1. `ApplicationRepository`: Cursor-based pagination using `updated_at` + `id` as cursor components. Encode as base64 JSON.
2. `AuditRepository`: Only `create` and `list_by_application`. No update or delete methods. This is enforced at both the code level (no methods) and the database level (grants from P1-WU04).

Each repository takes an `AsyncSession` in its constructor.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/db && pytest tests/test_repositories_core.py -v
```

Note: The test file `tests/test_repositories_core.py` must be written as part of this WU with at least one test per repository method (happy path). Use an in-memory or test database.

---

### P1-WU06b: Supporting Database Repositories

**What to build:** Supporting repository classes (agent_decision, user, configuration) implementing the contracts from Section 2.3.

**Satisfies:** US-053 (partial), US-054 (partial)

**Depends on:** P1-WU06a (for `__init__.py`)

**Files to create:**
- `packages/db/src/repositories/agent_decision.py`
- `packages/db/src/repositories/user.py`
- `packages/db/src/repositories/configuration.py`

**Files to read for context:**
- Interface contract Section 2.3 (Repository Interfaces)
- `packages/db/src/models.py` (from P1-WU03)
- `packages/db/src/repositories/__init__.py` (from P1-WU06a)

**Task description:**

Implement supporting repository classes per the signatures in Section 2.3. Key implementation details:

1. `UserRepository.get_by_api_key_hash`: Join `api_keys` and `users` tables. Filter by `key_hash`, `revoked_at IS NULL`, and `expires_at IS NULL OR expires_at > now()`.
2. `ConfigurationRepository`: Simple key-value CRUD on the configuration table.

Each repository takes an `AsyncSession` in its constructor.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/db && pytest tests/test_repositories_supporting.py -v
```

Note: The test file `tests/test_repositories_supporting.py` must be written as part of this WU with at least one test per repository method (happy path). Use an in-memory or test database.

---

### P1-WU07: Application Settings and Dependency Injection

**What to build:** Pydantic Settings configuration and FastAPI dependency injection setup.

**Satisfies:** US-068 (partial)

**Files to create:**
- `packages/api/src/settings.py`
- `packages/api/src/dependencies.py`

**Files to read for context:**
- `plans/architecture.md` Appendix A (Environment Variables)
- `plans/architecture.md` Section 2.6 (Mocked service pattern)

**Task description:**

1. `packages/api/src/settings.py`:
   ```python
   class Settings(BaseSettings):
       DATABASE_URL: str
       REDIS_URL: str = "redis://localhost:6379/0"
       MINIO_ENDPOINT: str = "localhost:9000"
       MINIO_ACCESS_KEY: str = "minioadmin"
       MINIO_SECRET_KEY: str = "minioadmin"
       ANTHROPIC_API_KEY: str = ""
       OPENAI_API_KEY: str = ""
       SSN_ENCRYPTION_KEY: str  # Required, validated on startup
       FRED_API_KEY: str = ""
       BATCHDATA_API_KEY: str = ""
       LANGFUSE_PUBLIC_KEY: str = ""
       LANGFUSE_SECRET_KEY: str = ""
       LOG_LEVEL: str = "INFO"
       API_HOST: str = "0.0.0.0"
       API_PORT: int = 8000
       CREDIT_BUREAU_PROVIDER: str = "mock"
       CORS_ORIGINS: str = "http://localhost:3000"

       model_config = SettingsConfigDict(env_file=".env")
   ```
   Validate `SSN_ENCRYPTION_KEY` is base64-encoded and at least 32 bytes. Refuse to start if invalid.

2. `packages/api/src/dependencies.py`:
   - `get_db_session()` -- yields `AsyncSession` from engine.
   - `get_settings()` -- returns singleton `Settings`.
   - `get_application_repo(session)` -- returns `ApplicationRepository`.
   - `get_audit_repo(session)` -- returns `AuditRepository`.
   - `get_document_repo(session)` -- returns `DocumentRepository`.
   - `get_agent_decision_repo(session)` -- returns `AgentDecisionRepository`.
   - `get_user_repo(session)` -- returns `UserRepository`.
   - `get_config_repo(session)` -- returns `ConfigurationRepository`.
   - `get_credit_bureau_service()` -- returns `MockCreditBureauService` or real based on setting.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && python -c "
from src.settings import Settings
from src.dependencies import get_settings, get_application_repo
s = Settings(_env_file='.env.test')
print(f'Settings loaded: {s.API_PORT}')
print('Dependency functions importable')
"
```

---

### P1-WU08: Health Check Routes and Error Handling

**What to build:** Health/readiness endpoints and the RFC 7807 error handling framework.

**Satisfies:** US-067

**Files to create:**
- `packages/api/src/routes/health.py`
- `packages/api/src/errors.py`

**Files to read for context:**
- `plans/architecture.md` Section 7.4 (Health Check Strategy)
- `plans/architecture.md` Section 4.3 (Error Response Format)
- `.claude/rules/error-handling.md`

**Task description:**

1. `packages/api/src/routes/health.py`:
   - `GET /health` -- Returns 200 `{"status": "ok", "timestamp": "..."}` always.
   - `GET /ready` -- Checks database (async `SELECT 1`), Redis (`PING`), MinIO (list buckets). Returns 200 with per-dependency status or 503 if any critical dependency fails. Include latency_ms for each.

2. `packages/api/src/errors.py`:
   - Define `AppError` base class with `status_code`, `error_type`, `title`, `detail`.
   - Define subclasses: `ValidationError` (422), `NotFoundError` (404), `AuthenticationError` (401), `AuthorizationError` (403), `ConflictError` (409), `RateLimitError` (429), `ServiceUnavailableError` (503).
   - Register FastAPI exception handlers that convert `AppError` subclasses to RFC 7807 JSON responses.
   - Register handler for Pydantic `ValidationError` that converts to 422 with `errors` array.
   - PII fields (`ssn`) in validation errors must show `[REDACTED]` not the actual value.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_health.py tests/unit/test_errors.py -v
```

---

### P1-WU09: Correlation ID and Structured Logging Middleware

**What to build:** Request correlation ID generation/propagation and structured JSON logging.

**Satisfies:** US-064, US-065

**Files to create:**
- `packages/api/src/middleware/__init__.py`
- `packages/api/src/middleware/correlation_id.py`
- `packages/api/src/middleware/logging.py`
- `packages/api/src/middleware/request_limits.py`

**Files to read for context:**
- `plans/architecture.md` Section 8.1 (Structured Logging), Section 8.2 (Correlation ID)
- `.claude/rules/observability.md`

**Task description:**

1. `packages/api/src/middleware/correlation_id.py`:
   - Define `correlation_id_var: contextvars.ContextVar[str]`.
   - Middleware reads `X-Request-ID` header or generates `uuid4()`.
   - Sets `correlation_id_var`, attaches to `request.state.correlation_id`.
   - Adds `X-Request-ID` to response headers.

2. `packages/api/src/middleware/logging.py`:
   - Configure `structlog` for JSON output to stdout.
   - Log request start: method, path, correlation_id.
   - Log request completion: status_code, duration_ms, correlation_id.
   - Never log request/response bodies.
   - Never log PII fields (SSN, financial data).
   - Include `service: "api"` in all entries.

Correlation IDs are request-scoped (new UUID per request) and used ONLY for tracing/logging. They are never used for session management or authorization. Session IDs (Phase 3b) are a separate concept with different scope and Redis keys.

3. Request size limit enforcement:
   - JSON request bodies: 1 MB maximum.
   - Multipart requests: 50 MB total.
   - Return 413 Payload Too Large (RFC 7807 format) if limits exceeded.
   - File upload limit (10 MB per file) is enforced in P1-WU13 upload handler.
   - Request limits logic is included in the middleware module (`packages/api/src/middleware/request_limits.py`).

4. Security headers middleware:
   - Add to all responses: X-Content-Type-Options: nosniff, X-Frame-Options: DENY, Referrer-Policy: strict-origin-when-cross-origin.
   - HSTS and CSP are production-only (controlled by ENVIRONMENT setting); skip in development.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_middleware.py -v -k "correlation or logging or request_limit or security_header"
# Test: 2 MB JSON body returns 413
# Test: security headers present in response
```

---

### P1-WU10: Authentication Middleware

**What to build:** API key authentication middleware that extracts Bearer token, hashes it, looks up the user, and attaches identity to request state.

**Satisfies:** US-058

**Files to create:**
- `packages/api/src/middleware/auth.py`
- `packages/api/src/services/encryption.py`

**Files to read for context:**
- `plans/architecture.md` Section 4.1 (Authentication Flow), Section 6.1 (Authentication)
- Interface contract Section 2.6 (Middleware Contracts)

**Task description:**

1. `packages/api/src/middleware/auth.py`:
   - Skip paths: `/health`, `/ready`, any path starting with `/v1/public/`.
   - Extract `Authorization: Bearer <token>` header.
   - Compute `SHA-256(token)`.
   - Look up in `api_keys` table via `UserRepository.get_by_api_key_hash()`.
   - On success: attach `AuthenticatedUser(user_id, name, role)` to `request.state.user`.
   - On failure: return 401 RFC 7807 response.
   - Log auth success (INFO) and failure (WARN) per architecture Section 8.1.

2. `packages/api/src/services/encryption.py`:
   - `hash_api_key(key: str) -> str` -- SHA-256 hex digest.
   - `encrypt_ssn(ssn: str, key: bytes) -> bytes` -- AES-256-GCM encryption.
   - `decrypt_ssn(ciphertext: bytes, key: bytes) -> str` -- AES-256-GCM decryption.
   - Validate encryption key length on init.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_auth.py tests/unit/test_encryption.py -v
```

---

### P1-WU11: RBAC Middleware

**What to build:** Role-based authorization middleware.

**Satisfies:** US-059

**Files to create:**
- `packages/api/src/middleware/authorization.py`

**Files to read for context:**
- `plans/architecture.md` Section 6.2 (Role Hierarchy)
- Interface contract Section 2.1 (Role enum)

**Task description:**

1. `packages/api/src/middleware/authorization.py`:
   - `require_role(minimum: Role)` -- FastAPI dependency that reads `request.state.user` and checks `Role[user.role.upper()] >= minimum`. Returns 403 if insufficient.
   - Role hierarchy: `LOAN_OFFICER(1) < SENIOR_UNDERWRITER(2) < REVIEWER(3)`.

Note: The default API key startup warning (US-060) is handled in P1-WU18 where `main.py` is created.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_authorization.py -v
# Test: loan_officer cannot access reviewer endpoint (403)
# Test: reviewer can access loan_officer endpoint (200)
```

---

### P1-WU12: Application CRUD Routes

**What to build:** API routes for creating, viewing, and listing loan applications.

**Satisfies:** US-053, US-054

**Files to create:**
- `packages/api/src/routes/applications.py`

**Files to read for context:**
- `plans/architecture.md` Section 4.2 (Request/Response Shapes)
- Interface contracts Section 2.2 (Schemas), Section 2.3 (Repositories)

**Task description:**

1. `POST /v1/applications` -- Create application.
   - Requires auth (any role).
   - Validates `CreateApplicationRequest`.
   - Encrypts SSN via `encrypt_ssn()`.
   - Stores SHA-256 hash of SSN for lookups.
   - Creates application record via `ApplicationRepository.create()`.
   - Creates audit event `APPLICATION_CREATED`.
   - Returns 201 with `ApplicationSummary` in data envelope.
   - Sets `Location` header.
   - SSN is NEVER returned in response.

2. `GET /v1/applications/{id}` -- Get application detail.
   - Requires auth (any role).
   - Returns `ApplicationDetail` with documents and agent analysis (if available).
   - Returns 404 if not found.

3. `GET /v1/applications` -- List applications.
   - Requires auth (any role).
   - Query params: `status` (optional filter), `cursor`, `limit` (default 20, max 100).
   - Returns paginated list of `ApplicationSummary`.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_application_crud.py -v
# Test: create application returns 201
# Test: get application by ID returns detail
# Test: list applications with pagination
# Test: create application records audit event
```

---

### P1-WU13: Document Upload Route

**What to build:** API route for uploading documents to an application.

**Satisfies:** US-008

**Files to create:**
- `packages/api/src/routes/documents.py`

**Files to read for context:**
- `plans/architecture.md` Section 2.5 (Object Storage)
- Interface contracts Section 2.2 (DocumentSummary), Section 2.3 (DocumentRepository)

**Task description:**

1. `POST /v1/applications/{id}/documents` -- Upload document.
   - Requires auth (any role).
   - Accepts multipart file upload.
   - Validates file type: PDF only (`application/pdf`). Returns 400 for other types.
   - Validates file size: max 10MB. Returns 413 if exceeded.
   - Validates application exists and is in DRAFT or AWAITING_DOCUMENTS status. Returns 409 otherwise.
   - Stores file in MinIO at `mortgage-documents/{application_id}/{document_id}/{filename}`.
   - Creates document record via `DocumentRepository.create()`.
   - Creates audit event `DOCUMENTS_UPLOADED`.
   - Returns 201 with `DocumentSummary`.

2. `GET /v1/applications/{id}/documents` -- List documents.
   - Requires auth (any role).
   - Returns list of `DocumentSummary` for the application.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_document_upload.py -v
# Test: upload PDF returns 201
# Test: upload non-PDF returns 400
# Test: upload to non-DRAFT application returns 409
# Test: document appears in list after upload
```

---

### P1-WU14: MinIO Client Service

**What to build:** MinIO client wrapper for document upload, download, and presigned URL generation.

**Satisfies:** US-008 (partial -- storage layer)

**Files to create:**
- `packages/api/src/services/minio_client.py`

**Files to read for context:**
- `plans/architecture.md` Section 2.5 (Object Storage)

**Task description:**

Create `MinIOService` class:
- `__init__(endpoint, access_key, secret_key)` -- Initialize MinIO client. Create `mortgage-documents` bucket if not exists.
- `async upload_file(bucket, key, data, content_type, size) -> str` -- Upload file, return storage key.
- `async download_file(bucket, key) -> bytes` -- Download file content.
- `async get_presigned_url(bucket, key, expires_seconds=900) -> str` -- Generate presigned download URL (15-minute expiry).
- `async file_exists(bucket, key) -> bool` -- Check if object exists.

Note: MinIO's Python SDK is synchronous. Wrap calls with `asyncio.to_thread()` for async compatibility.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_minio_client.py -v
# Test: upload and download round-trip
# Test: bucket creation on init
# Test: presigned URL generation
```

---

### P1-WU15: Audit Trail Route

**What to build:** API route for querying audit events for an application.

**Satisfies:** US-021

**Files to create:**
- `packages/api/src/routes/audit.py`

**Files to read for context:**
- Interface contracts Section 2.2 (AuditEventResponse), Section 2.3 (AuditRepository)

**Task description:**

1. `GET /v1/applications/{id}/audit` -- List audit events.
   - Requires auth (any role).
   - Returns paginated list of `AuditEventResponse`.
   - Sorted by `created_at` ascending (chronological order).
   - Query params: `cursor`, `limit` (default 50).

2. No PATCH, PUT, or DELETE endpoints for audit events. This is intentional -- audit records are immutable.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_audit_trail.py -v
# Test: audit events returned in chronological order
# Test: no PATCH/PUT/DELETE endpoints exist for audit
```

---

### P1-WU16: LangGraph Graph Definition with Stub Nodes

**What to build:** Complete Loan Processing Graph with all nodes defined as stubs, plus PostgresSaver checkpoint configuration.

**Satisfies:** US-001, US-002, US-006

**Files to create:**
- `packages/api/src/agents/state.py`
- `packages/api/src/agents/graphs/loan_processing.py`
- `packages/api/src/agents/nodes/supervisor.py`
- `packages/api/src/agents/nodes/stubs.py`
- `packages/api/src/agents/checkpointer.py`

**Files to read for context:**
- `plans/architecture.md` Section 5.1 (Loan Processing Graph), Section 5.5 (Checkpoint Persistence)
- Interface contract Section 2.4 (Agent State TypedDicts)

**Task description:**

1. `packages/api/src/agents/state.py` -- Copy the TypedDicts from Section 2.4 verbatim.

2. `packages/api/src/agents/nodes/supervisor.py`:
   - `supervisor_init_node(state)` -- Sets `workflow_status` to `PROCESSING`, records audit event, returns partial state.
   - `aggregation_node(state)` -- Collects non-None agent results, identifies disagreements, sets `has_disagreements` and `has_fraud_flags`. Returns aggregated results.
   - `routing_decision_node(state)` -- Implements confidence-based routing logic from architecture Section 5.4. Returns routing decision.
   - `human_review_wait_node(state)` -- Stub that returns current state (review is handled via API, not within the graph).

3. `packages/api/src/agents/nodes/stubs.py`:
   - Stub implementations for: `document_processing_node`, `credit_analysis_node`, `risk_assessment_node`, `compliance_check_node`, `fraud_detection_node`, `denial_coaching_node`.
   - Each returns hardcoded synthetic results per architecture Section 5.1 stub example.
   - Include inline comments explaining that these are stubs replaced in later phases.

4. `packages/api/src/agents/graphs/loan_processing.py`:
   - Complete graph definition per architecture Section 5.1.
   - All nodes, edges, conditional edges.
   - `route_after_documents`, `route_after_decision`, `route_after_review` routing functions.
   - `compile_loan_processing_graph(checkpointer)` function that returns the compiled graph.

5. `packages/api/src/agents/checkpointer.py`:
   - `create_checkpointer(database_url)` -- Creates `AsyncPostgresSaver` with separate connection pool (min 2, max 5).
   - Calls `checkpointer.setup()` to create checkpoint tables.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_graph_definition.py -v
# Test: graph compiles without error
# Test: stub nodes return expected TypedDict shapes
# Test: routing logic returns correct decisions for high/medium/low confidence
```

---

### P1-WU17: Application Submit Endpoint

**What to build:** The endpoint that submits an application for processing, triggering the LangGraph workflow.

**Satisfies:** US-001 (partial), US-002 (partial)

**Files to modify:**
- `packages/api/src/routes/applications.py` (add submit route)

**Files to read for context:**
- `packages/api/src/agents/graphs/loan_processing.py` (from P1-WU16)
- `plans/architecture.md` Section 3.4 (Data Flow)

**Task description:**

Add `POST /v1/applications/{id}/submit` route:
- Requires auth (any role).
- Validates application exists and is in `DRAFT` status. Returns 409 if not.
- Updates status to `PROCESSING`.
- Creates a LangGraph thread with ID `loan:{application_id}`.
- Invokes the compiled graph with initial state containing `application_id`, `correlation_id`, `workflow_status="PROCESSING"`, and empty agent result fields.
- The graph runs synchronously within the request (MVP -- no background workers).
- After graph completion, the final status (AUTO_APPROVED, ESCALATED, or DENIED) is written to the application record.
- Creates audit events for each workflow transition.
- Returns 200 with updated `ApplicationDetail`.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_submit_workflow.py -v
# Test: submit DRAFT application -> runs through stubs -> reaches AUTO_APPROVED
# Test: submit non-DRAFT application returns 409
# Test: audit events created for each workflow step
# Test: checkpoint written to database
```

---

### P1-WU18: Phase 1 Integration Test and FastAPI App Factory

**What to build:** The FastAPI application factory that wires everything together, plus end-to-end integration test.

**Satisfies:** US-068 (final), US-060, US-071 (partial), US-081 (partial)

**Files to create:**
- `packages/api/src/main.py`
- `packages/api/tests/conftest.py`
- `packages/api/tests/integration/test_phase1_e2e.py`

**Files to read for context:**
- All P1-WU files created so far
- `plans/architecture.md` Section 2.1 (Middleware Stack)

**Task description:**

1. `packages/api/src/main.py`:
   - `create_app()` factory function.
   - Register middleware in correct order: CORS, correlation ID, logging, auth.
   - Include all routers: health, applications, documents, audit.
   - Startup event: run migrations (or verify), create MinIO bucket, init checkpointer.
   - Startup event -- default key warning (US-060): On startup, query `api_keys` table for any row with `is_default=true` and `revoked_at IS NULL`. If found, log at ERROR level: `"Running with default API keys. Change keys before deploying to production."`
   - Shutdown event: close connection pools.
   - Include AI compliance comment at top of file.
   - Include inline comments explaining the middleware stack ordering and why.

2. `packages/api/tests/conftest.py`:
   - Test fixtures: test database setup/teardown, test client (httpx AsyncClient), test API key fixtures (pre-seeded for each role), mock MinIO.

3. `packages/api/tests/integration/test_phase1_e2e.py`:
   - End-to-end test: Create application -> Upload document -> Submit -> Verify status is AUTO_APPROVED (stubs produce high confidence) -> Verify audit trail has all expected events -> Verify checkpoint exists in database.
   - Test restart resilience: Submit, stop (rollback state), resume from checkpoint.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_phase1_e2e.py -v
# All tests pass: full lifecycle with stubs, audit trail complete, checkpoint persisted
# Test: startup logs warning when default keys exist
# Test: application submission completes within 30 seconds (with stub agents) (US-071)
```

---

### P1-WU19: README, .env.example, and Developer Docs

**What to build:** Developer-facing documentation for getting started.

**Satisfies:** US-069

**Files to create:**
- `.env.example`

**Files to modify:**
- `README.md`

**Files to read for context:**
- `plans/architecture.md` Appendix A (Environment Variables), Section 1.1 (diagram)
- `plans/product-plan.md` (Executive Summary)

**Task description:**

1. `.env.example` -- All environment variables from architecture Appendix A with comments explaining each. Include the well-known dev API keys from P1-WU05. Include `REDIS_PASSWORD` variable. Include LangFuse env vars with warning comment: `# WARNING: Enabling LangFuse logs full prompts/completions. Only use with synthetic/test data, not real PII.`

2. `README.md` -- Rewrite with:
   - Project description (AI Mortgage Quickstart -- what it is, who it serves).
   - Architecture diagram (ASCII from architecture Section 1.1).
   - Key technology decisions table.
   - Quickstart guide: `cp .env.example .env`, fill in API keys, `make setup && make dev`.
   - Development commands reference (from Makefile).
   - Code walkthrough section pointing to key files: supervisor agent, graph definition, checkpoint configuration.
   - Troubleshooting section for common issues.
   - AI compliance notice.

**Implementing agent:** `devops-engineer`

**Exit condition:**
```bash
test -f .env.example && grep -c "DATABASE_URL" .env.example && grep -c "make setup" README.md
# Expected: both return >= 1
```

---

### Phase 1 Context Package

**Work area: Infrastructure (P1-WU01, P1-WU02, P1-WU19)**

Files to read: `plans/architecture.md` Section 7, Section 9, Appendix A
Binding contracts: Makefile targets, compose service names, env var names
Key decisions: Native dev servers (not containerized API/UI in dev), pgvector image, uv for Python
Scope: Scaffolding and infra only -- no application code

**Work area: Database (P1-WU03, P1-WU04, P1-WU05, P1-WU06a, P1-WU06b)**

Files to read: `plans/architecture.md` Section 3, `packages/db/src/models.py`
Binding contracts: ORM models, repository method signatures (Section 2.3), migration naming
Key decisions: Append-only audit table, app_role grants, deterministic dev keys
Scope: Schema, migrations, seed data, repositories -- no API routes

**Work area: API Core (P1-WU07, P1-WU08, P1-WU09, P1-WU10, P1-WU11)**

Files to read: `packages/api/src/models/enums.py`, `packages/api/src/models/schemas.py`, `packages/api/src/dependencies.py`
Binding contracts: Pydantic schemas (Section 2.2), middleware contracts (Section 2.6), error format (Section 2.8)
Key decisions: Middleware ordering, RFC 7807 errors, contextvar for correlation ID
Scope: Settings, DI, middleware stack, error handling -- no business routes

**Work area: Application Routes (P1-WU12, P1-WU13, P1-WU14, P1-WU15)**

Files to read: All API Core files, repository interfaces, architecture Section 4.2
Binding contracts: API request/response shapes, MinIO bucket structure
Key decisions: SSN encrypted on create/never returned, PDF-only uploads, cursor pagination
Scope: CRUD routes and document handling -- no agent logic

**Work area: Agent Graph (P1-WU16, P1-WU17)**

Files to read: `packages/api/src/agents/state.py`, architecture Section 5.1
Binding contracts: LoanProcessingState TypedDict, node function signatures, graph structure
Key decisions: Full graph in Phase 1 with stubs, separate checkpoint pool, synchronous execution in request
Scope: Graph definition and submit endpoint -- stubs only, no real AI

---

## 4. Phase 2: Core AI Agents

**Goal:** Real document processing, credit analysis, risk assessment, confidence-based routing, human review workflow, and loan officer dashboard.

**User stories covered:** US-003, US-004, US-005, US-009, US-010, US-011, US-013, US-014, US-015, US-016, US-017, US-023, US-028, US-029, US-030, US-031, US-032, US-055, US-056, US-057

### Dependency Graph

```
P2-WU01 (PII redaction) ---> P2-WU02 (doc processing agent)
                                |
P2-WU03 (mock credit bureau) --+--> P2-WU04 (credit analysis agent)
                                |
                                +--> P2-WU05 (risk assessment agent)
                                       |
                                       +--> P2-WU06 (routing logic update)
                                               |
                                               +--> P2-WU07 (review queue + actions)
                                                       |
                                                       +--> P2-WU08 (config endpoints)

P2-WU09a (UI: build infra) ---> P2-WU09b (UI: auth + routing) ---> P2-WU10 (UI: app list + detail)
                                                                       |
                                                                       +--> P2-WU11 (UI: review panel)

P2-WU12 (Phase 2 integration test)
```

### P2-WU01: PII Redaction Service

**What to build:** Local OCR + NER pipeline for PII detection and redaction from document text before sending to external LLMs.

**Satisfies:** US-011

**Files to create:**
- `packages/api/src/services/pii_redaction.py`
- `packages/api/tests/unit/test_pii_redaction.py`

**Files to read for context:**
- `plans/architecture.md` Section 10.1 (OQ-9 resolution: Option A)

**Task description:**

Implement PII redaction using local OCR + NER approach (architecture decision):

1. `PiiRedactionService`:
   - `async redact_document(pdf_bytes: bytes) -> tuple[str, list[str]]` -- Returns (redacted_text, detected_pii_types).
   - Uses `pytesseract` for OCR to extract text from PDF pages.
   - Uses `presidio-analyzer` with `en_core_web_sm` spaCy model for NER to detect: SSN patterns, phone numbers, dates of birth, bank account numbers, addresses.
   - Replaces detected PII with `[REDACTED_SSN]`, `[REDACTED_PHONE]`, etc.
   - Returns the redacted text (not image) for downstream LLM processing.
   - If OCR confidence is below 0.5 for any page, raise `PiiRedactionError` -- processing should escalate to human review.

2. Add dependencies to `packages/api/pyproject.toml`: `pytesseract>=0.3`, `presidio-analyzer>=2.2`, `presidio-anonymizer>=2.2`, `spacy>=3.8`, `pdf2image>=1.17`.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_pii_redaction.py -v
# Test: SSN pattern "123-45-6789" is replaced with [REDACTED_SSN]
# Test: redacted text does not contain original SSN
# Test: non-PII text is preserved
# Test: low OCR confidence raises PiiRedactionError
```

---

### P2-WU02: Document Processing Agent (Replace Stub)

**What to build:** Real document classification and data extraction using GPT-4 Vision on redacted text.

**Satisfies:** US-009, US-010

**Files to create:**
- `packages/api/src/agents/nodes/document_processing.py` (replace stub)
- `packages/api/src/agents/llm.py`

**Files to read for context:**
- `packages/api/src/agents/state.py` (DocumentResult TypedDict)
- `packages/api/src/services/pii_redaction.py` (from P2-WU01)
- `plans/architecture.md` Section 5.6 (LLM Provider Strategy), Section 8.3 (Retry)

**Task description:**

1. `packages/api/src/agents/llm.py`:
   - `call_llm(model, messages, callbacks, correlation_id, agent_name)` -- Wrapper with tenacity retry (3 attempts, exponential backoff 1s/2s/4s), LangFuse callback if configured, token tracking.
   - `get_vision_model()` -- Returns `ChatOpenAI(model="gpt-4o")`.
   - `get_reasoning_model()` -- Returns `ChatAnthropic(model="claude-3-5-sonnet-latest")`.

2. `packages/api/src/agents/nodes/document_processing.py`:
   - For each document in the application:
     a. Retrieve PDF from MinIO.
     b. Run PII redaction to get clean text.
     c. Store redacted text version back to MinIO at `{app_id}/{doc_id}/redacted/`.
     d. Call vision model with redacted text for classification (W2, PAY_STUB, etc.).
     e. Call vision model with redacted text for structured data extraction.
     f. Validate LLM response structure:
        - Classification must return a valid DocumentType enum value.
        - Extraction must return expected fields for the document type.
        - If malformed, retry up to 3 times; on exhaustion, escalate to human review.
     g. Extract PDF metadata locally (creation date, producer, etc.) for fraud detection.
     h. Store results via `DocumentRepository.update_classification()` and `update_extraction()`.
     i. Create `AgentDecision` record.
   - Set `document_processing_complete = True` on success.
   - On any document failure: set `document_processing_complete = False` (will escalate).
   - Create audit events for each document processed.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_document_processing.py -v
# Test: document classified correctly given mock LLM response
# Test: extracted data matches expected structure
# Test: PII redaction called before LLM
# Test: failure sets document_processing_complete=False
```

---

### P2-WU03: Mock Credit Bureau Service

**What to build:** Mock implementation of the credit bureau service protocol.

**Satisfies:** US-013 (partial -- data source)

**Files to create:**
- `packages/api/src/services/protocols.py`
- `packages/api/src/services/credit_bureau_mock.py`

**Files to read for context:**
- Interface contract Section 2.7 (Service Protocol Contracts)
- `plans/architecture.md` Section 2.6 (Mocked service pattern)

**Task description:**

1. `packages/api/src/services/protocols.py` -- Define `CreditBureauService` and `PropertyDataService` protocols per Section 2.7.

2. `packages/api/src/services/credit_bureau_mock.py`:
   - `MockCreditBureauService.get_credit_report(ssn_hash)` -- Returns synthetic `CreditReport` data.
   - Use the last 2 digits of the SSN hash to deterministically generate diverse profiles:
     - Hash ending 00-30: excellent credit (750+, no derogatory marks)
     - Hash ending 31-60: good credit (680-749, 0-1 derogatory marks)
     - Hash ending 61-80: fair credit (620-679, 1-2 derogatory marks)
     - Hash ending 81-99: poor credit (500-619, 2+ derogatory marks)
   - Include realistic payment history, utilization ratio, and inquiry counts.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_credit_bureau_mock.py -v
# Test: returns CreditReport matching protocol
# Test: different SSN hashes produce different profiles
# Test: all credit score ranges represented
```

---

### P2-WU04: Credit Analysis Agent (Replace Stub)

**What to build:** LLM-based credit analysis using mocked credit report data.

**Satisfies:** US-013

**Files to modify:**
- `packages/api/src/agents/nodes/credit_analysis.py` (replace stub)

**Files to read for context:**
- `packages/api/src/services/credit_bureau_mock.py` (from P2-WU03)
- `packages/api/src/agents/llm.py` (from P2-WU02)
- `packages/api/src/agents/state.py` (CreditAnalysisResult)

**Task description:**

Replace stub with real implementation:
1. Retrieve credit report via `CreditBureauService.get_credit_report(ssn_hash)`.
2. Call Claude reasoning model with credit report data and prompt to:
   - Analyze credit score, payment history, derogatory marks, trends.
   - Produce confidence score (0.0-1.0).
   - Produce recommendation (APPROVE, DENY, REVIEW).
   - Produce plain-language summary.
   - Produce structured findings.
3. Store `AgentDecision` record with model_used, tokens, latency.
4. Create audit event `AGENT_COMPLETED`.
5. Return `CreditAnalysisResult` TypedDict.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_credit_analysis.py -v
# Test: agent returns CreditAnalysisResult shape
# Test: high credit score -> high confidence
# Test: agent decision stored in DB
```

---

### P2-WU05: Risk Assessment Agent (Replace Stub)

**What to build:** DTI calculation, LTV calculation, employment stability scoring, and cross-source income validation.

**Satisfies:** US-014, US-015, US-016, US-017

**Files to modify:**
- `packages/api/src/agents/nodes/risk_assessment.py` (replace stub)

**Files to read for context:**
- `packages/api/src/agents/state.py` (RiskAssessmentResult)
- `plans/requirements.md` US-014 through US-017

**Task description:**

Replace stub with real implementation:
1. Extract income data from document extraction results (W-2 wages, pay stub gross pay, tax return income).
2. Cross-validate income across sources (10% tolerance per US-016).
3. Calculate DTI: `(monthly_debts + estimated_monthly_payment) / gross_monthly_income`.
   - Use `Decimal` for all calculations.
   - Flag DTI > 43% (QM threshold).
4. Calculate LTV: `loan_amount / appraised_value`.
   - Flag LTV > 80% (PMI), > 95% (elevated risk).
5. Score employment stability (0.0-1.0):
   - Gaps > 3 months reduce score.
   - Recent job change (< 6 months) reduces score.
6. Call Claude reasoning model for overall risk narrative and confidence.
7. Store `AgentDecision`, create audit event.
8. Return `RiskAssessmentResult` TypedDict.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_risk_assessment.py -v
# Test: DTI calculated correctly (8000 income, 2400 debt = 0.30)
# Test: LTV calculated correctly (320000/400000 = 0.80)
# Test: income discrepancy > 10% flagged
# Test: DTI > 43% flagged
```

---

### P2-WU06: Confidence-Based Routing (Update Real Logic)

**What to build:** Update the routing decision node and aggregation node with real threshold-based logic and configuration endpoint.

**Satisfies:** US-005, US-023

**Files to modify:**
- `packages/api/src/agents/nodes/supervisor.py` (update aggregation + routing)

**Files to create:**
- `packages/api/src/routes/config.py`

**Files to read for context:**
- `plans/architecture.md` Section 5.4 (Confidence-Based Routing Logic)
- Interface contracts Section 2.2 (ThresholdUpdateRequest)

**Task description:**

1. Update `aggregation_node` in `supervisor.py`:
   - Collect results from credit_analysis and risk_assessment (Phase 2 active agents).
   - Filter out None results (compliance, fraud still stubs returning None).
   - Detect disagreements: different recommendations among active agents.
   - Generate consolidated narrative via Claude.
   - Set `has_disagreements`, `has_fraud_flags`.

2. Update `routing_decision_node`:
   - Load current thresholds from configuration table.
   - Apply routing rules per architecture Section 5.4.
   - Store thresholds snapshot in state for audit purposes.

3. `packages/api/src/routes/config.py`:
   - `GET /v1/config/thresholds` -- Returns current thresholds. Requires `reviewer` role.
   - `PUT /v1/config/thresholds` -- Updates thresholds. Validates ordering. Records audit event `THRESHOLD_CHANGED`. Requires `reviewer` role.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_routing_decision.py tests/integration/test_config.py -v
# Test: all agents high confidence -> AUTO_APPROVE
# Test: one agent medium confidence -> ESCALATE with loan_officer
# Test: one agent low confidence -> ESCALATE with senior_underwriter
# Test: fraud flag -> ESCALATE regardless
# Test: disagreement -> ESCALATE
# Test: threshold update records audit event
# Test: invalid threshold ordering rejected (400)
```

---

### P2-WU07: Review Queue and Human Review Actions

**What to build:** Review queue listing endpoint and approve/deny/request-docs actions.

**Satisfies:** US-028, US-030, US-031, US-032

**Files to create:**
- `packages/api/src/routes/review.py`

**Files to read for context:**
- `plans/architecture.md` Section 4.2 (Review Queue, Human Review shapes)
- Interface contracts Section 2.2 (ReviewRequest, ReviewResponse, ReviewQueueItem)

**Task description:**

1. `GET /v1/review-queue` -- List escalated applications.
   - Requires auth (any role).
   - Filters by role: loan_officer sees MEDIUM confidence items, senior_underwriter sees both MEDIUM and LOW.
   - Returns paginated `ReviewQueueItem` list.
   - Sorted by escalation timestamp (oldest first).

2. `POST /v1/applications/{id}/review` -- Submit review decision.
   - Requires auth. Role must match or exceed `required_reviewer_role` on the application.
   - Validates application is in `ESCALATED` status. Returns 409 otherwise.
   - Accept `ReviewRequest` body.
   - On APPROVE: status -> `HUMAN_APPROVED`. Audit event `HUMAN_APPROVED`.
   - On DENY: status -> `HUMAN_DENIED`. Audit event `HUMAN_DENIED`.
   - On REQUEST_MORE_DOCUMENTS: status -> `AWAITING_DOCUMENTS`. Audit event `DOCUMENTS_REQUESTED`.
   - Returns `ReviewResponse`.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_review.py -v
# Test: escalated app appears in review queue
# Test: loan_officer can approve MEDIUM confidence escalation
# Test: loan_officer cannot see LOW confidence escalation
# Test: approve updates status to HUMAN_APPROVED
# Test: deny updates status to HUMAN_DENIED
# Test: request-docs updates status to AWAITING_DOCUMENTS
# Test: review on non-ESCALATED app returns 409
```

---

### P2-WU08: Fraud Sensitivity Config Endpoint

**What to build:** API endpoint for managing fraud detection sensitivity.

**Satisfies:** US-027 (partial -- API only, agent in Phase 3a)

**Files to modify:**
- `packages/api/src/routes/config.py` (add fraud sensitivity endpoints)

**Task description:**

Add to existing config routes:
- `GET /v1/config/fraud-sensitivity` -- Returns current sensitivity. Requires `reviewer` role.
- `PUT /v1/config/fraud-sensitivity` -- Updates sensitivity (LOW/MEDIUM/HIGH). Records audit event `SENSITIVITY_CHANGED`. Requires `reviewer` role.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_config.py -v -k "fraud_sensitivity"
# Test: get returns current level
# Test: update changes level and records audit event
# Test: non-reviewer gets 403
```

---

### P2-WU09a: UI Build Infrastructure

**What to build:** React app build configuration, TypeScript setup, and entry point.

**Satisfies:** US-057 (partial -- build infra)

**Files to create:**
- `packages/ui/vite.config.ts`
- `packages/ui/tsconfig.json`
- `packages/ui/index.html`
- `packages/ui/src/main.tsx`

**Files to read for context:**
- `plans/architecture.md` Section 9 (UI file structure)
- Interface contracts Section 2.2 (response shapes)

**Task description:**

1. `vite.config.ts` -- Vite config with React plugin. Proxy `/v1/*` and `/health` to `http://localhost:8000`.
2. `tsconfig.json` -- TypeScript strict mode, path aliases for `@/`.
3. `index.html` -- HTML entry point.
4. `main.tsx` -- App entry point with TanStack Router and Query providers.

**Implementing agent:** `frontend-developer`

**Exit condition:**
```bash
cd packages/ui && npx tsc --noEmit
```

---

### P2-WU09b: UI Auth and Routing

**What to build:** React auth context, API client, root layout, and initial routes.

**Satisfies:** US-057 (partial -- shell)

**Depends on:** P2-WU09a

**Files to create:**
- `packages/ui/src/routes/__root.tsx`
- `packages/ui/src/routes/index.tsx`
- `packages/ui/src/routes/dashboard.tsx`
- `packages/ui/src/hooks/use-auth.ts`
- `packages/ui/src/lib/api-client.ts`

**Files to read for context:**
- `packages/ui/src/main.tsx` (from P2-WU09a)
- Interface contracts Section 2.2 (response shapes)

**Task description:**

1. `__root.tsx` -- Root layout with navigation header. Shows "Login" when unauthenticated, user name and role when authenticated.
2. `hooks/use-auth.ts` -- Auth context that stores API key in localStorage. `login(apiKey)` calls a test endpoint to verify key. `logout()` clears key. Exposes `user`, `isAuthenticated`, `role`.
3. `lib/api-client.ts` -- Typed fetch wrapper that adds `Authorization: Bearer` header from auth context. Handles 401/403/422 errors.
4. `dashboard.tsx` -- Placeholder dashboard route (protected, redirects to login if unauthenticated).
5. `index.tsx` -- Public landing page placeholder.

**Implementing agent:** `frontend-developer`

**Exit condition:**
```bash
cd packages/ui && npx tsc --noEmit && npx vitest run --reporter=verbose 2>&1 | tail -5
```

---

### P2-WU10: UI -- Application List and Detail Views

**What to build:** Application list page with status filters and application detail page showing agent analysis.

**Satisfies:** US-055, US-029, US-056

**Files to create:**
- `packages/ui/src/routes/applications/index.tsx`
- `packages/ui/src/routes/applications/$id.tsx`
- `packages/ui/src/hooks/use-applications.ts`
- `packages/ui/src/components/agent-analysis.tsx`

**Files to read for context:**
- `packages/ui/src/lib/api-client.ts` (from P2-WU09b)
- Interface contracts Section 2.2 (ApplicationDetail, AgentAnalysisSummary)

**Task description:**

1. `hooks/use-applications.ts` -- TanStack Query hooks:
   - `useApplications(status?, cursor?)` -- fetches paginated list.
   - `useApplication(id)` -- fetches detail.
   - `useApplicationStatus(id)` -- polls every 5 seconds while status is PROCESSING*.

2. `routes/applications/index.tsx` -- Application list with:
   - Status filter dropdown.
   - Table with columns: borrower name, status (colored badge), created date, updated date.
   - Click row navigates to detail.
   - Cursor-based pagination.

3. `routes/applications/$id.tsx` -- Application detail:
   - Application metadata section.
   - Documents list.
   - Agent analysis section (using agent-analysis component).
   - Status polling while processing.

4. `components/agent-analysis.tsx` -- Renders agent analysis results:
   - Each agent as a card with confidence score (color-coded), recommendation badge, summary text, expandable findings.
   - Consolidated narrative at top.
   - Disagreements highlighted in yellow.

**Implementing agent:** `frontend-developer`

**Exit condition:**
```bash
cd packages/ui && npx tsc --noEmit && npx vitest run --reporter=verbose 2>&1 | tail -5
```

---

### P2-WU11: UI -- Review Panel

**What to build:** Review queue page and review action panel on application detail.

**Satisfies:** US-028, US-030, US-031, US-032

**Files to create:**
- `packages/ui/src/routes/review-queue.tsx`
- `packages/ui/src/components/review-panel.tsx`

**Files to read for context:**
- `packages/ui/src/hooks/use-applications.ts` (from P2-WU10)
- Interface contracts Section 2.2 (ReviewQueueItem, ReviewRequest)

**Task description:**

1. `routes/review-queue.tsx` -- Review queue page:
   - List of escalated applications with escalation reason, required role, timestamp.
   - Color-coded by age (green < 1hr, yellow 1-4hr, red > 4hr).
   - Click navigates to application detail.

2. `components/review-panel.tsx` -- Review action panel (rendered on application detail when status is ESCALATED):
   - Three buttons: Approve, Deny, Request More Documents.
   - Rationale text area (required, min 10 chars).
   - Request docs: additional text area for document description.
   - Submit calls `POST /v1/applications/{id}/review`.
   - Shows success/error state after submission.

**Implementing agent:** `frontend-developer`

**Exit condition:**
```bash
cd packages/ui && npx tsc --noEmit && npx vitest run --reporter=verbose 2>&1 | tail -5
```

---

### P2-WU12: Phase 2 Integration Test

**What to build:** End-to-end test verifying the full Phase 2 workflow.

**Satisfies:** Phase 2 exit criteria, US-071 (partial -- verify submission within 30 seconds with real agents)

**Files to create:**
- `packages/api/tests/integration/test_phase2_e2e.py`

**Task description:**

Write integration tests covering:
1. Create application -> Upload document -> Submit -> Document processing agent classifies and extracts -> Credit analysis runs -> Risk assessment runs -> High confidence -> AUTO_APPROVED.
2. Same flow but with mock data producing medium confidence -> ESCALATED -> Appears in review queue -> Loan officer approves with rationale -> HUMAN_APPROVED.
3. Same flow but with mock data producing disagreement -> ESCALATED regardless of confidence.
4. Threshold update -> subsequent application uses new thresholds.

Use mocked LLM responses (don't call real APIs in tests).

**Implementing agent:** `test-engineer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_phase2_e2e.py -v
# Test: application submission completes within 30 seconds (with real agents, mocked LLMs) (US-071)
```

---

### Phase 2 Context Package

**Work area: AI Agent Nodes (P2-WU01 through P2-WU05)**

Files to read: `packages/api/src/agents/state.py`, `packages/api/src/agents/llm.py`, `packages/api/src/agents/nodes/supervisor.py`
Binding contracts: LoanProcessingState, agent node function signatures, DocumentResult/CreditAnalysisResult/RiskAssessmentResult TypedDicts
Key decisions: OCR+NER for PII redaction, mock credit bureau with hash-based profiles, Decimal for financial calcs
Scope: Agent implementations only -- no API routes, no UI

**Work area: Review and Config API (P2-WU06 through P2-WU08)**

Files to read: `packages/api/src/routes/applications.py`, `packages/api/src/models/schemas.py`, `packages/api/src/middleware/authorization.py`
Binding contracts: ReviewRequest/ReviewResponse shapes, threshold validation rules, audit event types
Key decisions: Role-filtered queue, 409 for wrong-state actions, mandatory rationale text
Scope: Review and config endpoints -- no agent logic, no UI

**Work area: UI (P2-WU09a, P2-WU09b, P2-WU10, P2-WU11)**

Files to read: `packages/ui/src/lib/api-client.ts`, API response shapes from Section 2.2
Binding contracts: API response JSON shapes (camelCase aliases), auth header format
Key decisions: TanStack Router file-based routing, status polling every 5s, localStorage for API key
Scope: React UI only -- no backend changes

---

## 5. Phase 3a: Full Agent Suite

**Goal:** Compliance checking with RAG, fraud detection, denial coaching, cyclic document resubmission.

**User stories covered:** US-007, US-012, US-018, US-019, US-020, US-024, US-025, US-026, US-027, US-033, US-034, US-035, US-036

### P3a-WU01: Knowledge Base and RAG Infrastructure

**What to build:** Compliance document ingestion, embedding generation, and RAG query service.

**Satisfies:** US-018 (partial -- infrastructure for compliance agent)

**Files to create:**
- `packages/api/src/services/embedding_service.py`
- `packages/api/src/agents/tools/knowledge_search.py`
- `packages/db/src/repositories/embedding.py`

**Task description:**

1. `embedding_service.py`:
   - `EmbeddingService.ingest_document(content, metadata, collection)` -- Chunk text (512 tokens, 50 token overlap), generate embeddings via OpenAI `text-embedding-3-small`, store in embeddings table.
   - `EmbeddingService.search(query, collection, top_k=5)` -- Embed query, cosine similarity search via pgvector, return ranked results with metadata.
   - Redis caching: cache query results for 24 hours at `rag:{query_hash}`.

2. `knowledge_search.py` -- LangGraph tool wrapper around `EmbeddingService.search()`.

3. `embedding.py` repository -- CRUD for embeddings table.

4. Create a seed script that ingests sample regulatory content (ECOA excerpts, Fair Housing Act excerpts, QM rule excerpts) into the `compliance` collection.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_embedding_service.py -v
# Test: ingest stores embeddings
# Test: search returns relevant results
# Test: Redis cache hit on second query
```

---

### P3a-WU02: Compliance Checking Agent (Replace Stub)

**What to build:** RAG-based compliance verification, adverse action notice generation, and audit completeness checking.

**Satisfies:** US-018, US-019, US-020

**Files to modify:**
- `packages/api/src/agents/nodes/compliance.py` (replace stub)

**Task description:**

1. Query compliance knowledge base for relevant regulations based on the application's agent findings.
2. Verify denial reasons are based on objective financial criteria (DTI, LTV, credit score).
3. Flag non-objective criteria in reasoning text.
4. Generate adverse action notice with regulatory citations when recommendation is DENY.
5. Verify audit trail completeness: check all expected audit events exist for the application.
6. Return `ComplianceCheckResult` TypedDict with confidence, violations, regulatory_references.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_compliance_agent.py -v
# Test: denial with DTI > 43% passes compliance (objective criteria)
# Test: denial with subjective reason fails compliance
# Test: adverse action notice includes regulatory citations
# Test: incomplete audit trail flagged
```

---

### P3a-WU03: Fraud Detection Agent (Replace Stub)

**What to build:** Income discrepancy detection, property flip patterns, identity consistency, PDF metadata anomalies.

**Satisfies:** US-024, US-025, US-026, US-012

**Files to modify:**
- `packages/api/src/agents/nodes/fraud_detection.py` (replace stub)

**Files to create:**
- `packages/api/src/services/property_data_mock.py`

**Task description:**

1. Income discrepancy detection: Compare income across W-2, pay stubs, tax returns. Flag > 15% variance.
2. Property flip detection: Query property data service for transaction history. Flag if purchased within 6 months with > 20% price increase.
3. Identity consistency: Compare name, SSN hash, address across documents. Flag mismatches.
4. PDF metadata: Check creation date (future?), producer (editing software?), missing fields.
5. Load sensitivity from configuration (`fraud_sensitivity` key).
6. Any flag with severity >= current sensitivity -> `has_fraud_flags = True` in state.
7. `property_data_mock.py`: Mock `PropertyDataService` that returns realistic fixture data. Use address hash for deterministic diversity.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_fraud_detection.py -v
# Test: 20% income discrepancy -> fraud flag
# Test: property flip pattern detected
# Test: HIGH sensitivity catches minor variance
# Test: LOW sensitivity ignores minor variance
```

---

### P3a-WU04: Denial Coaching Agent (Replace Stub)

**What to build:** Actionable improvement recommendations for denied or low-confidence applications.

**Satisfies:** US-033, US-034, US-035, US-036

**Files to modify:**
- `packages/api/src/agents/nodes/denial_coaching.py` (replace stub)

**Files to create:**
- `packages/api/src/agents/tools/mortgage_calculator.py`

**Task description:**

1. `mortgage_calculator.py` -- Pure computation tool (no LLM):
   - Monthly payment (PITI) using standard amortization formula with `Decimal`.
   - Total interest over loan life.
   - DTI preview.
   - Affordability estimate.
   - Comparison mode: accepts two sets of parameters, returns side-by-side results (maps to US-050).
   - All monetary values in integer cents.

2. Denial coaching node:
   - Analyze which factors contributed to denial/low confidence (DTI, LTV, credit score).
   - For DTI issues: calculate debt reduction needed to reach 43%.
   - For LTV issues: calculate down payment scenarios (10%, 15%, 20%).
   - For credit issues: provide improvement guidance.
   - Use mortgage calculator tool for what-if scenarios.
   - Call Claude for plain-language recommendations.
   - Return `CoachingOutput` TypedDict.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_denial_coaching.py tests/unit/test_mortgage_calculator.py -v
# Test: monthly payment calculation matches known value
# Test: DTI coaching includes debt reduction target
# Test: LTV coaching includes down payment scenarios
# Test: coaching output in plain language
```

---

### P3a-WU05: Cyclic Document Resubmission

**What to build:** Workflow restart from document processing when new documents are uploaded after a request-more-docs action.

**Satisfies:** US-007

**Files to modify:**
- `packages/api/src/routes/documents.py` (trigger resubmission)
- `packages/api/src/agents/nodes/supervisor.py` (archive previous analyses)

**Task description:**

1. When documents are uploaded to an application in `AWAITING_DOCUMENTS` status:
   - Archive current agent analysis results to `previous_analyses` in graph state.
   - Reset agent result fields to None.
   - Increment `resubmission_count`.
   - Resume the LangGraph thread from `human_review_wait` node.
   - The `route_after_review` function returns `"resubmit"`, sending flow back to `document_processing`.
   - Status transitions: AWAITING_DOCUMENTS -> DOCUMENT_PROCESSING.
   - Audit event: `workflow_restarted` with resubmission_count.

2. Update `supervisor_init_node` to handle resubmission context.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_resubmission.py -v
# Test: request-docs -> upload new doc -> workflow restarts from doc processing
# Test: previous analyses preserved in state
# Test: resubmission_count incremented
# Test: audit trail records resubmission
```

---

### Phase 3a Context Package

**Work area: RAG + Compliance (P3a-WU01, P3a-WU02)**

Files to read: `packages/api/src/agents/state.py`, `packages/db/src/repositories/embedding.py`, architecture Section 5.1
Binding contracts: ComplianceCheckResult TypedDict, EmbeddingService.search() signature, knowledge_search tool
Key decisions: pgvector for RAG, 512-token chunks, Redis cache for queries, ECOA/Fair Housing regulatory content
Scope: Compliance agent and RAG infrastructure -- no other agents

**Work area: Fraud + Coaching (P3a-WU03, P3a-WU04)**

Files to read: `packages/api/src/agents/state.py`, `packages/api/src/services/protocols.py`
Binding contracts: FraudDetectionResult, CoachingOutput TypedDicts, mortgage_calculator tool signature
Key decisions: 15% income variance threshold (vs 10% in risk assessment), sensitivity from config, Decimal for calculator
Scope: Fraud detection and denial coaching agents -- no compliance, no API routes

**Threshold rationale:** The 15% variance threshold for fraud detection is intentionally higher than the 10% threshold in risk assessment (P2-WU05). Risk assessment treats income discrepancy as a risk factor that reduces confidence; fraud detection treats larger discrepancies as potential fraud indicators. A 12% income variance would be flagged as a risk factor (reducing confidence, possibly triggering escalation) but would not trigger a fraud flag.

---

## 6. Phase 3b: Public Access and External Data

**Goal:** Public-facing intake agent, mortgage calculator, external data integration, rate limiting.

**User stories covered:** US-037, US-038, US-039, US-040, US-041, US-042, US-043, US-044, US-045, US-046, US-047, US-048, US-049, US-050, US-051, US-052, US-061, US-062, US-063, US-074

### P3b-WU01: FRED API Client

**What to build:** Client for retrieving mortgage rates and economic indicators from FRED.

**Satisfies:** US-061, US-062

**Files to create:**
- `packages/api/src/services/fred_client.py`
- `packages/api/src/agents/tools/fred_api.py`

**Task description:**

1. `fred_client.py`:
   - `FredClient.get_series(series_id: str) -> dict` -- Fetch latest observation from FRED API.
   - Series: MORTGAGE30US (30-year rate), DGS10 (10-year Treasury), CSUSHPISA (housing price index).
   - Redis caching: `ext:fred:{series_id}` with 1-hour TTL.
   - Fallback: last cached value with notice; if never cached, hardcoded 6.5% for mortgage rate.

3. Response validation:
   - Define Pydantic model `FredRateResponse` for validating FRED responses.
   - Reject responses > 1 KB.
   - Validate rate is between 0.1% and 20%.
   - On validation failure, log warning and use cached/fallback value.

2. `fred_api.py` -- LangGraph tool: `get_current_rates() -> dict`.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_fred_client.py -v
# Test: valid response parsed correctly
# Test: Redis cache used on second call
# Test: fallback to cached value when API unavailable
# Test: out-of-bounds rate rejected
```

---

### P3b-WU02: Mortgage Calculator API and Tool

**What to build:** Calculator API endpoint and UI component for all calculation types.

**Satisfies:** US-045, US-046, US-047, US-048, US-049, US-050, US-051, US-052

**Files to create:**
- `packages/api/src/routes/public.py`
- `packages/ui/src/components/mortgage-calculator.tsx`

**Files to read for context:**
- `packages/api/src/agents/tools/mortgage_calculator.py` (from P3a-WU04)
- Architecture Section 4.2 (Calculator request/response shapes)

**Task description:**

1. `routes/public.py`:
   - `POST /v1/public/calculator` -- Accepts calculation type and inputs, returns results via mortgage_calculator tool. No auth required.
   - `GET /v1/public/rates` -- Returns current rates from FRED client. No auth required.
   - All responses include legal disclaimer.
   - Amortization supports cursor-based pagination (12 months per page, max 120).

2. `mortgage-calculator.tsx`:
   - Form with inputs: home price, down payment (slider + input), loan term, interest rate (auto-populated from rates API).
   - Results display: monthly PITI breakdown, total interest, DTI preview (if income provided).
   - Comparison mode: side-by-side scenarios A and B.
   - All outputs include disclaimer.

**Implementing agent:** `backend-developer` (API), `frontend-developer` (UI)

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_calculator_api.py -v
cd packages/ui && npx tsc --noEmit
```

---

### P3b-WU03: Intake Graph Implementation

**What to build:** The complete Intake Graph for public-facing conversational agent.

**Satisfies:** US-037, US-038, US-041

**Files to create:**
- `packages/api/src/agents/graphs/intake.py`
- `packages/api/src/agents/nodes/intake_nodes.py`
- `packages/api/src/agents/tools/property_lookup.py`

**Files to read for context:**
- Architecture Section 5.2 (Intake Graph)
- Interface contract Section 2.4 (IntakeState)

**Task description:**

1. `intake_nodes.py`:
   - `analyze_input_node` -- Sentiment analysis + prompt injection check. Returns sentiment and routing decision.
   - `retrieve_knowledge_node` -- RAG search via knowledge_search tool. Returns citations.
   - `invoke_tools_node` -- Calls calculator, FRED, or property tools as needed.
   - `generate_response_node` -- LLM generates final response with citations and disclaimer.

2. `intake.py` -- Graph definition per architecture Section 5.2. Routing: knowledge, tool, direct, or blocked.

3. `property_lookup.py` -- LangGraph tool wrapping PropertyDataService.

4. Add `POST /v1/public/chat` to `routes/public.py` -- Accepts message, creates/reuses session, invokes intake graph, returns response with citations and sentiment.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_intake_graph.py -v
# Test: mortgage question -> knowledge retrieval path
# Test: calculation question -> tool invocation path
# Test: prompt injection -> blocked path
# Test: response includes citations
```

---

### P3b-WU04: Rate Limiting and Session Management

**What to build:** Redis-based session and rate limiting middleware for the public tier.

**Satisfies:** US-042, US-043, US-072 (partial -- graceful degradation on Redis unavailability)

**Files to create:**
- `packages/api/src/middleware/rate_limit.py`
- `packages/api/src/services/redis_client.py`

**Task description:**

1. `redis_client.py` -- Redis connection wrapper. `get_redis()` dependency. Graceful fallback to in-memory counters if Redis unavailable.

2. `rate_limit.py`:
   - Session management: create UUID session on first public request, store in Redis with 24hr TTL, set signed cookie.
   - Session rate limits: 20 chat/hr, 5 property/session, 100 calc/hr.
   - IP rate limits: 100 chat/day, 20 property/day, 500 calc/day.
   - Concurrency limit: max 1 active inference per session via Redis `SET NX` with 60s TTL.
   - Returns 429 with `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers.

3. Per-session token budget:
   - Track cumulative (prompt_tokens + completion_tokens) per session in Redis at `ratelimit:tokens:session:{session_id}` with 24h TTL.
   - Budget: 50,000 tokens per 24-hour session.
   - Check before LLM invocation; reject with 429 if budget exhausted: "Session token budget exhausted."
   - After each LLM response, increment counter by actual token usage.

Session IDs are distinct from correlation IDs. Session state (rate limits, token budget) is keyed by session ID, never by correlation ID.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_rate_limiting.py -v
# Test: 21st chat in 1 hour returns 429
# Test: 6th property lookup in session returns 429
# Test: concurrent request returns 429
# Test: rate limit headers present in responses
# Test: cumulative tokens exceeding 50,000 returns 429
# Test: Redis unavailability falls back to in-memory counters
```

---

### P3b-WU05: Prompt Injection Defenses

**What to build:** Multi-layer prompt injection detection and blocking.

**Satisfies:** US-044

**Files to create:**
- `packages/api/src/security/prompt_injection.py`
- `packages/api/src/security/output_filter.py`

**Task description:**

1. `prompt_injection.py`:
   - `sanitize_input(text: str) -> str` -- Remove system-prompt markers, instruction delimiters, XML tags.
   - `detect_injection(text: str) -> tuple[bool, str | None]` -- Pattern match for common injection patterns. Returns (is_injection, pattern_matched).
   - `validate_length(text: str, max_length: int = 2000) -> bool`.

2. `output_filter.py`:
   - `filter_output(text: str) -> str` -- Remove reflected injection content from LLM responses.

3. Semantic injection detection:
   - Pattern match for common injection attempts: "ignore previous instructions", "you are now", "system prompt", "disregard safety".
   - Case-insensitive regex matching on sanitized input.
   - Flagged inputs logged with hash (not verbatim); return generic 400 rejection.

4. Integrate into `analyze_input_node` in the Intake Graph. Flagged inputs return generic error and log security event with hashed input.

**Implementing agent:** `backend-developer`

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_prompt_injection.py -v
# Test: "ignore previous instructions" input detected and rejected
# Test: "you are now" detected
# Test: "system prompt" detected
# Test: "disregard safety" detected
# Test: normal mortgage question passes
# Test: input > 2000 chars rejected
# Test: output filtering removes reflected content
```

---

### P3b-WU06: Streaming Chat (SSE)

**What to build:** Server-Sent Events endpoint for streaming chat responses.

**Satisfies:** US-074

**Files to modify:**
- `packages/api/src/routes/public.py` (add stream endpoint)

**Files to create:**
- `packages/ui/src/hooks/use-chat.ts`
- `packages/ui/src/components/chat-interface.tsx`

**Task description:**

1. `POST /v1/public/chat/stream` -- SSE endpoint. Same input as `/v1/public/chat`. Streams token events, then citations event, metadata event, done event per architecture Section 4.2.

2. `use-chat.ts` -- Hook managing chat state, message history, SSE connection via Fetch API ReadableStream (not EventSource, since POST is required).

3. `chat-interface.tsx` -- Chat UI: message list, input field, streaming text rendering, citation display, disclaimer.

**Implementing agent:** `backend-developer` (API), `frontend-developer` (UI)

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_chat_streaming.py -v
cd packages/ui && npx tsc --noEmit
```

---

### P3b-WU07: Public Landing Page

**What to build:** Public landing page with calculator and chat.

**Satisfies:** US-037 (partial -- UI)

**Files to modify:**
- `packages/ui/src/routes/index.tsx` (replace placeholder)

**Task description:**

Update the public landing page:
- Hero section with project description.
- Mortgage calculator widget (from P3b-WU02).
- Chat interface widget (from P3b-WU06).
- Current rates display (fetched from `/v1/public/rates`).
- No login required.
- Disclaimer banner.

**Implementing agent:** `frontend-developer`

**Exit condition:**
```bash
cd packages/ui && npx tsc --noEmit
```

---

## 7. Phase 4: Polish and Operability

**Goal:** Observability, audit export, admin UI, deployment manifests, CI, documentation.

**User stories covered:** US-022, US-066, US-070, US-075, US-076, US-077, US-078, US-079, US-080, US-081

Phase 4 WUs are listed at a higher level since they are further out and more likely to change during implementation. Phase 4 WUs will be refined with stronger exit conditions before implementation begins. UI WUs should include at least one vitest component test in addition to type checking.

### P4-WU01: LangFuse Integration

**Satisfies:** US-066. Add LangFuse callbacks to all LLM calls across all agents. Verify traces appear in dashboard.

**Exit condition:** `cd packages/api && pytest tests/integration/test_langfuse.py -v`

### P4-WU02: Audit Trail Export

**Satisfies:** US-022. `GET /v1/applications/{id}/audit/export` returns JSON document with complete audit trail. Reviewer role required.

Access control: Reviewers can only export audit trails for applications they have acted on (query audit_events for matching actor_id). Returns 403 if reviewer has not reviewed the application. Records AUDIT_EXPORTED event.

**Exit condition:** `cd packages/api && pytest tests/integration/test_audit_export.py -v # Test: reviewer cannot export audit for application they haven't reviewed (403)`

### P4-WU03: Admin Configuration UI

**Satisfies:** US-076. Web interface for threshold and fraud sensitivity management. Reviewer role required.

**Exit condition:** `cd packages/ui && npx tsc --noEmit`

### P4-WU04: Portfolio Analytics Dashboard

**Satisfies:** US-077. Aggregate metrics: approval rates, processing times, escalation frequency. Reviewer role.

**Exit condition:** `cd packages/ui && npx tsc --noEmit`

### P4-WU05: Cross-Session Chat Context

**Satisfies:** US-075. Persist chat history for authenticated users. Load prior context on new session.

**Exit condition:** `cd packages/api && pytest tests/integration/test_cross_session_chat.py -v`

### P4-WU06: Seed Data Expansion

**Satisfies:** US-070. 10+ diverse applications in various states with audit trails.

**Exit condition:** `cd packages/db && alembic upgrade head && python -c "..." # verify 10+ apps`

### P4-WU07: Container Deployment Manifests

**Satisfies:** US-078. Containerfiles for API and UI. Helm chart for OpenShift/K8s.

**Exit condition:** `docker build -f packages/api/Containerfile -t mortgage-api . && docker build -f packages/ui/Containerfile -t mortgage-ui .`

### P4-WU08: CI Pipeline

**Satisfies:** US-079. GitHub Actions workflow: lint, type-check, test, vulnerability scan.

**Exit condition:** `act -j test` (local CI runner) or verify `.github/workflows/ci.yml` syntax.

### P4-WU09: API Key Lifecycle Management

**Satisfies:** US-080. Generate, revoke, expire API keys. Reviewer role. Audit trail.

**Exit condition:** `cd packages/api && pytest tests/integration/test_key_management.py -v`

### P4-WU10: Documentation Polish

**Satisfies:** US-081. README code walkthrough, architecture overview, troubleshooting, inline comments.

**Exit condition:** `grep -c "walkthrough\|pattern\|checkpoint" README.md # >= 3 occurrences`

---

## 8. Cross-Task Dependencies

| Produces | Consumed By | Contract |
|----------|-------------|----------|
| P1-WU01 (scaffolding) | All subsequent WUs | Package structure, pyproject.toml, package.json |
| P1-WU03 (ORM models) | P1-WU04, P1-WU06a, P1-WU06b, all backend WUs | SQLAlchemy model classes |
| P1-WU04 (migrations) | P1-WU05, all WUs needing DB | Database tables exist |
| P1-WU06a (core repos) | P1-WU06b, P1-WU12, P1-WU13, P1-WU15, P1-WU16, P1-WU17, all P2+ API routes | Repository method signatures (Section 2.3) |
| P1-WU06b (supporting repos) | P1-WU10, P1-WU12, P1-WU17, all P2+ API routes | Repository method signatures (Section 2.3) |
| P1-WU07 (settings) | All API WUs | Settings class, dependency injection |
| P1-WU08 (errors) | All route WUs | AppError hierarchy, RFC 7807 format |
| P1-WU09 (middleware) | All route WUs | correlation_id_var, structured log format |
| P1-WU10 (auth) | All protected route WUs | AuthenticatedUser on request.state |
| P1-WU11 (RBAC) | All protected route WUs | require_role() dependency |
| P1-WU16 (graph + state) | P1-WU17, all agent WUs in P2/P3a | LoanProcessingState TypedDict, graph structure |
| P2-WU01 (PII redaction) | P2-WU02 (doc processing) | PiiRedactionService.redact_document() |
| P2-WU02 (LLM wrapper) | All agent WUs | call_llm(), get_vision_model(), get_reasoning_model() |
| P2-WU03 (mock credit) | P2-WU04 (credit agent) | CreditBureauService protocol |
| P3a-WU01 (RAG) | P3a-WU02 (compliance), P3b-WU03 (intake) | EmbeddingService, knowledge_search tool |
| P3a-WU04 (calculator tool) | P3b-WU02 (calculator API), P3b-WU03 (intake) | mortgage_calculator tool function |
| P2-WU09a (UI build infra) | P2-WU09b | Vite config, tsconfig, entry point |
| P2-WU09b (UI auth + routing) | P2-WU10, P2-WU11, all P3b UI WUs | Router, auth context, api-client |

---

## 9. Risks and Open Questions

| Risk | Mitigation |
|------|------------|
| PII redaction OCR accuracy on diverse document formats | Start with standard W-2/pay stub formats; escalate to human review on low OCR confidence; validate with test documents |
| LangGraph checkpoint schema changes between versions | Pin LangGraph version in pyproject.toml; test checkpoint resume in CI |
| Mock credit data not diverse enough to exercise all code paths | Hash-based deterministic generation covers 4 credit tiers; add edge cases to seed data |
| Phase 1 stub approach may not catch integration issues | Phase 2 integration tests verify real agent -> graph -> DB flow end-to-end |
| Tesseract + Presidio add significant dependencies to the API container | Consider separate redaction service in Phase 4 if container size is a concern |
| LLM prompt engineering for document extraction requires iteration | Start with simple prompts, tune based on test document results; encapsulate prompts in separate files for easy iteration |
| Redis unavailability degrades rate limiting to per-process counters | Acceptable for MVP; document limitation; add distributed fallback in Phase 4 if needed |

**Open questions requiring resolution before implementation:**

1. **OQ-1 (Knowledge base content):** Which specific regulatory documents to include for compliance RAG? Recommended: ECOA excerpts, Fair Housing Act excerpts, TILA, RESPA. Need stakeholder confirmation on document set and currency requirements.

2. **OQ-9 (PII redaction -- confirmed direction):** Option A (OCR + NER) is the recommended approach. Pending final stakeholder confirmation before P2-WU01 begins.

3. **Deterministic dev API keys (Resolved):** Use fixed well-known keys in `.env.example` flagged as `is_default=true`. The startup warning (P1-WU18) ensures these are not used in production unknowingly.

---

## Checklist

- [x] All cross-task interface contracts defined with concrete types (Section 2)
- [x] Data flow traced end-to-end for primary operations (architecture Section 3.4 referenced)
- [x] Error handling strategy specified (RFC 7807, AppError hierarchy in P1-WU08)
- [x] File/module structure mapped to existing project layout (architecture Section 9)
- [x] Key technical decisions documented with rationale (architecture Section 1.2)
- [x] Cross-task dependency map complete (Section 8)
- [x] Context Package defined for each phase (per-phase sections)
- [x] No TBDs in interface contracts -- OQ-1 and OQ-9 flagged as open questions
- [x] Every WU has machine-verifiable exit condition
- [x] Every WU maps to specific user stories
- [x] WU scoping follows chunking heuristics (3-5 files, single concern)

<!-- This project was developed with assistance from AI tools. -->

# Work Breakdown: AI Mortgage Quickstart

**Version:** 1.0
**Date:** 2026-02-12
**Status:** Approved

---

## Table of Contents

1. [Overview](#1-overview)
2. [Epic Structure](#2-epic-structure)
3. [Sprint Planning](#3-sprint-planning)
4. [Critical Path Analysis](#4-critical-path-analysis)
5. [Risk Register](#5-risk-register)
6. [Story Details by Phase](#6-story-details-by-phase)

---

## 1. Overview

This work breakdown organizes the 55 work units from the technical design into trackable stories grouped into epics and sprints. It provides the sequencing, dependency mapping, and effort estimation needed to execute the project.

### Summary Metrics

| Metric | Value |
|--------|-------|
| Total Epics | 11 |
| Total Stories | 55 |
| Total Estimated Effort | ~72 hours |
| Critical Path Duration | ~36 hours |
| Phases | 4 |
| Sprints | 9 |

### Effort Sizing Legend

- **S (Small):** ~30 minutes - Simple, well-defined task
- **M (Medium):** ~1 hour - Standard complexity task
- **L (Large):** ~2 hours - Complex task or significant integration

---

## 2. Epic Structure

### E-P1-01: Foundation Infrastructure
**Goal:** Establish the project skeleton, infrastructure services, and build system.
**PRD Reference:** Developer setup experience (P0)
**Priority:** P0
**Milestone:** Phase 1
**Estimated Size:** XL

**Stories:** S-P1-01, S-P1-02
**Acceptance Criteria (Epic-Level):**
- [ ] All three packages (api, db, ui) have working build configuration
- [ ] Docker Compose brings up all infrastructure services with health checks passing
- [ ] Makefile provides standard development commands
- [ ] Developer can run `make setup` and have a working environment

**Dependencies:** None (starting epic)
**Blocks:** All other epics

---

### E-P1-02: Database Foundation
**Goal:** Implement the complete database layer with schema, migrations, and data access patterns.
**PRD Reference:** Database schema and migrations (P0)
**Priority:** P0
**Milestone:** Phase 1
**Estimated Size:** L

**Stories:** S-P1-03, S-P1-04, S-P1-05, S-P1-06, S-P1-07
**Acceptance Criteria (Epic-Level):**
- [ ] All database tables defined in architecture Section 3.1 exist
- [ ] Migrations are idempotent and support rollback
- [ ] Seed data includes users with all three roles and valid API keys
- [ ] All repository interfaces from Section 2.3 are implemented
- [ ] Audit trail is append-only at the database permission level

**Dependencies:** E-P1-01
**Blocks:** E-P1-03, E-P1-04

---

### E-P1-03: API Core and Middleware
**Goal:** Build the FastAPI application core with authentication, authorization, logging, and error handling.
**PRD Reference:** API key authentication (P0), Role-based access control (P0), Health checks (P0)
**Priority:** P0
**Milestone:** Phase 1
**Estimated Size:** L

**Stories:** S-P1-08, S-P1-09, S-P1-10, S-P1-11, S-P1-12
**Acceptance Criteria (Epic-Level):**
- [ ] Health and readiness endpoints functional
- [ ] Correlation ID propagates through all requests
- [ ] Structured logging captures request lifecycle
- [ ] Authentication middleware validates API keys and resolves roles
- [ ] Authorization middleware enforces role hierarchy
- [ ] RFC 7807 error responses for all error conditions

**Dependencies:** E-P1-02
**Blocks:** E-P1-04, E-P1-05

---

### E-P1-04: Application and Document Management
**Goal:** Implement loan application CRUD, document upload to object storage, and audit trail access.
**PRD Reference:** Loan application management (P0), Immutable audit trail (P0)
**Priority:** P0
**Milestone:** Phase 1
**Estimated Size:** M

**Stories:** S-P1-13, S-P1-14, S-P1-15, S-P1-16
**Acceptance Criteria (Epic-Level):**
- [ ] Loan officers can create applications via API
- [ ] Documents can be uploaded and stored in MinIO
- [ ] Audit trail is queryable per application
- [ ] All CRUD operations record audit events

**Dependencies:** E-P1-03
**Blocks:** E-P1-05

---

### E-P1-05: Agent Orchestration Framework
**Goal:** Implement the supervisor-worker agent graph with stub nodes and checkpoint persistence.
**PRD Reference:** Supervisor-worker agent orchestration (P0)
**Priority:** P0
**Milestone:** Phase 1
**Estimated Size:** L

**Stories:** S-P1-17, S-P1-18, S-P1-19, S-P1-20
**Acceptance Criteria (Epic-Level):**
- [ ] LangGraph graph definition includes all nodes and edges per architecture
- [ ] Stub nodes simulate the full workflow without real AI
- [ ] Workflow state persists to PostgresSaver and survives service restart
- [ ] Application submission triggers workflow execution
- [ ] Integration test verifies end-to-end flow from submission to audit trail
- [ ] README enables developer setup in under 30 minutes

**Dependencies:** E-P1-04
**Blocks:** E-P2-01

---

### E-P2-01: Document Intelligence
**Goal:** Replace document processing stub with real vision-based extraction and PII redaction.
**PRD Reference:** Document processing agent (P0)
**Priority:** P0
**Milestone:** Phase 2
**Estimated Size:** M

**Stories:** S-P2-01, S-P2-02
**Acceptance Criteria (Epic-Level):**
- [ ] Document images are redacted of PII before external LLM calls
- [ ] Vision model classifies documents into types with confidence scores
- [ ] Structured data extraction produces per-field confidence
- [ ] Extraction results persist to document records

**Dependencies:** E-P1-05
**Blocks:** E-P2-02

---

### E-P2-02: Financial Analysis Agents
**Goal:** Implement credit analysis and risk assessment agents with real mocked data sources.
**PRD Reference:** Credit analysis agent (P0), Risk assessment agent (P0)
**Priority:** P0
**Milestone:** Phase 2
**Estimated Size:** M

**Stories:** S-P2-03, S-P2-04, S-P2-05
**Acceptance Criteria (Epic-Level):**
- [ ] Credit analysis agent evaluates mocked credit reports
- [ ] Risk assessment agent calculates DTI and LTV ratios
- [ ] Both agents produce confidence-scored recommendations
- [ ] Agent decisions persist with reasoning and model metadata

**Dependencies:** E-P2-01
**Blocks:** E-P2-03

---

### E-P2-03: Decision Routing and Human Review
**Goal:** Implement confidence-based routing and human-in-the-loop workflow.
**PRD Reference:** Confidence-based routing (P0), Human-in-the-loop review workflow (P0)
**Priority:** P0
**Milestone:** Phase 2
**Estimated Size:** M

**Stories:** S-P2-06, S-P2-07, S-P2-08
**Acceptance Criteria (Epic-Level):**
- [ ] Routing logic evaluates aggregated confidence against thresholds
- [ ] Applications escalate to review queue when confidence is insufficient
- [ ] Review queue is role-filtered based on escalation severity
- [ ] Reviewers can approve, deny, or request more documents
- [ ] All review actions record to audit trail with rationale

**Dependencies:** E-P2-02
**Blocks:** E-P2-04

---

### E-P2-04: Loan Officer Dashboard
**Goal:** Build the React UI for application management and review workflow.
**PRD Reference:** Loan officer dashboard (P0)
**Priority:** P0
**Milestone:** Phase 2
**Estimated Size:** L

**Stories:** S-P2-09, S-P2-10, S-P2-11, S-P2-12, S-P2-13
**Acceptance Criteria (Epic-Level):**
- [ ] UI displays application list with status and filters
- [ ] Application detail view shows agent analysis with confidence scores
- [ ] Review panel allows approval/denial with rationale
- [ ] UI authenticates via API key in Authorization header
- [ ] Integration test verifies full user flow

**Dependencies:** E-P1-01
**Blocks:** None (completes Phase 2)

---

### E-P3a-01: Compliance and Risk Intelligence
**Goal:** Add compliance checking, fraud detection, and denial coaching agents.
**PRD Reference:** Compliance checking agent (P0), Fraud detection agent (P0), Denial coaching agent (P0)
**Priority:** P0
**Milestone:** Phase 3a
**Estimated Size:** L

**Stories:** S-P3a-01, S-P3a-02, S-P3a-03, S-P3a-04, S-P3a-05
**Acceptance Criteria (Epic-Level):**
- [ ] RAG knowledge base supports compliance document retrieval
- [ ] Compliance agent verifies decisions against regulations
- [ ] Fraud detection identifies suspicious patterns
- [ ] Denial coaching generates actionable improvement recommendations
- [ ] Workflow supports document resubmission loop

**Dependencies:** E-P1-05
**Blocks:** E-P3b-01

---

### E-P3b-01: Public Access Tier
**Goal:** Implement public-facing intake agent, calculator, and rate limiting.
**PRD Reference:** Intake agent (public chat) (P0), Mortgage calculator (P0), Public access tier with rate limiting (P0)
**Priority:** P0
**Milestone:** Phase 3b
**Estimated Size:** XL

**Stories:** S-P3b-01, S-P3b-02, S-P3b-03, S-P3b-04, S-P3b-05, S-P3b-06, S-P3b-07
**Acceptance Criteria (Epic-Level):**
- [ ] Unauthenticated users can access chat, calculator, and rates
- [ ] Intake agent operates as independent LangGraph graph
- [ ] Rate limiting enforces session and IP caps
- [ ] Prompt injection defenses prevent abuse
- [ ] Streaming chat delivers incremental responses
- [ ] UI landing page provides public access

**Dependencies:** E-P2-03
**Blocks:** E-P4-01

---

### E-P4-01: Operability and Polish
**Goal:** Add observability, admin tools, analytics, deployment manifests, and CI pipeline.
**PRD Reference:** LLM observability dashboard (P1), Admin configuration interface (P2), Portfolio analytics dashboard (P2), Container deployment manifests (P2), CI pipeline (P2)
**Priority:** P1/P2
**Milestone:** Phase 4
**Estimated Size:** XL

**Stories:** S-P4-01 through S-P4-10
**Acceptance Criteria (Epic-Level):**
- [ ] LangFuse traces all LLM interactions with cost visibility
- [ ] Audit trails are exportable as documents
- [ ] Reviewers can adjust thresholds via admin UI
- [ ] Portfolio analytics provide aggregate metrics
- [ ] Chat context persists across sessions for authenticated users
- [ ] Seed data includes diverse test cases
- [ ] Helm charts enable container platform deployment
- [ ] CI pipeline runs on every push
- [ ] API key lifecycle management supports rotation
- [ ] Documentation is comprehensive and developer-friendly

**Dependencies:** E-P3b-01
**Blocks:** None (final epic)

---

## 3. Sprint Planning

### Sprint 1: Foundation Infrastructure (Phase 1, Week 1)

**Sprint Goal:** Establish the project structure, infrastructure services, and database foundation.

**Stories Included:**
- S-P1-01: Project Scaffolding
- S-P1-02: Docker Compose and Build System
- S-P1-03: ORM Models
- S-P1-04: Database Migrations
- S-P1-05: Seed Data
- S-P1-06: Core Repositories
- S-P1-07: Supporting Repositories

**Parallelization Map:**
- Sequential: S-P1-01 → S-P1-02 → S-P1-03 → S-P1-04 → S-P1-05
- Parallel after S-P1-04: S-P1-06, S-P1-07 (both depend on migrations but independent of each other)

**Sprint Exit Criteria:**
- [ ] `make setup` completes without errors
- [ ] All infrastructure services show healthy status
- [ ] Migrations run and rollback successfully
- [ ] Seed data populates all required tables
- [ ] Repository unit tests pass

**Estimated Effort:** 8 hours (S-P1-01: 0.5h, S-P1-02: 1h, S-P1-03: 1.5h, S-P1-04: 1h, S-P1-05: 1h, S-P1-06: 1.5h, S-P1-07: 1.5h)

---

### Sprint 2: API Core and Middleware (Phase 1, Week 1-2)

**Sprint Goal:** Build the FastAPI application with authentication, authorization, and error handling.

**Stories Included:**
- S-P1-08: Settings and Dependency Injection
- S-P1-09: Health Checks and Error Handling
- S-P1-10: Correlation ID and Logging
- S-P1-11: Authentication Middleware
- S-P1-12: RBAC Middleware

**Parallelization Map:**
- S-P1-07 (Supporting Repos) can run in parallel with S-P1-08 and S-P1-09 (no dependency between them)
- Sequential core: S-P1-08 → S-P1-09 → S-P1-10
- After S-P1-10, parallel: S-P1-11, S-P1-12 (but S-P1-12 may want to test with S-P1-11 first)
- Recommended sequence: S-P1-08 → S-P1-09 → S-P1-10 → S-P1-11 → S-P1-12

**Sprint Exit Criteria:**
- [ ] `/health` and `/ready` endpoints return 200
- [ ] All requests log with correlation ID
- [ ] Invalid API keys return 401
- [ ] Insufficient permissions return 403
- [ ] Error responses follow RFC 7807 format

**Estimated Effort:** 5.5 hours (S-P1-08: 1h, S-P1-09: 1.5h, S-P1-10: 1h, S-P1-11: 1h, S-P1-12: 1h)

---

### Sprint 3: Application Management and Agent Framework (Phase 1, Week 2)

**Sprint Goal:** Implement application CRUD, document upload, and the LangGraph orchestration framework with stubs.

**Note:** This is the most loaded sprint (8 stories, 10h). Consider splitting into Sprint 3a (Application Routes: S-P1-13 through S-P1-16, ~4.5h with parallelism ~2.5h) and Sprint 3b (Agent Framework: S-P1-17 through S-P1-20, ~5.5h sequential) for smaller review surfaces and earlier feedback on application routes.

**Stories Included:**
- S-P1-13: Application CRUD Routes
- S-P1-14: Document Upload Route
- S-P1-15: MinIO Client Service
- S-P1-16: Audit Trail Route
- S-P1-17: LangGraph Graph Definition
- S-P1-18: Application Submit Endpoint
- S-P1-19: Phase 1 Integration Test
- S-P1-20: Developer Documentation

**Parallelization Map:**
- Start: S-P1-13, S-P1-15 (parallel)
- After S-P1-13 and S-P1-15: S-P1-14, S-P1-16 (parallel)
- After S-P1-14 and S-P1-16: S-P1-17 → S-P1-18 → S-P1-19 → S-P1-20

**Sprint Exit Criteria:**
- [ ] Applications can be created, listed, and retrieved
- [ ] Documents upload to MinIO and link to applications
- [ ] Audit trail is queryable
- [ ] Workflow executes through all stub nodes
- [ ] Workflow state survives service restart
- [ ] Integration test passes end-to-end
- [ ] Developer can follow README from clone to working demo

**Estimated Effort:** 10 hours (S-P1-13: 1.5h, S-P1-14: 1h, S-P1-15: 1h, S-P1-16: 1h, S-P1-17: 2h, S-P1-18: 1h, S-P1-19: 1.5h, S-P1-20: 1h)

**Phase 1 Complete After Sprint 3**

---

### Sprint 4: Document Intelligence (Phase 2, Week 3)

**Sprint Goal:** Replace document processing stub with real vision-based extraction and PII redaction.

**Stories Included:**
- S-P2-01: PII Redaction Service
- S-P2-02: Document Processing Agent

**Parallelization Map:**
- Sequential: S-P2-01 → S-P2-02 (redaction must exist before document processing uses it)

**Sprint Exit Criteria:**
- [ ] Document images are redacted before external API calls
- [ ] Vision model classifies documents with confidence scores
- [ ] Extracted data persists with per-field confidence
- [ ] Unit tests cover redaction and extraction

**Estimated Effort:** 3.5 hours (S-P2-01: 1.5h, S-P2-02: 2h)

---

### Sprint 5: Financial Analysis and Routing (Phase 2, Week 3-4)

**Sprint Goal:** Implement credit, risk, and routing agents to enable intelligent decision-making.

**Stories Included:**
- S-P2-03: Mock Credit Bureau Service
- S-P2-04: Credit Analysis Agent
- S-P2-05: Risk Assessment Agent
- S-P2-06: Confidence-Based Routing
- S-P2-07: Review Queue and Human Review Actions
- S-P2-08: Fraud Sensitivity Config

**Parallelization Map:**
- Start: S-P2-03
- After S-P2-03, parallel: S-P2-04, S-P2-05
- After S-P2-04 and S-P2-05: S-P2-06
- After S-P2-06, parallel: S-P2-07, S-P2-08

**Sprint Exit Criteria:**
- [ ] Credit and risk agents produce confidence-scored assessments
- [ ] Routing logic escalates low-confidence applications
- [ ] Review queue is role-filtered
- [ ] Reviewers can approve/deny with rationale
- [ ] Fraud sensitivity is configurable

**Estimated Effort:** 7.5 hours (S-P2-03: 1h, S-P2-04: 1.5h, S-P2-05: 1.5h, S-P2-06: 1.5h, S-P2-07: 1.5h, S-P2-08: 0.5h)

---

### Sprint 6: Loan Officer Dashboard (Phase 2, Week 4)

**Sprint Goal:** Build the React UI for application management and review workflow.

**Note:** With corrected dependencies, UI build infrastructure (S-P2-09) depends only on S-P1-01 (Project Scaffolding), not on backend Phase 2 work. This means UI work can overlap with Sprint 4-5 backend work. Sprint 6 is shown here for logical grouping, but the UI track can start as early as Sprint 1 completion.

**Stories Included:**
- S-P2-09: UI Build Infrastructure
- S-P2-10: UI Auth and Routing
- S-P2-11: Application List and Detail Views
- S-P2-12: Review Panel
- S-P2-13: Phase 2 Integration Test

**Parallelization Map:**
- Sequential: S-P2-09 → S-P2-10 → S-P2-11, S-P2-12 (parallel) → S-P2-13

**Sprint Exit Criteria:**
- [ ] UI builds and runs with Vite
- [ ] Authentication works with API keys
- [ ] Application list displays with status
- [ ] Agent analysis is visible in detail view
- [ ] Review actions work end-to-end
- [ ] Integration test verifies user flow

**Estimated Effort:** 7 hours (S-P2-09: 1h, S-P2-10: 1.5h, S-P2-11: 2h, S-P2-12: 1.5h, S-P2-13: 1h)

**Phase 2 Complete After Sprint 6**

---

### Sprint 7: Compliance and Risk Intelligence (Phase 3a, Week 5)

**Sprint Goal:** Add compliance checking, fraud detection, and denial coaching agents with RAG support.

**Note:** With corrected dependencies, RAG infrastructure (S-P3a-01) depends only on S-P1-19 (Phase 1 Integration Test), not on Phase 2. This means RAG infrastructure can start as early as Sprint 4-5, overlapping with Phase 2 backend and UI work.

**Stories Included:**
- S-P3a-01: Knowledge Base and RAG Infrastructure
- S-P3a-02: Compliance Checking Agent
- S-P3a-03: Fraud Detection Agent
- S-P3a-04: Denial Coaching Agent
- S-P3a-05: Cyclic Document Resubmission

**Parallelization Map:**
- Start: S-P3a-01
- After S-P3a-01, parallel: S-P3a-02, S-P3a-03, S-P3a-04
- After all three agents: S-P3a-05

**Sprint Exit Criteria:**
- [ ] RAG knowledge base is queryable with citations
- [ ] Compliance agent verifies against regulations
- [ ] Fraud detection flags suspicious patterns
- [ ] Denial coaching provides actionable recommendations
- [ ] Document resubmission cycles back to processing

**Estimated Effort:** 7.5 hours (S-P3a-01: 2h, S-P3a-02: 1.5h, S-P3a-03: 1.5h, S-P3a-04: 1.5h, S-P3a-05: 1h)

**Phase 3a Complete After Sprint 7**

---

### Sprint 8: Public Access Tier (Phase 3b, Week 6)

**Sprint Goal:** Implement public-facing intake agent, calculator, and rate limiting.

**Note:** With corrected dependencies, FRED API client (S-P3b-01) depends on S-P2-08 (end of Phase 2 backend), not on Phase 3a. This means FRED client and calculator work can overlap with Phase 3a agent implementation, reducing total elapsed time.

**Stories Included:**
- S-P3b-01: FRED API Client
- S-P3b-02: Mortgage Calculator API and Tool
- S-P3b-03: Intake Graph Implementation
- S-P3b-04: Rate Limiting and Session Management
- S-P3b-05: Prompt Injection Defenses
- S-P3b-06: Streaming Chat (SSE)
- S-P3b-07: Public Landing Page

**Parallelization Map:**
- Start: S-P3b-01, S-P3b-02 (parallel)
- After S-P3b-01 and S-P3b-02: S-P3b-03
- After S-P3b-03, parallel: S-P3b-04, S-P3b-05, S-P3b-06
- After all three: S-P3b-07

**Sprint Exit Criteria:**
- [ ] FRED API provides live mortgage rates
- [ ] Calculator computes payment scenarios
- [ ] Intake agent answers mortgage questions with citations
- [ ] Rate limiting enforces session and IP caps
- [ ] Prompt injection attacks are blocked
- [ ] Chat responses stream incrementally
- [ ] Public landing page is accessible without auth

**Estimated Effort:** 9.5 hours (S-P3b-01: 1h, S-P3b-02: 1.5h, S-P3b-03: 2h, S-P3b-04: 1.5h, S-P3b-05: 1.5h, S-P3b-06: 1h, S-P3b-07: 1h)

**Phase 3b Complete After Sprint 8**

---

### Sprint 9: Operability and Polish (Phase 4, Week 7-8)

**Sprint Goal:** Add observability, admin tools, deployment manifests, and CI pipeline.

**Note:** Each story should be submitted as an individual PR for meaningful review, not batched.

**Stories Included:**
- S-P4-01: LangFuse Integration
- S-P4-02: Audit Trail Export
- S-P4-03: Admin Configuration UI
- S-P4-04: Portfolio Analytics Dashboard
- S-P4-05: Cross-Session Chat Context
- S-P4-06: Seed Data Expansion
- S-P4-07: Container Deployment Manifests
- S-P4-08: CI Pipeline
- S-P4-09: API Key Lifecycle Management
- S-P4-10: Documentation Polish

**Parallelization Map:**
- High parallelism sprint -- most stories are independent:
  - Parallel group 1: S-P4-01, S-P4-02, S-P4-06, S-P4-07, S-P4-08
  - Parallel group 2: S-P4-03, S-P4-04, S-P4-09 (all admin features)
  - After all: S-P4-05 (may depend on some infrastructure), S-P4-10 (final polish)

**Sprint Exit Criteria:**
- [ ] LangFuse dashboard shows all LLM traces
- [ ] Audit trails export as documents
- [ ] Admin UI allows threshold and knowledge base management
- [ ] Analytics dashboard shows portfolio metrics
- [ ] Chat context persists for authenticated users
- [ ] Seed data covers diverse edge cases
- [ ] Helm charts deploy to container platforms
- [ ] CI pipeline passes on main branch
- [ ] API keys support rotation
- [ ] Documentation is comprehensive

**Estimated Effort:** 13 hours (S-P4-01: 1.5h, S-P4-02: 1h, S-P4-03: 2h, S-P4-04: 1.5h, S-P4-05: 1.5h, S-P4-06: 1h, S-P4-07: 1.5h, S-P4-08: 1h, S-P4-09: 1h, S-P4-10: 1h)

**Phase 4 Complete After Sprint 9**

---

## 4. Critical Path Analysis

The critical path is the longest sequential chain of dependent stories. This determines the minimum elapsed time to complete the project.

### Critical Path

After applying dependency corrections (TL-01, TL-03, TL-06, TL-08, TL-09, TL-10, TL-11), the critical path runs through the backend track. The UI track (S-P2-09 through S-P2-13) and Phase 3a (S-P3a-01 through S-P3a-05) are now parallel tracks, not on the critical path.

```
S-P1-01 (0.5h)
  → S-P1-02 (1h)
    → S-P1-03 (1.5h)
      → S-P1-04 (1h)
        → S-P1-06 (1.5h)
          → S-P1-08 (1h)
            → S-P1-09 (1.5h)
              → S-P1-10 (1h)
                → S-P1-11 (1h)
                  → S-P1-12 (1h)
                    → S-P1-13 (1.5h)
                      → S-P1-16 (1h)
                        → S-P1-17 (2h)
                          → S-P1-18 (1h)
                            → S-P1-19 (1.5h)
                              → S-P2-01 (1.5h)
                                → S-P2-02 (2h)
                                  → S-P2-03 (1h)
                                    → S-P2-04 (1.5h)
                                      → S-P2-06 (1.5h)
                                        → S-P2-07 (1.5h)
                                          → S-P2-08 (0.5h)
                                            → S-P3b-01 (1h)
                                              → S-P3b-02 (1.5h)
                                                → S-P3b-03 (2h)
                                                  → S-P3b-04 (1.5h)
                                                    → S-P3b-07 (1h)
```

**Critical Path Total:** ~36 hours

**Parallel tracks (not on critical path):**
- UI track: S-P2-09(1h) → S-P2-10(1.5h) → S-P2-11(2h)/S-P2-12(1.5h) → S-P2-13(1h) = ~6h (starts from S-P1-01)
- Phase 3a: S-P3a-01(2h) → S-P3a-02/03/04(1.5h each) → S-P3a-05(1h) = ~4.5h (starts from S-P1-19)
- S-P1-05(1h) → S-P1-07(1.5h) run in parallel with the S-P1-06 → S-P1-08 chain

### Bottleneck Stories

Stories that block the most downstream work:

1. **S-P1-02 (Docker Compose)** -- Blocks all database and API work
2. **S-P1-04 (Migrations)** -- Blocks all repository and route work
3. **S-P1-17 (LangGraph Graph Definition)** -- Blocks all agent replacement work
4. **S-P1-19 (Phase 1 Integration Test)** -- Gates Phase 2 and Phase 3a start
5. **S-P2-06 (Confidence-Based Routing)** -- Blocks review workflow and Phase 3b
6. **S-P3a-01 (RAG Infrastructure)** -- Blocks all RAG-dependent agents
7. **S-P3b-03 (Intake Graph)** -- Blocks public tier features

### Parallelization Opportunities

High-value opportunities to reduce elapsed time:

**Phase 1:**
- After S-P1-04, parallelize S-P1-05 (seed data), S-P1-06 (core repos), and S-P1-07 (supporting repos)
- S-P1-07 can also run in parallel with S-P1-08 and S-P1-09 (no dependency between them)
- After S-P1-13, parallelize S-P1-14 (document upload) and S-P1-15 (MinIO client)

**Cross-Phase (major unlock from dependency corrections):**
- UI track (S-P2-09 through S-P2-13) depends only on S-P1-01 and can run in parallel with all backend Phase 2 work -- this is the largest parallelism unlock
- Phase 3a RAG infrastructure (S-P3a-01) depends only on S-P1-19 and can start during Phase 2 backend work
- Phase 3b FRED client (S-P3b-01) depends on S-P2-08 and can overlap with Phase 3a agent implementation

**Phase 2 Backend:**
- After S-P2-03, parallelize S-P2-04 (credit) and S-P2-05 (risk) agents
- After S-P2-06, parallelize S-P2-07 (review queue) and S-P2-08 (fraud config)

**Phase 2 UI (parallel track):**
- After S-P2-10, parallelize S-P2-11 (app views) and S-P2-12 (review panel)

**Phase 3a:**
- After S-P3a-01, parallelize S-P3a-02, S-P3a-03, S-P3a-04 (all three agents)

**Phase 3b:**
- After S-P3b-03, parallelize S-P3b-04, S-P3b-05, S-P3b-06 (rate limit, defenses, streaming)

**Phase 4:**
- High parallelism -- up to 5 stories can run simultaneously (observability, admin features, deployment, CI)

---

## 5. Risk Register

### Sprint 1 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P1-01 | Dependency resolution conflicts between api and db packages | Medium | High | Pin versions explicitly; test in clean environment |
| S-P1-02 | Docker Compose health checks may timeout on slow machines | Low | Medium | Increase health check timeout; document minimum system requirements |
| S-P1-04 | Migration schema design errors expensive to fix in later phases | Medium | High | Code review by database-engineer and architect before merge |

### Sprint 2 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P1-11 | API key lookup performance degrades at scale | Low | Medium | Redis caching implemented from day one; monitor latency |
| S-P1-12 | RBAC hierarchy misunderstood, permissions incorrect | Medium | High | Unit tests for all role combinations; security-engineer review |

### Sprint 3 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P1-17 | LangGraph checkpoint schema incompatible with future phases | High | Critical | Design checkpoint schema forward-compatible from Phase 1; see TD Section 9 |
| S-P1-17 | LangGraph graph definition is the most complex Phase 1 WU; tight 2h estimate on critical path | Medium | High | Pre-load implementing agent with LangGraph context; consider 2.5h buffer if unfamiliar |
| S-P1-19 | Integration test flaky due to async timing issues | Medium | Medium | Use polling with timeout instead of sleep; clear state between tests |

### Sprint 4 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P2-01 | PII redaction approach fails to detect all PII types | Medium | High | Design spike to validate approach; security-engineer review before implementation |
| S-P2-02 | Vision model accuracy lower than expected on diverse documents | High | High | Test with diverse document set early; plan fallback extraction strategies |
| S-P2-02 | LLM response validation logic is complex; non-deterministic outputs make testing fragile | High | Medium | Use Pydantic models for response validation; mock LLM responses in tests with known-good and known-bad shapes |

### Sprint 5 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P2-06 | Confidence threshold calibration requires extensive tuning | High | Medium | Start with conservative defaults; document tuning process; plan time for adjustment |
| S-P2-07 | Review queue complexity underestimated | Medium | Medium | Break into smaller tasks if story exceeds 2 hours |

### Sprint 6 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P2-11 | UI component complexity exceeds estimate | Medium | Medium | Use shadcn/ui for pre-built components; focus on functionality over polish |
| S-P2-13 | Integration test brittle due to UI state management | Medium | High | Use Playwright best practices; isolate test data |

### Sprint 7 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P3a-01 | Knowledge base quality directly determines compliance checking quality | High | High | Curate regulatory documents carefully; validate RAG retrieval accuracy |
| S-P3a-03 | Fraud detection false positive rate too high | High | Medium | Configurable sensitivity from day one; monitor metrics during testing |

### Sprint 8 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P3b-04 | Rate limiting calibration wrong, causes either abuse or poor UX | High | High | Start conservative; monitor usage patterns; adjust based on data |
| S-P3b-05 | Prompt injection defenses bypass rate limiting | Medium | High | Security-engineer review; test with known attack patterns |
| S-P3b-03 | Intake graph design requires more complexity than estimated | Medium | High | Reuse patterns from loan processing graph; keep intake simpler |

### Sprint 9 Risks

| Story | Risk | Likelihood | Impact | Mitigation |
|-------|------|------------|--------|------------|
| S-P4-07 | Helm charts don't work on target container platform | Medium | High | Test deployment early; verify against OpenShift constraints |
| S-P4-08 | CI pipeline too slow, blocks development velocity | Low | Medium | Optimize test execution; consider parallel jobs |

---

## 6. Story Details by Phase

### Phase 1: Foundation (Stories S-P1-01 through S-P1-20)

#### S-P1-01: Project Scaffolding
**Epic:** E-P1-01
**WU Reference:** P1-WU01
**Implementing Agent:** devops-engineer
**Priority:** P0
**Dependencies:** None
**User Stories Satisfied:** US-068 (partial)
**Exit Condition:** `cd packages/api && uv sync && python -c "import src; print('api importable')" && cd ../../packages/db && uv sync && python -c "import src; print('db importable')" && cd ../../packages/ui && pnpm install`
**Estimated Effort:** S (0.5h)

**Description:**
Create the monorepo package skeleton for api, db, and ui with their respective build configuration files. Set up pyproject.toml for Python packages with correct dependencies and package.json for UI.

---

#### S-P1-02: Docker Compose and Build System
**Epic:** E-P1-01
**WU Reference:** P1-WU02
**Implementing Agent:** devops-engineer
**Priority:** P0
**Dependencies:** S-P1-01
**User Stories Satisfied:** US-068 (partial)
**Exit Condition:** `make containers-up && sleep 10 && docker compose ps --format '{{.Service}} {{.Status}}' | grep -c "healthy"` (Expected: 3)
**Estimated Effort:** M (1h)

**Description:**
Implement Docker Compose for local infrastructure (PostgreSQL with pgvector, Redis with authentication, MinIO) and Makefile with standard development commands. Add Turborepo pipeline configuration.

---

#### S-P1-03: ORM Models
**Epic:** E-P1-02
**WU Reference:** P1-WU03
**Implementing Agent:** database-engineer
**Priority:** P0
**Dependencies:** S-P1-02
**User Stories Satisfied:** US-073 (partial)
**Exit Condition:** `cd packages/db && pytest tests/unit/test_models.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Create SQLAlchemy ORM models for all tables in architecture Section 3.1: applications, documents, agent_decisions, audit_events, users, api_keys, configuration, embeddings. Create `tests/unit/test_models.py` to verify all models import and compile correctly.

---

#### S-P1-04: Database Migrations
**Epic:** E-P1-02
**WU Reference:** P1-WU04
**Implementing Agent:** database-engineer
**Priority:** P0
**Dependencies:** S-P1-03
**User Stories Satisfied:** US-073 (partial)
**Exit Condition:** `make db-upgrade && make db-rollback && make db-upgrade`
**Estimated Effort:** M (1h)

**Description:**
Create Alembic initial migration with all tables, indexes, and permission grants. Ensure migrations are idempotent and support rollback. Enforce audit_events append-only at database level.

---

#### S-P1-05: Seed Data
**Epic:** E-P1-02
**WU Reference:** P1-WU05
**Implementing Agent:** database-engineer
**Priority:** P0
**Dependencies:** S-P1-04
**User Stories Satisfied:** US-073 (partial)
**Exit Condition:** `make db-seed && psql $DATABASE_URL -c "SELECT COUNT(*) FROM users;" | grep -q 3`
**Estimated Effort:** M (1h)

**Description:**
Create seed data migration with three users (loan officer, senior underwriter, reviewer) and their corresponding API keys. Include startup warning detection for default keys.

---

#### S-P1-06: Core Database Repositories
**Epic:** E-P1-02
**WU Reference:** P1-WU06a
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-04
**User Stories Satisfied:** US-021 (partial)
**Exit Condition:** `cd packages/db && pytest tests/unit/test_repositories.py -v -k "application or audit"`
**Estimated Effort:** M (1.5h)

**Description:**
Implement ApplicationRepository and AuditRepository with all methods from interface contract Section 2.3. Include unit tests for CRUD operations and audit append-only enforcement.

---

#### S-P1-07: Supporting Database Repositories
**Epic:** E-P1-02
**WU Reference:** P1-WU06b
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-04
**User Stories Satisfied:** US-021 (partial)
**Exit Condition:** `cd packages/db && pytest tests/unit/test_repositories.py -v -k "document or user or agent_decision or configuration"`
**Estimated Effort:** M (1.5h)

**Description:**
Implement DocumentRepository, UserRepository, AgentDecisionRepository, and ConfigurationRepository with all methods from interface contract Section 2.3. Include unit tests.

---

#### S-P1-08: Settings and Dependency Injection
**Epic:** E-P1-03
**WU Reference:** P1-WU07
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-06
**User Stories Satisfied:** US-068 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_settings.py -v && pytest tests/unit/test_dependencies.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement Pydantic Settings for environment configuration and FastAPI dependency injection functions. Include validation for required settings and default value handling.

---

#### S-P1-09: Health Checks and Error Handling
**Epic:** E-P1-03
**WU Reference:** P1-WU08
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-08
**User Stories Satisfied:** US-065 (verify against requirements.md -- TD maps P1-WU08 to US-067)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_health.py -v && pytest tests/unit/test_errors.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement `/health` and `/ready` endpoints with dependency health checks. Create RFC 7807 error response hierarchy and global exception handler.

---

#### S-P1-10: Correlation ID and Logging
**Epic:** E-P1-03
**WU Reference:** P1-WU09
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-09
**User Stories Satisfied:** US-067 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_middleware.py -v -k "correlation or logging"`
**Estimated Effort:** M (1h)

**Description:**
Implement correlation ID middleware (reads X-Request-ID or generates UUID) and structured logging middleware. Ensure correlation ID propagates through async context.

---

#### S-P1-11: Authentication Middleware
**Epic:** E-P1-03
**WU Reference:** P1-WU10
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-10, S-P1-05
**User Stories Satisfied:** US-058, US-059
**Exit Condition:** `cd packages/api && pytest tests/unit/test_auth.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement authentication middleware that validates Bearer tokens against api_keys table with Redis caching. Attach AuthenticatedUser to request state. Skip public paths.

---

#### S-P1-12: RBAC Middleware
**Epic:** E-P1-03
**WU Reference:** P1-WU11
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-11
**User Stories Satisfied:** US-060
**Exit Condition:** `cd packages/api && pytest tests/unit/test_authorization.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement authorization middleware and role requirement decorator. Enforce role hierarchy (loan_officer < senior_underwriter < reviewer). Return 403 on insufficient permissions.

---

#### S-P1-13: Application CRUD Routes
**Epic:** E-P1-04
**WU Reference:** P1-WU12
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-12
**User Stories Satisfied:** US-021, US-064
**Exit Condition:** `cd packages/api && pytest tests/unit/test_routes_applications.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement POST /v1/applications, GET /v1/applications, GET /v1/applications/{id} routes with Pydantic validation. Record audit events for creation. Encrypt SSN at rest.

---

#### S-P1-14: Document Upload Route
**Epic:** E-P1-04
**WU Reference:** P1-WU13
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-13, S-P1-15
**User Stories Satisfied:** US-064 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_routes_documents.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement POST /v1/applications/{id}/documents for multipart file upload. Store in MinIO, create document record, record audit event. Validate file size and type.

---

#### S-P1-15: MinIO Client Service
**Epic:** E-P1-04
**WU Reference:** P1-WU14
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-02
**User Stories Satisfied:** US-064 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_minio_client.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement MinIO client wrapper with upload, download, presigned URL generation. Initialize bucket on startup if it doesn't exist. Handle connection errors gracefully.

---

#### S-P1-16: Audit Trail Route
**Epic:** E-P1-04
**WU Reference:** P1-WU15
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-13
**User Stories Satisfied:** US-053, US-054
**Exit Condition:** `cd packages/api && pytest tests/unit/test_routes_audit.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement GET /v1/applications/{id}/audit with cursor-based pagination. Return audit events in reverse chronological order. No mutation endpoints (append-only).

---

#### S-P1-17: LangGraph Graph Definition
**Epic:** E-P1-05
**WU Reference:** P1-WU16
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-16
**User Stories Satisfied:** US-001, US-002, US-006, US-008
**Exit Condition:** `cd packages/api && pytest tests/unit/test_loan_processing_graph.py -v`
**Estimated Effort:** L (2h)

**Description:**
Implement LoanProcessingState TypedDict and full graph structure per architecture Section 5.1 with stub nodes. Configure PostgresSaver for checkpoint persistence. Ensure forward-compatible schema.

---

#### S-P1-18: Application Submit Endpoint
**Epic:** E-P1-05
**WU Reference:** P1-WU17
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-17
**User Stories Satisfied:** US-001 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_routes_submit.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement POST /v1/applications/{id}/submit that invokes the loan processing graph. Update application status to PROCESSING. Return immediately with 202 Accepted.

---

#### S-P1-19: Phase 1 Integration Test
**Epic:** E-P1-05
**WU Reference:** P1-WU18
**Implementing Agent:** test-engineer
**Priority:** P0
**Dependencies:** S-P1-18
**User Stories Satisfied:** US-081
**Exit Condition:** `cd packages/api && pytest tests/integration/test_phase1_flow.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Create end-to-end integration test: create application, upload document, submit for processing, verify workflow executes through all stub nodes, verify audit trail completeness.

---

#### S-P1-20: Developer Documentation
**Epic:** E-P1-05
**WU Reference:** P1-WU19
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-19
**User Stories Satisfied:** US-068, US-069
**Exit Condition:** `test -f README.md && test -f .env.example && grep -q DATABASE_URL .env.example && grep -q REDIS_PASSWORD .env.example && grep -q SSN_ENCRYPTION_KEY .env.example && grep -q ANTHROPIC_API_KEY .env.example && grep -q "make setup" README.md`
**Estimated Effort:** M (1h)

**Description:**
Write comprehensive README with architecture overview, quickstart guide, troubleshooting section. Create .env.example with all required variables documented. Include REDIS_PASSWORD.

---

### Phase 2: Core AI Agents (Stories S-P2-01 through S-P2-13)

#### S-P2-01: PII Redaction Service
**Epic:** E-P2-01
**WU Reference:** P2-WU01
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-19
**User Stories Satisfied:** US-012 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_pii_redaction.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement PII redaction service that processes document images before external LLM calls. Use local OCR/NER pipeline or multi-pass LLM approach per design decision. Store redacted versions in MinIO.

---

#### S-P2-02: Document Processing Agent
**Epic:** E-P2-01
**WU Reference:** P2-WU02
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-01
**User Stories Satisfied:** US-012, US-013, US-014
**Exit Condition:** `cd packages/api && pytest tests/unit/test_document_processing_agent.py -v`
**Estimated Effort:** L (2h)

**Description:**
Replace document processing stub with vision model integration. Classify documents into types with confidence scores. Extract structured data with per-field confidence. Persist results.

---

#### S-P2-03: Mock Credit Bureau Service
**Epic:** E-P2-02
**WU Reference:** P2-WU03
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-02
**User Stories Satisfied:** US-016 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_credit_bureau_mock.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement mock credit bureau service following Protocol pattern. Generate realistic credit reports with randomized scores, payment history, derogatory marks. Include diverse test cases.

---

#### S-P2-04: Credit Analysis Agent
**Epic:** E-P2-02
**WU Reference:** P2-WU04
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-03
**User Stories Satisfied:** US-016, US-017, US-018
**Exit Condition:** `cd packages/api && pytest tests/unit/test_credit_analysis_agent.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Replace credit analysis stub with LLM-based credit evaluation. Analyze credit score, payment history, trends. Produce confidence-scored recommendation with reasoning. Record agent decision.

---

#### S-P2-05: Risk Assessment Agent
**Epic:** E-P2-02
**WU Reference:** P2-WU05
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-03
**User Stories Satisfied:** US-019, US-020
**Exit Condition:** `cd packages/api && pytest tests/unit/test_risk_assessment_agent.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Replace risk assessment stub with financial ratio calculations (DTI, LTV) and risk scoring. Cross-validate income from multiple sources. Produce confidence-scored recommendation with reasoning.

---

#### S-P2-06: Confidence-Based Routing
**Epic:** E-P2-03
**WU Reference:** P2-WU06
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-04, S-P2-05
**User Stories Satisfied:** US-031, US-032, US-033, US-034
**Exit Condition:** `cd packages/api && pytest tests/unit/test_routing_logic.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Replace routing stub with real confidence-based decision logic. Compare aggregated confidence against configurable thresholds. Route to auto-approval, escalation, or denial. Handle agent disagreements. Also implements GET/PUT /v1/config/thresholds endpoints for threshold management (reviewer role required).

---

#### S-P2-07: Review Queue and Human Review Actions
**Epic:** E-P2-03
**WU Reference:** P2-WU07
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-06
**User Stories Satisfied:** US-035, US-036, US-037, US-038, US-039, US-040
**Exit Condition:** `cd packages/api && pytest tests/unit/test_review_routes.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement GET /v1/review-queue with role-based filtering and POST /v1/applications/{id}/review for human decisions. Support approve, deny, request more documents. Record audit events.

---

#### S-P2-08: Fraud Sensitivity Config
**Epic:** E-P2-03
**WU Reference:** P2-WU08
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-07
**User Stories Satisfied:** US-057 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_fraud_config.py -v`
**Estimated Effort:** S (0.5h)

**Description:**
Implement PUT /v1/config/fraud-sensitivity endpoint for reviewer role. Store configuration in database. Record audit event on changes.

---

#### S-P2-09: UI Build Infrastructure
**Epic:** E-P2-04
**WU Reference:** P2-WU09a
**Implementing Agent:** frontend-developer
**Priority:** P0
**Dependencies:** S-P1-01
**User Stories Satisfied:** US-041 (partial)
**Exit Condition:** `cd packages/ui && pnpm build && npx tsc --noEmit`
**Estimated Effort:** M (1h)

**Description:**
Set up Vite build configuration, Tailwind CSS, TypeScript config, ESLint, and Prettier. Configure TanStack Router and TanStack Query. Create basic app shell.

---

#### S-P2-10: UI Auth and Routing
**Epic:** E-P2-04
**WU Reference:** P2-WU09b
**Implementing Agent:** frontend-developer
**Priority:** P0
**Dependencies:** S-P2-09
**User Stories Satisfied:** US-041 (partial)
**Exit Condition:** `cd packages/ui && pnpm test -- auth.test.tsx`
**Estimated Effort:** M (1.5h)

**Description:**
Implement authentication context that stores API key in localStorage. Configure TanStack Router with protected routes. Add API client with Authorization header injection.

---

#### S-P2-11: Application List and Detail Views
**Epic:** E-P2-04
**WU Reference:** P2-WU10
**Implementing Agent:** frontend-developer
**Priority:** P0
**Dependencies:** S-P2-10
**User Stories Satisfied:** US-041, US-042, US-043, US-044, US-045
**Exit Condition:** `cd packages/ui && pnpm test -- applications.test.tsx`
**Estimated Effort:** L (2h)

**Description:**
Build application list view with status filters and detail view showing agent analysis with confidence scores. Use shadcn/ui components. Display document list with classification results.

---

#### S-P2-12: Review Panel
**Epic:** E-P2-04
**WU Reference:** P2-WU11
**Implementing Agent:** frontend-developer
**Priority:** P0
**Dependencies:** S-P2-10
**User Stories Satisfied:** US-046, US-047, US-048, US-049
**Exit Condition:** `cd packages/ui && pnpm test -- review-panel.test.tsx`
**Estimated Effort:** M (1.5h)

**Description:**
Build review queue view and review panel with approve/deny/request-docs actions. Display escalation reason and agent summaries. Require rationale text for all decisions.

---

#### S-P2-13: Phase 2 Integration Test
**Epic:** E-P2-04
**WU Reference:** P2-WU12
**Implementing Agent:** test-engineer
**Priority:** P0
**Dependencies:** S-P2-11, S-P2-12
**User Stories Satisfied:** US-081
**Exit Condition:** `cd packages/ui && pnpm playwright test`
**Estimated Effort:** M (1h)

**Description:**
Create Playwright end-to-end test: login, create application, upload document, submit, verify processing, review escalated application, approve with rationale, verify audit trail.

---

### Phase 3a: Full Agent Suite (Stories S-P3a-01 through S-P3a-05)

#### S-P3a-01: Knowledge Base and RAG Infrastructure
**Epic:** E-P3a-01
**WU Reference:** P3a-WU01
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P1-19
**User Stories Satisfied:** US-025 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_rag_service.py -v`
**Estimated Effort:** L (2h)

**Description:**
Implement vector embeddings table, embedding generation service, and RAG retrieval service with Redis caching. Load initial compliance documents into knowledge base. Provide citation extraction.

---

#### S-P3a-02: Compliance Checking Agent
**Epic:** E-P3a-01
**WU Reference:** P3a-WU02
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3a-01
**User Stories Satisfied:** US-025, US-026, US-027, US-028
**Exit Condition:** `cd packages/api && pytest tests/unit/test_compliance_agent.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Replace compliance stub with RAG-based verification. Check decisions against regulatory requirements with specific citations. Produce confidence-scored recommendation with regulatory references.

---

#### S-P3a-03: Fraud Detection Agent
**Epic:** E-P3a-01
**WU Reference:** P3a-WU03
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3a-01
**User Stories Satisfied:** US-029, US-030
**Exit Condition:** `cd packages/api && pytest tests/unit/test_fraud_detection_agent.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Replace fraud detection stub with pattern analysis: income discrepancies, property flip patterns, document metadata anomalies. Any fraud flag forces human review. Configurable sensitivity.

---

#### S-P3a-04: Denial Coaching Agent
**Epic:** E-P3a-01
**WU Reference:** P3a-WU04
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3a-01
**User Stories Satisfied:** US-050, US-051, US-052
**Exit Condition:** `cd packages/api && pytest tests/unit/test_denial_coaching_agent.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Replace denial coaching stub with actionable recommendation generation. Provide DTI improvement strategies, down payment scenarios, credit score guidance, what-if calculations. Plain language suitable for borrowers.

---

#### S-P3a-05: Cyclic Document Resubmission
**Epic:** E-P3a-01
**WU Reference:** P3a-WU05
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3a-02, S-P3a-03, S-P3a-04
**User Stories Satisfied:** US-040 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_document_resubmission.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement workflow logic to cycle back to document processing when reviewer requests more documents. Store previous analysis history. Increment resubmission counter.

---

### Phase 3b: Public Access and External Data (Stories S-P3b-01 through S-P3b-07)

#### S-P3b-01: FRED API Client
**Epic:** E-P3b-01
**WU Reference:** P3b-WU01
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P2-08
**User Stories Satisfied:** US-078 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_fred_client.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement FRED API client for mortgage rates and economic indicators. Apply Redis caching with 1-hour TTL. Validate responses per architecture Section 2.6. Handle fallback to cached/default values.

---

#### S-P3b-02: Mortgage Calculator API and Tool
**Epic:** E-P3b-01
**WU Reference:** P3b-WU02
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3b-01
**User Stories Satisfied:** US-071, US-072
**Exit Condition:** `cd packages/api && pytest tests/unit/test_calculator.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement POST /v1/public/calculator endpoint and LangChain tool wrapper. Calculate monthly payment, total interest, DTI preview, affordability estimate. Include disclaimers in all outputs.

---

#### S-P3b-03: Intake Graph Implementation
**Epic:** E-P3b-01
**WU Reference:** P3b-WU03
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3b-02
**User Stories Satisfied:** US-061, US-062, US-063
**Exit Condition:** `cd packages/api && pytest tests/unit/test_intake_graph.py -v`
**Estimated Effort:** L (2h)

**Description:**
Implement IntakeState TypedDict and intake graph per architecture Section 5.2. Include RAG retrieval, calculator tool, sentiment analysis. Provide citations for knowledge retrieval responses.

---

#### S-P3b-04: Rate Limiting and Session Management
**Epic:** E-P3b-01
**WU Reference:** P3b-WU04
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3b-03
**User Stories Satisfied:** US-066
**Exit Condition:** `cd packages/api && pytest tests/unit/test_rate_limiting.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement rate limiting middleware with session and IP-based counters in Redis. Enforce tiered limits (per-second, per-minute, per-hour, per-day). Return 429 with X-RateLimit-* headers.

---

#### S-P3b-05: Prompt Injection Defenses
**Epic:** E-P3b-01
**WU Reference:** P3b-WU05
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3b-03
**User Stories Satisfied:** US-066 (partial)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_prompt_injection.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Implement input sanitization, output filtering, and semantic detection for prompt injection attacks. Test with known attack patterns. Security-engineer review required.

---

#### S-P3b-06: Streaming Chat (SSE)
**Epic:** E-P3b-01
**WU Reference:** P3b-WU06
**Implementing Agent:** backend-developer
**Priority:** P1
**Dependencies:** S-P3b-03
**User Stories Satisfied:** US-074
**Exit Condition:** `cd packages/api && pytest tests/unit/test_streaming_chat.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement GET /v1/public/chat/stream endpoint using Server-Sent Events. Stream LLM responses incrementally. Handle client disconnection gracefully.

---

#### S-P3b-07: Public Landing Page
**Epic:** E-P3b-01
**WU Reference:** P3b-WU07
**Implementing Agent:** frontend-developer
**Priority:** P0
**Dependencies:** S-P3b-04, S-P3b-05, S-P3b-06
**User Stories Satisfied:** US-063 (partial), US-071 (partial)
**Exit Condition:** `cd packages/ui && pnpm test -- landing.test.tsx`
**Estimated Effort:** M (1h)

**Description:**
Build public landing page with chat interface, mortgage calculator, and current rates display. No authentication required. Use streaming for chat responses. Show rate limiting feedback.

---

### Phase 4: Polish and Operability (Stories S-P4-01 through S-P4-10)

#### S-P4-01: LangFuse Integration
**Epic:** E-P4-01
**WU Reference:** P4-WU01
**Implementing Agent:** backend-developer
**Priority:** P1
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-075
**Exit Condition:** `cd packages/api && pytest tests/unit/test_langfuse_integration.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Integrate LangFuse SDK for LLM observability. Add callback handlers to all LangChain calls. Verify traces appear in LangFuse dashboard with cost and performance metrics.

---

#### S-P4-02: Audit Trail Export
**Epic:** E-P4-01
**WU Reference:** P4-WU02
**Implementing Agent:** backend-developer
**Priority:** P0
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-055, US-056
**Exit Condition:** `cd packages/api && pytest tests/unit/test_audit_export.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement GET /v1/applications/{id}/audit/export endpoint (reviewer only). Generate document with complete audit trail including agent decisions, human reviews, workflow transitions.

---

#### S-P4-03: Admin Configuration UI
**Epic:** E-P4-01
**WU Reference:** P4-WU03
**Implementing Agent:** frontend-developer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-057
**Exit Condition:** `cd packages/ui && pnpm test -- admin-config.test.tsx && npx tsc --noEmit`
**Estimated Effort:** L (2h)

**Description:**
Build admin configuration interface for reviewers. Adjust confidence thresholds, manage fraud sensitivity, view system-wide decision patterns. Record configuration changes to audit trail.

---

#### S-P4-04: Portfolio Analytics Dashboard
**Epic:** E-P4-01
**WU Reference:** P4-WU04
**Implementing Agent:** frontend-developer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** None (new P2 feature)
**Exit Condition:** `cd packages/ui && pnpm test -- analytics.test.tsx && npx tsc --noEmit`
**Estimated Effort:** M (1.5h)

**Description:**
Build portfolio analytics dashboard showing approval rates, average processing times, escalation frequency, common denial reasons. Useful for risk management leads.

---

#### S-P4-05: Cross-Session Chat Context
**Epic:** E-P4-01
**WU Reference:** P4-WU05
**Implementing Agent:** backend-developer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-076
**Exit Condition:** `cd packages/api && pytest tests/unit/test_cross_session_context.py -v`
**Estimated Effort:** M (1.5h)

**Description:**
Store authenticated user chat sessions in database. Maintain conversation context across sessions. Apply context window management for long conversations.

---

#### S-P4-06: Seed Data Expansion
**Epic:** E-P4-01
**WU Reference:** P4-WU06
**Implementing Agent:** backend-developer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-077
**Exit Condition:** `make db-seed && psql $DATABASE_URL -c "SELECT COUNT(*) FROM applications;" | grep -q 10`
**Estimated Effort:** M (1h)

**Description:**
Expand seed data with diverse test cases: high/low credit scores, various income levels, different property types, edge cases triggering escalation, fraud flags, denial coaching.

---

#### S-P4-07: Container Deployment Manifests
**Epic:** E-P4-01
**WU Reference:** P4-WU07
**Implementing Agent:** devops-engineer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-079
**Exit Condition:** `helm template deploy/helm/mortgage-quickstart | kubectl apply --dry-run=client -f -`
**Estimated Effort:** M (1.5h)

**Description:**
Create Helm charts for OpenShift/Kubernetes deployment. Include resource definitions, environment configuration, health checks, and operational documentation.

---

#### S-P4-08: CI Pipeline
**Epic:** E-P4-01
**WU Reference:** P4-WU08
**Implementing Agent:** devops-engineer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** US-080
**Exit Condition:** `act --list 2>/dev/null || cat .github/workflows/*.yml | grep -q "test"` (workflow files exist and define test jobs)
**Estimated Effort:** M (1h)

**Description:**
Create CI pipeline running tests, linting, type checking, and dependency vulnerability scanning on every push. Use GitHub Actions or equivalent.

---

#### S-P4-09: API Key Lifecycle Management
**Epic:** E-P4-01
**WU Reference:** P4-WU09
**Implementing Agent:** backend-developer
**Priority:** P2
**Dependencies:** S-P3b-07
**User Stories Satisfied:** None (P2 feature addressing MVP gap)
**Exit Condition:** `cd packages/api && pytest tests/unit/test_key_lifecycle.py -v`
**Estimated Effort:** M (1h)

**Description:**
Implement POST /v1/admin/keys (generate), DELETE /v1/admin/keys/{id} (revoke), PUT /v1/admin/keys/{id}/expiration (set expiration) endpoints. Record all lifecycle events to audit trail.

---

#### S-P4-10: Documentation Polish
**Epic:** E-P4-01
**WU Reference:** P4-WU10
**Implementing Agent:** backend-developer
**Priority:** P2
**Dependencies:** S-P4-01, S-P4-02, S-P4-03, S-P4-04, S-P4-05, S-P4-06, S-P4-07, S-P4-08, S-P4-09
**User Stories Satisfied:** US-069 (polish)
**Exit Condition:** `test -f README.md && test -f docs/api-reference.md && test -f docs/deployment-guide.md && test -f docs/troubleshooting.md`
**Estimated Effort:** M (1h)

**Description:**
Polish all documentation: README, API reference, architecture overview, deployment guide, troubleshooting. Ensure developer persona can understand and extend the system independently.

---

## Summary

This work breakdown provides:

- **11 epics** organized by capability milestone
- **55 stories** mapped to technical design work units
- **9 sprints** with clear goals and exit criteria
- **Critical path analysis** showing ~36-hour minimum duration
- **Parallelization opportunities** reducing elapsed time significantly (UI track, Phase 3a, and Phase 3b FRED/calculator all run in parallel with the backend critical path)
- **Risk register** with mitigation strategies for each sprint
- **Complete dependency mapping** ensuring correct execution order

All stories include implementing agent assignments, user story references, machine-verifiable exit conditions, and effort estimates. The breakdown is ready for import into project management tools or execution by autonomous agents.

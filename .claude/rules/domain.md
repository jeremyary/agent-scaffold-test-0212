# Mortgage Lending Domain Rules

> **Scope:** All agents. These rules encode domain-specific constraints for mortgage loan processing in a regulated US lending context.

## Financial Data Handling

- **PII protection:** SSN, date of birth, bank account numbers, and income figures are PII. Never log, cache in plaintext, or include in error responses.
- **Monetary precision:** Use `Decimal` (Python) or integer cents (TypeScript) for all financial calculations. Never use floating-point for money.
- **Currency:** All monetary values are USD. Store amounts in cents as integers in the database; format for display only at the UI layer.

## Audit Trail Requirements

- Every agent decision must produce an immutable audit event containing: timestamp, agent name, confidence score (0.0-1.0), reasoning text, and input document references.
- Every human review action must record: user identity, role, timestamp, decision (approve/deny/request-more-docs), and free-text rationale.
- Every workflow state transition must be logged with before/after states and the trigger (agent decision, human action, or timeout).
- Audit records are append-only. No updates or deletes on audit tables.

## Confidence-Based Escalation

- Each AI agent produces a confidence score between 0.0 and 1.0.
- High confidence (above configurable threshold): eligible for auto-approval.
- Low/medium confidence: pauses workflow for human review with full context.
- Any fraud flag forces human review regardless of confidence scores.
- Agent disagreements (conflicting recommendations) always escalate to human review — no automated tie-breaking.
- Confidence thresholds are configurable but changes must be recorded in the audit trail.

## Fair Lending Compliance

- The system must not use prohibited factors in lending decisions: race, color, religion, national origin, sex, marital status, age, or public assistance status (ECOA / Fair Housing Act).
- Adverse action notices are legally required when denying a loan. They must include specific reasons for denial with regulatory citations.
- All denial reasons must be traceable to objective financial criteria (DTI, LTV, credit score, employment stability) — never subjective assessments.

## Document Processing

- Uploaded documents (W-2, pay stubs, tax returns, bank statements, appraisals) must be stored in object storage (MinIO), not in the database.
- PII in document images must be redacted before sending to external LLM APIs for analysis.
- PDF metadata (creation dates, producer fields, modification history) should be examined as part of fraud detection.
- Document classification confidence must meet a minimum threshold before extraction proceeds.

## Mocked Services

- Credit bureau, email notifications, employment verification, and BatchData (by default) are mocked with realistic synthetic data.
- All mocked services must implement the same interface as the real service, enabling swap-in without code changes.
- Mocked data should include diverse test cases: high/low credit scores, various income levels, different property types, edge cases that trigger escalation.

## Role-Based Access

Three roles with hierarchical permissions:

| Role | Access Level |
|------|-------------|
| `loan_officer` | Standard processing, MEDIUM-confidence reviews |
| `senior_underwriter` | All loan officer capabilities + LOW-confidence escalations |
| `reviewer` | All capabilities + audit export, compliance reports, threshold config, knowledge base management |

## Mortgage Calculations

- Monthly payment calculations must include PITI: principal, interest, taxes, and insurance.
- DTI (Debt-to-Income) = total monthly debt payments / gross monthly income.
- LTV (Loan-to-Value) = loan amount / property appraised value.
- All calculator outputs must include legal disclaimers stating estimates are not binding offers.
- Interest rates should be sourced from FRED API (DGS10, CSUSHPISA) when available, with fallback to configured defaults.

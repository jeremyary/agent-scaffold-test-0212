# Technical Design Review -- Security Engineer

**Reviewer:** Security Engineer
**Document reviewed:** `/home/jary/redhat/git/agent-scaffold-test-0212/plans/technical-design.md`
**Date:** 2026-02-12

---

## Executive Summary

This Technical Design Document successfully translates the architecture's security controls into implementable work units. The document demonstrates strong security-first thinking by embedding security requirements directly into WU descriptions and exit conditions rather than deferring them to "security hardening" phases.

**Key Strengths:**
- SSN encryption key validation on startup (P1-WU07)
- PII redaction in validation errors specified explicitly (P1-WU08)
- Audit trail immutability enforced at database grants level (P1-WU04)
- Authentication and authorization split into separate WUs with clear contracts (P1-WU10, P1-WU11)
- PII redaction service with fail-safe behavior (P2-WU01)

**Critical Gaps:**
- **Per-session token budget** (architecture Finding #3) is missing from public tier WU descriptions
- **External API response validation** (architecture Finding #4) is missing from external integration WUs
- **Request size limits** are not specified in any middleware WU

---

## Follow-Up on Architecture Review Conditions

### Condition 1: SSN Encryption Key Management (CRITICAL - Architecture Finding #1)

**Status:** ✅ **RESOLVED**

**Location:** P1-WU07 (Application Settings and Dependency Injection)

**Evidence:**
- `SSN_ENCRYPTION_KEY` field marked as required in Settings schema (line 953)
- Explicit validation requirement: "Validate `SSN_ENCRYPTION_KEY` is base64-encoded and at least 32 bytes. Refuse to start if invalid." (line 966)
- Encryption service implementation specified in P1-WU10 with key validation (lines 1090-1094)

**Remaining gap:** Key rotation and operational security guidance are still not specified in the TD. This is acceptable for MVP since the architecture was updated with this guidance. Developers implementing P1-WU07 should reference architecture Section 6.4 for key management details.

**Verification:** Exit condition for P1-WU07 includes settings validation test. Recommend adding explicit test case: "Settings load fails if SSN_ENCRYPTION_KEY is missing or < 32 bytes."

---

### Condition 2: PII in Validation Error Messages (HIGH - Architecture Finding #2)

**Status:** ✅ **RESOLVED**

**Location:** P1-WU08 (Health Check Routes and Error Handling)

**Evidence:**
- Explicit requirement: "PII fields (`ssn`) in validation errors must show `[REDACTED]` not the actual value." (line 1014)
- Specified in the error handling WU where Pydantic validation errors are converted to RFC 7807 responses

**Verification:** Exit condition for P1-WU08 includes `test_errors.py` but does not explicitly call out PII redaction test. Recommend adding explicit test case to WU description: "Test: validation error for invalid SSN format shows `[REDACTED]` not the submitted value."

---

### Condition 3: Per-Session Token Budget (HIGH - Architecture Finding #3)

**Status:** ❌ **NOT ADDRESSED**

**Location:** P3b-WU04 (Rate Limiting and Session Management)

**Gap:** The rate limiting WU (P3b-WU04) specifies session and IP rate limits (lines 2400-2401) but does not include per-session cumulative token budget tracking. This was a HIGH-severity finding in the architecture review and a mandatory condition for approval.

**Impact:** Public tier chat endpoint has no protection against "make the LLM talk forever" attacks. An attacker could craft prompts that generate extremely long responses within the 20 message/hour limit, leading to unbounded cost.

**Recommendation:** Add to P3b-WU04 task description:
```
3. Per-session token budget:
   - Track cumulative (prompt_tokens + completion_tokens) per session in Redis at `ratelimit:tokens:session:{session_id}`.
   - Budget: 50,000 tokens per 24-hour session (consistent with architecture).
   - Check before LLM invocation; reject with 429 if budget exhausted.
   - After each LLM response, increment counter by actual token usage.
   - Include in exit condition: "Test: 51st thousand tokens in a session returns 429."
```

**Severity:** HIGH (same as architecture finding) -- this gap directly violates an approval condition from the architecture review.

---

### Condition 4: External API Response Validation (HIGH - Architecture Finding #4)

**Status:** ❌ **NOT ADDRESSED**

**Location:** P3b-WU06 (FRED Integration), P3b-WU07 (BatchData Integration), P2-WU02 (Document Processing Agent)

**Gap:** None of the WUs that call external services (FRED, BatchData, LLM APIs) specify input validation of the responses. The architecture (Section 2.6) added detailed validation requirements after the architecture review, but those requirements were not translated into WU implementation specifications.

**Impact:** Compromised or malicious external services could inject malformed data, excessively large responses, or XSS payloads into the application data.

**Recommendation:**

**For P3b-WU06 (FRED Integration):**
Add to task description:
```
3. Response validation:
   - Define Pydantic model `FredRateResponse` with required fields: `value` (float, 0.1 <= x <= 20), `timestamp` (datetime).
   - Reject responses > 1 KB (length validation).
   - On validation failure, log warning and use last cached value or hardcoded 6.5% default.
```

**For P3b-WU07 (BatchData Integration):**
Add to task description:
```
3. Response validation:
   - Define Pydantic model `PropertyDataResponse` with required fields: `address` (str, max 500 chars), `estimated_value` (int, > 0), `metadata` (dict).
   - Strip HTML tags from `address` field before storage (use `bleach.clean()`).
   - Reject responses > 10 KB.
   - On validation failure, log warning and return error to caller.
```

**For P2-WU02 (Document Processing Agent):**
Add to task description:
```
f. Validate LLM response structure:
   - GPT-4 Vision classification must return one of the enum values from DocumentType.
   - Extraction must return a dict with expected field names for the document type.
   - If response is malformed, retry up to 3 times; on exhaustion, escalate application to human review.
```

**Severity:** HIGH (same as architecture finding) -- this gap directly violates an approval condition from the architecture review.

---

## New Findings

### 1. [HIGH] Request Size Limits Not Specified in Middleware Stack

**Category:** Security Misconfiguration (OWASP #5), Lack of Resources & Rate Limiting (OWASP API #4)
**Location:** Phase 1 middleware WUs (P1-WU08 through P1-WU11)
**Description:** The architecture specified request size limits in the updated Section 2.1 (middleware stack) after the architecture review (Finding #12, SUGGESTION severity). However, no WU in Phase 1 implements these limits. Without request size limits, the API is vulnerable to memory exhaustion attacks via large JSON payloads.

**Impact:**
- Denial of service via large request body upload (e.g., 100 MB JSON in application creation request)
- Memory exhaustion on API server
- Slow request processing affects other users

**Recommendation:**
Add new WU to Phase 1: **P1-WU08b: Request Size Limit Middleware**

Insert between P1-WU08 (health/errors) and P1-WU09 (correlation ID).

**Files to create:**
- `packages/api/src/middleware/request_limits.py`

**Task description:**
```python
1. Request size limit middleware:
   - JSON request bodies: 1 MB maximum (enforced by FastAPI `max_body_size`).
   - File uploads: 10 MB per file (enforced in upload handler, already specified in P1-WU13).
   - Multipart requests: 50 MB total.
   - Returns 413 Payload Too Large if limits exceeded.
   - Logs size limit violations with correlation ID at WARN level.

2. Apply middleware globally to all routes.
```

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_request_limits.py -v
# Test: 2 MB JSON body returns 413
# Test: 1 MB JSON body succeeds
# Test: 413 response includes RFC 7807 format with type "payload-too-large"
```

**References:** OWASP API Security Top 10 - API4:2019 Lack of Resources & Rate Limiting, architecture Section 2.1

---

### 2. [MEDIUM] Redis Authentication Configuration Not Specified in compose.yml

**Category:** Security Misconfiguration (OWASP #5)
**Location:** P1-WU02 (Docker Compose)
**Description:** The Docker Compose task (P1-WU02) specifies Redis without authentication (line 733: `redis:7-alpine` with no ACL or password configuration). The architecture (Section 2.4, updated after architecture review) specifies that production Redis requires authentication, but the TD does not translate this into a configuration requirement or `.env.example` entry.

**Impact:**
- Development setup runs without Redis auth, establishing a pattern that may be copied to production
- No `.env.example` entry means developers may not realize Redis auth is configurable
- Risk of production misconfiguration (unauthenticated Redis exposed)

**Recommendation:**
Update P1-WU02 task description to add Redis authentication in `compose.yml`:

```yaml
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD:-devpassword}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-devpassword}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
```

Add to P1-WU19 (README + .env.example):
```
REDIS_PASSWORD=devpassword  # Change in production
REDIS_URL=redis://:${REDIS_PASSWORD}@localhost:6379/0
```

Update P1-WU07 Settings to parse password from REDIS_URL format.

**References:** Architecture Section 2.4 (Cache Layer), architecture review Finding #9 (MEDIUM)

---

### 3. [MEDIUM] HTTPS Enforcement and Security Headers Not Specified

**Category:** Security Misconfiguration (OWASP #5)
**Location:** No WU addresses security headers
**Description:** The architecture (Section 7.5, added after architecture review Finding #8) specifies HTTPS enforcement and HTTP security headers (HSTS, CSP, X-Frame-Options, etc.). No WU in any phase implements these requirements.

**Impact:**
- Traffic between browser and API server vulnerable to MITM in production
- Missing CSP allows XSS if input sanitization has gaps
- Missing X-Frame-Options allows clickjacking
- Missing HSTS allows protocol downgrade attacks

**Recommendation:**
Add new WU to Phase 1: **P1-WU09b: Security Headers Middleware**

Insert between P1-WU09 (correlation ID) and P1-WU10 (auth).

**Files to create:**
- `packages/api/src/middleware/security_headers.py`

**Task description:**
```python
1. Security headers middleware:
   - Add headers to all responses:
     - Strict-Transport-Security: max-age=31536000; includeSubDomains
     - Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self' https://api.anthropic.com https://api.openai.com
     - X-Content-Type-Options: nosniff
     - X-Frame-Options: DENY
     - X-XSS-Protection: 1; mode=block
     - Referrer-Policy: strict-origin-when-cross-origin

2. HTTPS enforcement (production only):
   - If `settings.ENVIRONMENT == "production"` and request scheme is HTTP, redirect to HTTPS (301).
   - Development: skip HTTPS enforcement.
```

**Exit condition:**
```bash
cd packages/api && pytest tests/unit/test_security_headers.py -v
# Test: all security headers present in response
# Test: CSP header includes allowed domains
# Test: HTTPS redirect in production mode
```

**References:** OWASP Top 10 2021 - A05:2021 Security Misconfiguration, architecture Section 7.5

---

### 4. [MEDIUM] Prompt Injection Detection Implementation Missing Semantic Analysis

**Category:** Injection (OWASP #3)
**Location:** P3b-WU05 (Prompt Injection Defenses)
**Description:** The prompt injection defenses WU (P3b-WU05, lines 2420-2444) specifies input sanitization, length validation, and output filtering but does not specify the "semantic detection" layer mentioned in the architecture's five-layer defense (architecture Section 6.3). Semantic detection uses pattern matching to identify common injection attempts like "Ignore previous instructions" or "You are now in developer mode."

**Impact:**
- Four-layer defense instead of five-layer reduces detection coverage
- Sophisticated prompt injection attacks that bypass syntactic filters may succeed
- Teaching quickstart demonstrates incomplete defense-in-depth pattern

**Recommendation:**
Add to P3b-WU05 task description:

```python
3. Semantic injection detection:
   - Define patterns for common injection attempts:
     - "ignore (previous|all) (instructions|rules|prompts)"
     - "you are now (a|in) (developer mode|jailbreak|DAN)"
     - "(system|admin|root) (prompt|mode|override)"
     - "disregard (safety|content) (policy|filter)"
   - Case-insensitive regex matching on sanitized input.
   - If pattern matched, log flagged input (hashed, not verbatim) and return 400 with message "Input rejected: potential prompt injection detected."
   - Add detection result to audit trail without logging the actual input.
```

Update exit condition:
```bash
# Test: "Ignore all previous instructions" input is rejected
# Test: "You are now in developer mode" input is rejected
# Test: semantic detection flags are logged hashed, not verbatim
```

**References:** OWASP Top 10 2021 - A03:2021 Injection, architecture Section 6.3

---

### 5. [MEDIUM] Correlation ID vs Session ID Distinction Not Specified

**Category:** Authentication Failures (OWASP #7)
**Location:** P1-WU09 (Correlation ID), P3b-WU04 (Session Management)
**Description:** The architecture review (Finding #7) flagged the need to clarify that correlation IDs are request-scoped and distinct from session IDs. P1-WU09 implements correlation IDs, and P3b-WU04 implements session management, but neither WU explicitly states that these are separate concepts with different scopes and purposes.

**Impact:**
- Implementers may conflate correlation IDs and session IDs
- Session state could be mistakenly keyed by correlation ID (implementation error)
- Privacy violation if multiple users' activity is linked via reused correlation ID

**Recommendation:**
Update P1-WU09 task description to add clarification:

```
3. Correlation ID scope and purpose:
   - Correlation IDs are request-scoped identifiers for distributed tracing.
   - They are NEVER used for session management, authorization, or state storage.
   - A new correlation ID is generated (or accepted from header) for every request.
   - Correlation IDs may be reused across requests by clients for tracing purposes.
   - Session IDs are managed separately in Phase 3b and use different Redis keys.
```

Update P3b-WU04 task description to add:

```
1. Session management:
   - Session IDs are distinct from correlation IDs.
   - Session ID is generated server-side on first request to public tier, stored in Redis at `session:{session_id}`.
   - Session ID is returned in `X-Session-ID` response header and accepted in subsequent requests.
   - Session state (rate limit counters, token budget) is keyed by session ID, never by correlation ID.
```

Add integration test to P3b-WU04 exit condition:
```
# Test: two requests with same correlation ID but different session IDs access separate session state
```

**References:** Architecture review Finding #7 (MEDIUM), architecture Section 8.2

---

### 6. [MEDIUM] Audit Trail Export Access Control Not Specified

**Category:** Broken Access Control (OWASP #1)
**Location:** P1-WU15 (Audit Trail Route)
**Description:** The audit trail route WU specifies `GET /v1/applications/{id}/audit` and `GET /v1/applications/{id}/audit/export` endpoints (lines 655-656) but does not specify access control for the export endpoint. The architecture review (Finding #5) recommended restricting export to reviewers who have reviewed the application.

**Impact:**
- Reviewer with legitimate access can export audit trails for all applications in the system
- Bulk audit export could be used to exfiltrate sensitive business data
- Violates least-privilege principle

**Recommendation:**
Add P1-WU15 to Phase 1 (currently only mentioned in dependency graph but no WU definition exists).

**P1-WU15: Audit Trail Route**

**Files to create:**
- `packages/api/src/routes/audit.py`

**Task description:**
```python
1. GET /v1/applications/{id}/audit -- List audit events.
   - Requires auth (any role).
   - Returns paginated list of `AuditEventResponse`.
   - Cursor-based pagination (default limit 50, max 200).

2. GET /v1/applications/{id}/audit/export -- Export audit trail.
   - Requires `reviewer` role (not just any authenticated user).
   - Access control: Reviewer can only export audit trails for applications they have reviewed.
   - Check: Query `audit_events` table for event_type IN ('human_approved', 'human_denied', 'documents_requested') WHERE actor_id = requesting_user.id AND application_id = {id}.
   - If no matching event found, return 403 with detail: "You can only export audit trails for applications you have reviewed."
   - Alternatively, if unrestricted export is required for compliance, document the justification in architecture and allow any reviewer to export.
   - Returns audit trail as newline-delimited JSON (NDJSON) for streaming.
   - Records audit event `AUDIT_EXPORTED` with reviewer identity.
```

**Exit condition:**
```bash
cd packages/api && pytest tests/integration/test_audit.py -v
# Test: reviewer can export audit for application they reviewed
# Test: reviewer cannot export audit for application not reviewed (403)
# Test: audit export records AUDIT_EXPORTED event
# Test: non-reviewer gets 403 on export attempt
```

**References:** Architecture review Finding #5 (MEDIUM), OWASP Top 10 2021 - A01:2021 Broken Access Control

---

### 7. [LOW] LangFuse PII Warning Missing from .env.example

**Category:** Logging Failures (OWASP #9)
**Location:** P1-WU19 (README + .env.example)
**Description:** The architecture review (Finding #10) recommended adding a warning to `.env.example` that LangFuse logs full prompts/completions and should only be used with synthetic data. P1-WU19 creates `.env.example` but does not specify the LangFuse warning.

**Impact:**
- Developers may enable LangFuse in production without realizing it logs PII
- PII transmitted to third-party tracing service (LangFuse)
- Regulatory compliance violation (GDPR, CCPA)

**Recommendation:**
Update P1-WU19 task description to include in `.env.example`:

```bash
# LangFuse integration (optional)
# LANGFUSE_PUBLIC_KEY=  # WARNING: Enabling LangFuse will log full prompts/completions.
# LANGFUSE_SECRET_KEY=  # Only use with synthetic/test data, not real PII.
```

**References:** Architecture review Finding #10 (LOW), architecture Section 8.5

---

### 8. [LOW] Container Image Pinning Not Specified

**Category:** Vulnerable Components (OWASP #6)
**Location:** P1-WU02 (Docker Compose)
**Description:** The architecture review (Finding #6) recommended pinning container images to specific SHA256 digests instead of mutable tags. P1-WU02 specifies tags (`pgvector:pg16`, `redis:7-alpine`) but not digests.

**Impact:**
- Tag mutation could introduce vulnerable versions of dependencies
- Supply chain compromise via base image substitution
- No visibility into known CVEs in container images

**Recommendation:**
Update P1-WU02 task description to use digest-pinned images:

```yaml
  postgres:
    image: pgvector/pgvector:pg16@sha256:<digest>  # Pin to specific digest
  redis:
    image: redis:7-alpine@sha256:<digest>  # Pin to specific digest
  minio:
    image: minio/minio:latest@sha256:<digest>  # Pin to specific digest
```

Add to task description:
```
4. Image digest pinning:
   - All third-party images must be pinned to SHA256 digests, not tags.
   - Digests should be updated via a controlled process (manual update + review), not automatically.
   - Document the digest update process in README.
```

Add to P1-WU19 (README):
```
## Updating Container Image Digests

To update a container image digest:
1. Pull the latest image: `docker pull pgvector/pgvector:pg16`
2. Get the digest: `docker inspect --format='{{.RepoDigests}}' pgvector/pgvector:pg16`
3. Update `compose.yml` with the new digest.
4. Test locally: `make containers-up && make test`
5. Commit the change with justification (e.g., "Update pgvector to 0.7.4 for CVE-2024-XXXX fix").
```

**References:** Architecture review Finding #6 (MEDIUM), NIST 800-190 Application Container Security Guide

---

### 9. [SUGGESTION] Separate Database Roles for Application vs Checkpointer

**Category:** Security Misconfiguration (OWASP #5)
**Location:** P1-WU04 (Alembic Migrations -- Initial Schema)
**Description:** The architecture review (Finding #11) suggested using separate database roles for application queries vs LangGraph checkpoints to implement least privilege. P1-WU04 creates `app_role` and `migration_role` but does not create a separate `checkpoint_role`.

**Impact:**
- If LangGraph checkpointer is compromised (code injection, dependency vulnerability), it has full application database access
- Reduces blast radius if either pool is exploited

**Recommendation:**
Add to P1-WU04 task description:

```sql
-- In upgrade():
CREATE ROLE checkpoint_role;
GRANT CONNECT ON DATABASE mortgage TO checkpoint_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON checkpoint_blobs, checkpoint_writes, checkpoint_metadata TO checkpoint_role;
-- checkpoint_role has NO access to application tables
```

Update P1-WU03 (engine.py) to create two engines:
```python
# Application engine
app_engine = create_async_engine(settings.DATABASE_URL, pool_size=5, max_overflow=15)

# Checkpoint engine (uses checkpoint_role)
checkpoint_dsn = settings.DATABASE_URL.replace("app_role", "checkpoint_role")  # Simplified
checkpoint_pool = await asyncpg.create_pool(dsn=checkpoint_dsn, min_size=2, max_size=5)
```

**Note:** This is a SUGGESTION (not blocking) because it adds complexity. For MVP, a single `app_role` is acceptable. Consider implementing if time permits during Phase 1.

**References:** Architecture review Finding #11 (SUGGESTION), NIST 800-53 AC-6 (Least Privilege)

---

### 10. [POSITIVE] SSN Encryption Key Validation on Startup

**Section:** P1-WU07 (Application Settings)
**Finding:** The TD correctly implements the critical security control from architecture Finding #1 by requiring SSN encryption key validation on application startup. The Settings schema marks the key as required, and the validation logic refuses to start if the key is invalid. This is a strong defense-in-depth pattern that prevents deployment with weak or missing keys.

---

### 11. [POSITIVE] PII Redaction Fail-Safe Behavior

**Section:** P2-WU01 (PII Redaction Service)
**Finding:** The PII redaction service specification (line 1569) includes the critical fail-safe behavior: "If OCR confidence is below 0.5 for any page, raise `PiiRedactionError` -- processing should escalate to human review." This demonstrates security-first thinking by failing closed rather than open. If redaction cannot be performed with confidence, the system does not proceed with potentially unredacted data.

---

### 12. [POSITIVE] Audit Trail Immutability Enforced at Three Levels

**Section:** P1-WU04 (Alembic Migrations), P1-WU06 (Database Repositories)
**Finding:** The TD faithfully translates the architecture's three-level audit immutability enforcement:
1. **Database level:** P1-WU04 specifies `REVOKE UPDATE, DELETE ON audit_events FROM app_role` (line 822)
2. **Application level:** P1-WU06 specifies `AuditRepository` with "Only `create` and `list_by_application`. No update or delete methods." (line 910)
3. **API level:** Implied by absence of PATCH/DELETE endpoints for audit events

This is the gold standard for regulated industry audit trails.

---

### 13. [POSITIVE] Default API Key Warning on Startup

**Section:** P1-WU11 (RBAC Middleware and Default Key Warning)
**Finding:** The startup check for default API keys (lines 1127-1129) addresses a common production misconfiguration vector. The ERROR-level log message is appropriately alarming and actionable. This is a practical defense against "forgot to change the defaults" mistakes.

---

## Summary of Findings

| # | Severity | Section | Summary |
|---|----------|---------|---------|
| - | ARCHITECTURE | - | Per-session token budget (arch #3) not implemented in P3b-WU04 |
| - | ARCHITECTURE | - | External API response validation (arch #4) not implemented in P2-WU02, P3b-WU06, P3b-WU07 |
| 1 | HIGH | Phase 1 | Request size limits not specified in middleware stack |
| 2 | MEDIUM | P1-WU02 | Redis authentication configuration not specified in compose.yml |
| 3 | MEDIUM | Phase 1 | HTTPS enforcement and security headers not specified |
| 4 | MEDIUM | P3b-WU05 | Prompt injection semantic analysis missing |
| 5 | MEDIUM | P1-WU09, P3b-WU04 | Correlation ID vs session ID distinction not specified |
| 6 | MEDIUM | P1-WU15 | Audit trail export access control not specified |
| 7 | LOW | P1-WU19 | LangFuse PII warning missing from .env.example |
| 8 | LOW | P1-WU02 | Container image pinning not specified |
| 9 | SUGGESTION | P1-WU04 | Separate database roles for application vs checkpointer |
| 10 | POSITIVE | P1-WU07 | SSN encryption key validation on startup |
| 11 | POSITIVE | P2-WU01 | PII redaction fail-safe behavior |
| 12 | POSITIVE | P1-WU04, P1-WU06 | Audit trail immutability enforced at three levels |
| 13 | POSITIVE | P1-WU11 | Default API key warning on startup |

---

## Architecture Review Conditions Compliance

### Summary Table

| Condition | Status | Location in TD | Notes |
|-----------|--------|----------------|-------|
| **1. SSN Key Management** (CRITICAL) | ✅ RESOLVED | P1-WU07 | Key validation on startup specified; rotation guidance in architecture |
| **2. PII in Validation Errors** (HIGH) | ✅ RESOLVED | P1-WU08 | Explicit requirement for `[REDACTED]` in validation errors |
| **3. Per-Session Token Budget** (HIGH) | ❌ NOT ADDRESSED | P3b-WU04 | **BLOCKING:** Must add token budget tracking to rate limiting WU |
| **4. External API Validation** (HIGH) | ❌ NOT ADDRESSED | P2-WU02, P3b-WU06, P3b-WU07 | **BLOCKING:** Must add Pydantic validation for external responses |

### Compliance Status

**2 of 4 conditions met.** The two unmet conditions are both HIGH severity and BLOCKING for implementation to begin.

---

## Verdict

**REQUEST CHANGES**

### Blocking Issues (Must Fix Before Implementation)

The following issues must be resolved before implementation can begin:

1. **[ARCHITECTURE CONDITION #3] Per-session token budget missing from P3b-WU04**
   - Add cumulative token tracking to public tier rate limiting WU
   - 50,000 token budget per 24-hour session
   - Check before LLM invocation, increment after response
   - Include in exit condition test

2. **[ARCHITECTURE CONDITION #4] External API response validation missing**
   - Add Pydantic validation to P2-WU02 (LLM responses), P3b-WU06 (FRED), P3b-WU07 (BatchData)
   - Define response schemas, length limits, content sanitization
   - Fail gracefully on validation error (cached value or escalation)

3. **[FINDING #1] Request size limits not specified**
   - Add new WU: P1-WU08b (Request Size Limit Middleware)
   - 1 MB JSON, 10 MB files, 50 MB multipart
   - Returns 413 on violation

### Recommended Improvements (Non-Blocking)

The following MEDIUM and LOW findings should be addressed during implementation or before production deployment:

- Finding #2: Redis authentication in compose.yml
- Finding #3: Security headers middleware (add P1-WU09b)
- Finding #4: Prompt injection semantic detection in P3b-WU05
- Finding #5: Correlation ID vs session ID clarification
- Finding #6: Audit export access control in P1-WU15
- Finding #7: LangFuse PII warning in .env.example
- Finding #8: Container image digest pinning
- Finding #9: Separate database roles (suggestion)

### Strengths

This Technical Design Document demonstrates exceptional security discipline:

- **Security-first WU design:** Security requirements are embedded in WU descriptions, not deferred to hardening phases
- **Fail-safe patterns:** PII redaction with low-confidence escalation, startup key validation, immutable audit trail
- **Machine-verifiable security:** Exit conditions include security tests (auth failures, PII redaction, audit immutability)
- **Architecture fidelity:** Successfully translates complex security architecture into implementable work units

The document is structured for success once the two blocking architecture conditions are addressed.

---

## Recommendations for Technical Design Revision

1. **Add per-session token budget to P3b-WU04** (BLOCKING)
2. **Add external API validation to P2-WU02, P3b-WU06, P3b-WU07** (BLOCKING)
3. **Add P1-WU08b for request size limits** (HIGH priority)
4. **Add P1-WU09b for security headers** (MEDIUM priority)
5. **Update P1-WU15 to specify audit export access control** (MEDIUM priority)
6. **Update P1-WU02 to include Redis authentication** (MEDIUM priority)
7. **Update P3b-WU05 to include semantic injection detection** (MEDIUM priority)
8. **Update P1-WU09 and P3b-WU04 to clarify correlation ID vs session ID** (MEDIUM priority)
9. **Update P1-WU19 to include LangFuse PII warning** (LOW priority)
10. **Update P1-WU02 to use digest-pinned container images** (LOW priority)

Once the two BLOCKING issues are resolved, the Technical Design will meet all architecture review conditions and be ready for implementation.

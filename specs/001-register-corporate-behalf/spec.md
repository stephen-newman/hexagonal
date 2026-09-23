# Feature Specification: Business Banking Registration Application

**Feature Branch**: `001-register-corporate-behalf`

**Created**: 2026-09-23

**Status**: Draft

**Input**: User description: "register a business for corporate banking;
submit business name, phone, address, trade license number; applicant
supplies name, address, phone number and one form of id (passport or
driver license). Applicant is an officer of the business or, for a sole
trader, the owner. No authentication. Persisted; unique application ID
generated." (Term "business" used throughout; submission-only scope.)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Submit business banking application (Priority: P1)

A self-declared applicant submits a business banking registration. The
applicant is either an officer of the business or, where the business is
a sole trader, the owner. There is no login or authentication. The
submission captures the business name, phone, address, and trade license
number, plus the applicant's name, address, phone number, and details of
exactly one form of ID (passport OR driver license: ID type, number,
expiry).

**Why this priority**: The only story in v1 scope — captures the full
submission value. Without it nothing downstream can ever happen.

**Independent Test**: Submit one officer-for-business application and one
owner-for-sole-trader application with all required fields; verify each
is persisted as pending with a distinct system-generated unique
application ID returned to the applicant.

**Acceptance Scenarios**:

1. **Given** an officer with business name, phone, address, trade license
number, plus applicant name, address, phone, and passport details,
**When** they submit, **Then** the system persists a pending application
and returns its unique application ID.
2. **Given** a sole-trader owner with the same field set but driver
license details instead of passport, **When** they submit as owner,
**Then** the system persists a pending application and returns its unique
application ID.
3. **Given** any missing required field (business or applicant), **When**
submission is attempted, **Then** the system rejects it and identifies
each missing field.
4. **Given** applicant type owner but business type is not sole trader,
**When** submission is attempted, **Then** the system rejects it and
explains the mismatch.
5. **Given** an ID submission that is neither passport nor driver license,
or that supplies both, or that omits number/expiry, **When** submission
is attempted, **Then** the system rejects it and states the one-ID rule.

---

### Edge Cases

- Officer applicant for a business vs owner applicant for a sole trader
mismatch (rejected at submission with explanation).
- ID document invalid, expired, or not passport/driver license (rejected
with the one-ID rule).
- Impersonation or false officer/owner claims (recorded as declared;
verification deferred to a future review feature).
- Duplicate submission for the same trade license number (flag conflict;
no silent second application).
- Unauthenticated spam or flooding of submissions (noted risk; mitigation
is a plan-level concern, not mandated here).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow submission without authentication: any
person may declare as applicant, stating applicant type (officer of the
business or owner where the business is a sole trader).
- **FR-002**: System MUST capture and validate business fields: business
name, business phone, business address, and trade license number; missing
or blank fields MUST be rejected with field-level reasons.
- **FR-003**: System MUST capture and validate applicant fields: applicant
name, applicant address, applicant phone number, and exactly one ID
document — passport XOR driver license — with ID number and expiry;
violations MUST be rejected with the one-ID rule stated.
- **FR-004**: Officer type is valid only for non-sole-trader businesses;
owner type is valid only when the business is a sole trader; mismatches
MUST be rejected with an explanation.
- **FR-005**: Every accepted submission MUST be durably persisted as a
pending application with a system-generated unique application ID that is
returned to the applicant.
- **FR-006**: System MUST detect probable duplicate applications for the
same trade license number and surface the conflict instead of silently
creating a second application.

### Key Entities

- **Business**: The business being registered for banking; business name,
business phone, business address, trade license number, business type
(non-sole-trader vs sole trader), registration status (pending in v1).
- **Banking Application**: The registration case; unique application ID,
linked business snapshot, applicant snapshot, status (pending in v1
scope), submission timestamp.
- **Applicant**: Self-declared individual; type (officer or owner),
applicant name, applicant address, applicant phone number, ID document
(type passport/driver license, ID number, expiry); recorded as declared,
verification deferred.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Applicants complete a valid submission in under 5 minutes.
- **SC-002**: 95% of first-time valid submissions pass entry validation
without rework.
- **SC-003**: Duplicates for the same trade license number are flagged in
100% of test cases.
- **SC-004**: 100% of persisted applications carry a unique application ID
with all required business and applicant fields present.

## Assumptions

- There is no authentication of the applicant in v1; impersonation risk is
accepted and verification is deferred to a future review feature.
- One application covers one business; bulk/multi-business filing is out
of scope for v1.
- Trade license number format validation follows the jurisdiction pattern
supplied at implementation time.
- Durable persistence and ID uniqueness are guaranteed by the constitution
stack (PostgreSQL; design details belong to `/speckit-plan`).
- Review/decision and status lookup are explicit non-goals of v1 and are
deferred to follow-up specs; submitted applications remain pending.

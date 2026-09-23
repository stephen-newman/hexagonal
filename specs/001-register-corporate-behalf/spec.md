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

## Clarifications

### Session 2026-09-23

- Q: How should the applicant's government ID number be handled at rest and
  in responses? → A: Store the full number encrypted at rest; return only a
  masked form (last 4 digits) in responses.
- Q: When a submission arrives for a trade license number that already has an
  application, what should the system do? → A: Replace - newest wins; the
  earlier application is deleted.
- Q: Should v1 verify the trade license number against an external registry
  service, or only validate its format? → A: Asynchronous registry check
  after persisting, via the event/command path.
- Q: When the asynchronous registry check cannot complete, what should the
  application record and do? → A: Bounded retries, then record
  NEEDS_MANUAL_REVIEW and route to a manual queue.
- Q: Should an application carry one status field covering both submission
  and registry-verification states, or separate fields? → A: Separate
  submission status plus verification outcome fields.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Submit business banking application (Priority: P1)

A self-declared applicant submits a business banking registration. The
applicant is either an officer of the business or, where the business is
a sole trader, the owner. There is no login or authentication. The
submission captures the business name, phone, address, and trade license
number, plus the applicant's name, address, phone number, and details of
exactly one form of ID (passport OR driver license: ID type, number,
expiry). The ID number is stored encrypted and is never returned in full.
Accepting a submission replaces any earlier application for the same trade
license number. After persisting, the trade license number is checked
asynchronously against an external registry, and the outcome is recorded
on the application.

**Why this priority**: The only story in v1 scope — captures the full
submission value. Without it nothing downstream can ever happen.

**Independent Test**: Submit one officer-for-business application and one
owner-for-sole-trader application with all required fields; verify each
is persisted with submission status SUBMITTED and a distinct
system-generated unique application ID returned to the applicant.

**Acceptance Scenarios**:

1. **Given** an officer with business name, phone, address, trade license
number, plus applicant name, address, phone, and passport details,
**When** they submit, **Then** the system persists an application with
submission status SUBMITTED and returns its unique application ID.
2. **Given** a sole-trader owner with the same field set but driver
license details instead of passport, **When** they submit as owner,
**Then** the system persists an application with submission status
SUBMITTED and returns its unique application ID.
3. **Given** any missing required field (business or applicant), **When**
submission is attempted, **Then** the system rejects it and identifies
each missing field.
4. **Given** applicant type owner but business type is not sole trader,
**When** submission is attempted, **Then** the system rejects it and
explains the mismatch.
5. **Given** an ID submission that is neither passport nor driver license,
or that supplies both, or that omits number/expiry, **When** submission
is attempted, **Then** the system rejects it and states the one-ID rule.
6. **Given** an earlier application exists for the same trade license
number, **When** a new submission for that number is accepted, **Then**
the earlier application is deleted and only the new one remains.
7. **Given** an accepted application, **When** the asynchronous registry
check completes with a valid license, **Then** the application records a
VERIFIED verification outcome.
8. **Given** an accepted application whose registry check cannot complete
after the allowed retries, **When** retries are exhausted, **Then** the
application records NEEDS_MANUAL_REVIEW and is routed to the manual queue.

---

### Edge Cases

- Officer applicant for a business vs owner applicant for a sole trader
mismatch (rejected at submission with explanation).
- ID document invalid, expired, or not passport/driver license (rejected
with the one-ID rule).
- Impersonation or false officer/owner claims (recorded as declared;
verification deferred to a future review feature).
- Submission for a trade license number that already has an application
(replaces it; the earlier application is deleted).
- Registry check cannot complete after the allowed retries (application is
routed to the manual queue with a NEEDS_MANUAL_REVIEW outcome).
- Registry reports the trade license number as invalid.
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
violations MUST be rejected with the one-ID rule stated. The ID number
MUST be stored encrypted at rest and MUST NEVER be returned in full;
responses MUST show only a masked form (last 4 digits).
- **FR-004**: Officer type is valid only for non-sole-trader businesses;
owner type is valid only when the business is a sole trader; mismatches
MUST be rejected with an explanation.
- **FR-005**: Every accepted submission MUST be durably persisted with
submission status SUBMITTED and a system-generated unique application ID
that is returned to the applicant.
- **FR-006**: A new accepted submission for a trade license number that
already has an application MUST replace the earlier application, which MUST
be deleted, so exactly one live application exists per trade license
number.
- **FR-007**: On acceptance, the system MUST publish a trade license
verification command through the event/command path (Kafka topics declared
in AsyncAPI) and MUST NOT block the submission response on the result.
- **FR-008**: System MUST record a verification outcome (VERIFIED, INVALID,
UNAVAILABLE, or NEEDS_MANUAL_REVIEW) for every application; after bounded
retries without a result the outcome MUST become NEEDS_MANUAL_REVIEW and
the application MUST be routed to the manual queue.

### Key Entities

- **Business**: The business being registered for banking; business name,
business phone, business address, trade license number, business type
(non-sole-trader vs sole trader). Deleted with its application when that
application is replaced.
- **Banking Application**: The registration case; unique application ID,
linked business snapshot, applicant snapshot, submission status
(SUBMITTED), verification outcome (NOT_CHECKED, VERIFIED, INVALID,
UNAVAILABLE, NEEDS_MANUAL_REVIEW), manual-queue routing, submission
timestamp.
- **Applicant**: Self-declared individual; type (officer or owner),
applicant name, applicant address, applicant phone number, ID document
(type passport/driver license, ID number stored encrypted, expiry; masked
to the last 4 digits in responses); recorded as declared, verification
deferred.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Applicants complete a valid submission in under 5 minutes.
- **SC-002**: 95% of first-time valid submissions pass entry validation
without rework.
- **SC-003**: Exactly one live application per trade license number after
any replacement, in 100% of test cases.
- **SC-004**: 100% of persisted applications carry a unique application ID
with all required business and applicant fields present.
- **SC-005**: Every accepted application reaches a recorded verification
outcome or the manual queue within 5 minutes in the pilot.
- **SC-006**: Zero responses expose a full ID number (verified by contract
tests).

## Assumptions

- There is no authentication of the applicant in v1; impersonation risk is
accepted and verification is deferred to a future review feature.
- One application covers one business; bulk/multi-business filing is out
of scope for v1.
- Trade license number format validation follows the jurisdiction pattern
supplied at implementation time.
- Durable persistence and ID uniqueness are guaranteed by the constitution
stack (PostgreSQL; design details belong to `/speckit-plan`).
- Reviewer decisions and status lookup remain explicit non-goals of v1 and
are deferred to follow-up specs; the manual queue and its NEEDS_MANUAL_REVIEW
routing ARE in scope.
- Replaced applications are hard-deleted; v1 retains no record of earlier
submissions for the same trade license number (accepted tradeoff, to be
revisited when the review feature lands).
- The external registry is reached through a separate simple proxy web
service (constitution Principle VII); registry availability is why the check
runs asynchronously and why exhausted retries route to the manual queue.
- Encryption of the ID number follows the constitution stack; the specific
encryption and key-management approach is fixed in `/speckit-plan`.

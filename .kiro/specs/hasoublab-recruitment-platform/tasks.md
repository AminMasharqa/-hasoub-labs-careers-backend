# Implementation Plan: HasoubLabs Recruitment Platform (Phase 1)

## Overview

This plan implements **Phase 1 (Requirements 1–8)** of the HasoubLabs Recruitment Platform as described in `design.md`: a modular monolith on **TypeScript/Node.js + NestJS**, backed by **PostgreSQL 15+** (via Prisma/TypeORM), **Redis** (sessions, rate limits, lockouts), and an **S3-compatible object store** (encrypted CV blobs), with a transactional **EmailPort** and pluggable **MalwareScannerPort**.

The plan is sequenced so shared infrastructure (schema, domain-event bus, hash-chained audit ledger, RBAC guard) exists before the domain services that depend on it, and endpoints come after their services. Each domain write appends its audit entry inside the same transaction (Requirement 8), and side effects (email) are emitted as post-commit domain events.

Property-based tests use **fast-check** at a **minimum of 100 iterations** per property and are tagged in the exact format:
`// Feature: hasoublabs-recruitment-platform, Property {number}: {property_text}`
Each property test corresponds to one of the 36 properties in `design.md`. Unit, integration, and smoke tests supplement per the Testing Strategy.

**Conventions**
- Tasks marked with `*` are optional (test-related sub-tasks) and can be skipped for a faster MVP; core implementation tasks are never optional.
- Each task references the requirement acceptance criteria (`_Requirements: X.Y_`) and/or design Property numbers it implements.
- Checkpoints ensure incremental validation.

## Tasks

- [ ] 1. Project and environment setup
  - Initialize the NestJS + TypeScript monorepo/app skeleton with module boundaries mirroring the design (Auth, Registration, Users, Profile, Skill, CV, Jobs, Applications, Audit, Events)
  - Configure PostgreSQL connection, Redis client, and the S3-compatible object-storage client (env-driven config, KMS/SSE settings)
  - Add ORM (Prisma or TypeORM) with explicit-transaction support; set up migration tooling
  - Configure fast-check as the property-based test library and the unit/integration test runner (jest/vitest with `--run`/single-run mode)
  - Add linting, formatting, and a base CI script that runs build + tests
  - _Requirements: Performance (500 concurrent / 3s), Security (HTTPS), design "Technology Stack"_

- [ ] 2. Database schema and migrations
  - [ ] 2.1 Create Account, RoleAssignment, and MeetingCompletion schema
    - `Account` (id, full_name, email `citext UNIQUE`, password_hash, status enum, mfa_enabled, mfa_secret encrypted, language enum, timestamps)
    - `RoleAssignment` (UNIQUE(account_id, role)) plus DB trigger enforcing Admin ⊄ {Candidate, Senior}
    - `MeetingCompletion` (account_id, recorded_by, recorded_at)
    - _Requirements: 1.5, 1.10, 1.15, 1.16; design "Account & Roles"_

  - [ ] 2.2 Create Registration & Geographic Verification schema
    - `RegistrationLink` (token UNIQUE non-guessable, role, single_use, used_at, expires_at ≤72h, created_by)
    - `GeographicVerification` (account_id UNIQUE, state enum, residency_proof_type, residency_proof_value encrypted, override_by/reason/at, updated_at) with `CHECK` on override_reason length ≥10
    - `VerificationCode` (code_hash, issued_at, expires_at, consumed_at, invalidated, attempt_count)
    - _Requirements: 1.8, 2.4, 2.5, 2.12, 2.16; design "Registration & Geographic Verification"_

  - [ ] 2.3 Create Candidate Profile and Skill Taxonomy schema
    - `CandidateProfile`, `EducationEntry` (0–20, grad_year ≥ start_year via CHECK), `WorkExperienceEntry` (0–20), `ProfileLanguage` (0–10)
    - `Skill` (taxonomy_id nullable, raw_term, normalized, needs_review), `CandidateSkill` (1–20), `SkillTaxonomy` (canonical_term UNIQUE, active)
    - _Requirements: 4.1, 4.3, 4.14; design "Candidate Profile"_

  - [ ] 2.4 Create CV Version Control schema
    - `CvVersion` (version_number, object_key, file_size, sha256_checksum, uploaded_at, scan_state, is_active, admin_designated) with `UNIQUE(account_id, version_number)` and partial unique index `WHERE is_active = true`
    - DB trigger rejecting UPDATE/DELETE of object_key/checksum/version_number rows (immutability)
    - _Requirements: 5.4, 5.5, 5.6, 5.12; design "CV Version Control"_

  - [ ] 2.5 Create Job Description and Application schema
    - `JobDescription` (all enums, openings, recruiter_email, description, status), `JobRequiredSkill` (1–20)
    - `Application` (candidate_id, jd_id, cv_version_id, status, submitted_at) with partial unique index `(candidate_id, jd_id) WHERE status IN ('Submitted','Under Review')`
    - `ApplicationStatusHistory`
    - _Requirements: 6.1, 6.2, 7.3, 7.5, 7.6; design "Job Description & Application"_

  - [ ] 2.6 Create hash-chained Audit Log schema
    - `AuditLogEntry` (BIGSERIAL id, actor_id, action, entity_type, entity_id, before_values jsonb, after_values jsonb, reason, occurred_at ms-precision, prev_hash, entry_hash)
    - DB trigger + revoked UPDATE/DELETE grants enforcing insert-only
    - _Requirements: 8.1, 8.2, 8.3, 8.8; design "Audit Log"_

- [ ] 3. Checkpoint - schema and migrations
  - Ensure all migrations apply cleanly and DB constraints/triggers are exercised by a smoke migration test. Ask the user if questions arise.

- [ ] 4. Shared infrastructure: domain-event bus and audit ledger
  - [ ] 4.1 Implement the internal domain-event bus
    - In-process publish/subscribe emitting events after transaction commit (`RegistrationSubmitted`, `EmailVerified`, `AccountStatusChanged`, `CvVersionActivated`, `CvQuarantined`, `ApplicationSubmitted`, `JdCreated`, etc.)
    - Ensure emitters are post-commit so failed side effects never roll back domain writes
    - _Requirements: design "Domain-event bus", "Transactional Email Integration"; 8.5_

  - [ ] 4.2 Implement the Audit Log Module service
    - `append(entry, tx)` computing `entry_hash = SHA-256(canonical_serialization || prev_hash)` chained to the prior entry within the domain transaction
    - `search(filters)` (actor, action, entity type/id, date range) with pagination
    - `verifyChain()` integrity check and `reconstruct(entityRef)` folding before/after deltas in timestamp order
    - Reject update/delete attempts and record the attempt as a new entry
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.6, 8.8_

  - [ ]* 4.3 Write property test for audit completeness
    - **Property 30: Audit completeness** — every entity create/modify/delete yields ≥1 referencing entry with actor, action, entity, before/after, reason, ms timestamp
    - **Validates: Requirements 1.21, 2.17, 3.9, 4.5, 5.13, 6.8, 6.12, 7.14, 8.1, 8.2**

  - [ ]* 4.4 Write property test for audit append-only and hash-chain integrity
    - **Property 31: Audit append-only and hash-chain integrity**
    - **Validates: Requirements 8.3, 8.8**

  - [ ]* 4.5 Write property test for audit state reconstruction
    - **Property 32: Audit state reconstruction** — folding before/after deltas reproduces current state
    - **Validates: Requirements 8.6**

  - [ ]* 4.6 Write property test for transaction atomicity
    - **Property 33: Transaction atomicity** — failed multi-step operation leaves no intermediate state; exactly one failure entry
    - **Validates: Requirements 8.5**

  - [ ]* 4.7 Write property test for audit search filtering
    - **Property 34: Audit search filtering** — every returned entry matches all filters; paginated
    - **Validates: Requirements 8.4**

- [ ] 5. Checkpoint - shared infra
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Auth, session, MFA, and password policy
  - [ ] 6.1 Implement password policy and hashing
    - Pure `isPasswordAcceptable(pw, breachedCheck)` (≥10 chars, ≤64+ max, breached-list screening via k-anonymity/bloom filter); bcrypt cost ≥12 on storage
    - _Requirements: Security constraint; 1.11_

  - [ ]* 6.2 Write property test for password policy predicate
    - **Property 35: Password policy predicate**
    - **Validates: Requirements 1.11**

  - [ ] 6.3 Implement session management and MFA
    - Redis-backed server sessions, 30-min idle expiry, rotation on privilege change/context switch/MFA completion
    - TOTP (RFC 6238) enrollment offered to all; challenge after primary credential check; required for Admin
    - _Requirements: Security constraints (session expiry, MFA)_

  - [ ] 6.4 Implement login, logout, password-reset endpoints with rate limiting
    - `POST /auth/login`, `/auth/logout`, `/auth/password-reset`, `/auth/mfa/verify`
    - Redis sliding-window rate limits per source IP and per account; lock account after repeated failed logins pending recovery
    - _Requirements: Security constraint (rate limiting/lockout); design "Auth & Session Module"_

  - [ ]* 6.5 Write unit tests for auth flows
    - Login success/failure, lockout threshold, session expiry, MFA-required-for-Admin gating
    - _Requirements: Security constraints_

  - [ ]* 6.6 Write smoke tests for security configuration
    - bcrypt cost ≥12, 30-minute session expiry, MFA-required-for-Admin
    - _Requirements: Security constraints_

- [ ] 7. RBAC authorization guard
  - [ ] 7.1 Implement the Authorization Guard
    - Session resolution → account-status check (only auth/code/onboarding unless `Approved`; `Suspended`/`Deactivated` see status notice only) → active-context resolution (dual-role select, single-role default, Admin→admin) → pure `authorize(roles, activeContext, target)` capability evaluation with no per-user overrides
    - Implement the Phase 1 capability matrix and Senior applicant-list field limiting (full name, applied role title, Application status only)
    - `POST /auth/context` context switch re-derives capabilities with no carryover (session rotation)
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.7, 3.10, 3.11, 1.12, 1.18, 2.13_

  - [ ] 7.2 Implement non-disclosure and default-deny behavior
    - Authorize-before-lookup ordering, single canonical 403-style error body for forbidden vs not-found on cross-owner access, constant-time response floor (padded latency)
    - Default-deny for all non-public endpoints; public whitelist = registration-via-link, code entry, login, password reset; JD endpoints never reachable unauthenticated
    - Record authorization-check failures to the Audit_Log (actor, action, target id, denied, ms timestamp)
    - _Requirements: 3.6, 3.8, 3.9_

  - [ ]* 7.3 Write property test for authorization matrix
    - **Property 11: Authorization equals the capability matrix**
    - **Validates: Requirements 3.1, 3.3, 3.4, 3.5, 3.7, 3.10, 6.5**

  - [ ]* 7.4 Write property test for no-access-before-Approved
    - **Property 2: No access before Approved**
    - **Validates: Requirements 1.12, 1.18, 3.11**

  - [ ]* 7.5 Write property test for non-disclosure of resource existence
    - **Property 12: Non-disclosure of resource existence** — identical status/body/timing regardless of existence
    - **Validates: Requirements 3.6**

  - [ ]* 7.6 Write property test for default-deny on non-public endpoints
    - **Property 13: Default-deny for non-public endpoints**
    - **Validates: Requirements 3.8**

- [ ] 8. Checkpoint - auth and RBAC
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Transactional email port
  - [ ] 9.1 Implement the EmailPort interface and provider adapter
    - Abstract `EmailPort` with a provider adapter (SES/SendGrid/Postmark); subscribe to domain events after commit for the Phase 1 minimal notification set (verification codes within 60s, meeting-arrangement notice, approval/rejection notices, application confirmations, CV quarantine/integrity alerts)
    - Retry + log on send failure; never roll back committed domain writes
    - _Requirements: Security constraint (transactional email); Phase 1 Notifications (Minimal)_

  - [ ]* 9.2 Write integration test for email dispatch
    - Mock `EmailPort`; assert send invoked with correct recipient/type and within timing budget for verification-code, approval/rejection, application-confirmation, quarantine events
    - _Requirements: Phase 1 Notifications (Minimal)_

- [ ] 10. Registration and geographic verification
  - [ ] 10.1 Implement residency-proof format validators
    - Pure validators: Israeli phone (`+972`+9 digits or `0`+9 digits), national ID (9 digits + check-digit algorithm), address (street + number + city ≤200 chars); reject when no valid proof
    - _Requirements: 2.1, 2.2, 2.3_

  - [ ]* 10.2 Write property test for residency format validation
    - **Property 7: Residency format validation**
    - **Validates: Requirements 2.1, 2.2, 2.3**

  - [ ] 10.3 Implement registration-link generation and resolution
    - Admin `POST /admin/registration-links` (non-guessable token, single-use or ≤72h expiry, role-specific); `GET /register/{linkToken}` resolves and opens the correct flow; links not publicly discoverable
    - _Requirements: 1.7, 1.8_

  - [ ] 10.4 Implement registration submission and account creation
    - `POST /register/{linkToken}`: field + password-policy + residency-format validation with field-level errors; case-insensitive email conflict rejection disclosing nothing else; create account in `PendingVerification`; create `GeographicVerification` in `PendingCode`; issue Verification_Code; emit `RegistrationSubmitted`
    - _Requirements: 1.9, 1.10, 1.11, 2.4, 2.5_

  - [ ]* 10.5 Write property test for case-insensitive email uniqueness
    - **Property 6: Case-insensitive email uniqueness**
    - **Validates: Requirements 1.10**

  - [ ] 10.6 Implement Verification_Code lifecycle
    - `POST /auth/verify-code` (correct code → `Verified` + `ApprovedPendingMeeting` + notify Admin), `POST /auth/resend-code` (new single-use code, invalidate prior, reset attempts); lock after 5 wrong attempts; 72h expiry → `Expired` + release email
    - _Requirements: 1.13, 1.14, 1.22, 1.23, 2.6, 2.7, 2.8, 2.9, 2.10_

  - [ ]* 10.7 Write property test for verification code lifecycle
    - **Property 5: Verification code lifecycle — expiry, lockout, resend**
    - **Validates: Requirements 1.14, 1.22, 1.23, 2.8, 2.9, 2.10**

  - [ ]* 10.8 Write property test for geographic verification gate invariant
    - **Property 8: Geographic verification gate invariant**
    - **Validates: Requirements 2.6, 2.13**

  - [ ] 10.9 Implement residency re-validation on residency-field change
    - When an `Approved` user changes a residency-proof field, re-run validation; on failure set `PendingCode`, email new code, notify Admin within 60s, restrict account until re-verified
    - _Requirements: 2.14, 2.15_

- [ ] 11. User and role management (Admin operations)
  - [ ] 11.1 Implement Admin lifecycle and role operations
    - `meeting-complete` (→`Approved`, record Admin + UTC), `reject` (reason ≥10 chars, notify user), `suspend`/`reactivate`, `roles` add/remove (grant capabilities on `Approved` without re-registration; enforce Admin exclusivity), `reopen` rejected/release email, `geo-override` (reason ≥10 chars → `ManualOverride`), `GET /admin/accounts` + `GET /admin/accounts/{id}` (full profile incl. residency proof + audit history)
    - All actions write audit entries inside their transaction
    - _Requirements: 1.6, 1.7, 1.16, 1.17, 1.19, 1.20, 2.11, 2.12, 4.10_

  - [ ]* 11.2 Write property test for single account status invariant
    - **Property 1: Single account status invariant**
    - **Validates: Requirements 1.15**

  - [ ]* 11.3 Write property test for meeting gate precedes Approved
    - **Property 3: Meeting gate precedes Approved**
    - **Validates: Requirements 1.16**

  - [ ]* 11.4 Write property test for Admin role exclusivity
    - **Property 4: Admin role exclusivity**
    - **Validates: Requirements 1.5**

  - [ ]* 11.5 Write property test for manual override completeness
    - **Property 9: Manual override completeness**
    - **Validates: Requirements 2.12**

  - [ ]* 11.6 Write property test for residency-proof confidentiality
    - **Property 10: Residency-proof confidentiality** — no national ID / residency-proof value in Candidate/Senior-facing responses
    - **Validates: Requirements 2.16, 3.5**

  - [ ]* 11.7 Write integration test for residency-proof encryption at rest
    - Verify national ID / residency-proof value stored encrypted and only decryptable in Admin-scoped code paths
    - _Requirements: 2.16_

- [ ] 12. Checkpoint - registration, verification, user management
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Skill taxonomy module
  - [ ] 13.1 Implement Skill Taxonomy service and lookup
    - Serve canonical `Skill_Taxonomy`; `GET /skills/search?q=`; store off-taxonomy terms linked to a normalized term and flag `needs_review`
    - _Requirements: 4.2, 4.3_

  - [ ]* 13.2 Write property test for off-taxonomy skill linking
    - **Property 17: Off-taxonomy skill linking**
    - **Validates: Requirements 4.3**

- [ ] 14. Candidate profile module
  - [ ] 14.1 Implement profile completeness and readiness predicates
    - Pure `computeCompleteness(profile)` (name 1–100, verified email, valid E.164 phone, ≥1 education, ≥1 skill) and `isApplicationReady(profile, cvCount)`
    - _Requirements: 4.6, 4.7_

  - [ ]* 14.2 Write property test for profile completeness predicate
    - **Property 14: Profile completeness predicate**
    - **Validates: Requirements 4.6, 4.8**

  - [ ]* 14.3 Write property test for Application-Ready predicate
    - **Property 15: Application-Ready predicate**
    - **Validates: Requirements 4.7, 4.9**

  - [ ] 14.4 Implement profile read/edit endpoints and validation
    - `GET /profile`, `PUT /profile`: validate email (RFC 5322 addr-spec + domain resolvability ≤5s), phone (E.164), LinkedIn (HTTPS URL), date ordering; reject invalid saves retaining prior values; persist `Draft`/`Complete` naming missing/invalid fields; block `Application-Ready`-required actions when not ready; record before/after audit; store Arabic verbatim
    - _Requirements: 4.1, 4.4, 4.5, 4.8, 4.9, 4.11, 4.12, 4.13, 4.14, 4.15; Localization constraint_

  - [ ]* 14.5 Write property test for profile field validation with prior-value retention
    - **Property 16: Profile field validation with prior-value retention**
    - **Validates: Requirements 4.11, 4.12, 4.13, 4.14**

  - [ ]* 14.6 Write property test for Arabic content storage round-trip
    - **Property 36: Arabic content storage round-trip**
    - **Validates: Requirements 4.1**

  - [ ]* 14.7 Write integration test for email domain resolvability
    - DNS domain-resolvability check within 5s budget (mocked resolver)
    - _Requirements: 4.12_

- [ ] 15. Checkpoint - profile and skills
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. CV upload and version control pipeline
  - [ ] 16.1 Implement CV validation and malware-scan pipeline
    - Content-inspect PDF (magic bytes/structure, not extension), size ≤10MB, reject password-protected/unreadable with specific violation and store nothing; `MalwareScannerPort` scan; on malware quarantine (scan_state=Quarantined), exclude from candidate storage, emit `CvQuarantined` → notify Candidate + Admin
    - _Requirements: 5.1, 5.2, 5.3_

  - [ ]* 16.2 Write property test for CV validation rejects and stores nothing
    - **Property 21: CV validation rejects and stores nothing**
    - **Validates: Requirements 5.1, 5.2, 5.3**

  - [ ] 16.3 Implement version assignment, active designation, and encrypted storage
    - `POST /cv`: compute SHA-256 while streaming; in-transaction `max(version)+1` under row lock; store encrypted object (object-lock); set new version active and clear prior Admin designation; append audit; `POST /admin/cv/{versionId}/designate-active` (Admin designation holds until newer upload/change)
    - _Requirements: 5.4, 5.5, 5.6, 5.7, 5.11, 5.13_

  - [ ]* 16.4 Write property test for CV version numbering and immutability
    - **Property 18: CV version numbering and immutability**
    - **Validates: Requirements 5.4, 5.5, 5.12**

  - [ ]* 16.5 Write property test for exactly one active CV version
    - **Property 19: Exactly one active CV version**
    - **Validates: Requirements 5.6, 5.7**

  - [ ] 16.6 Implement CV download with checksum re-verification
    - `GET /cv` (own history), `GET /cv/{versionId}/download` (Candidate own / Admin any): recompute SHA-256 on retrieval, fail + raise Admin integrity alert on mismatch; return exact original PDF
    - _Requirements: 5.8, 5.9, 5.10_

  - [ ]* 16.7 Write property test for CV upload/download integrity round-trip
    - **Property 20: CV upload/download integrity round-trip**
    - **Validates: Requirements 5.8, 5.10**

  - [ ]* 16.8 Write integration tests for object storage and scanner
    - Object-storage encryption round-trip and object-lock immutability; malware scanner integration (mocked/representative)
    - _Requirements: 5.3, 5.4, 5.11_

- [ ] 17. Checkpoint - CV pipeline
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 18. Job description module
  - [ ] 18.1 Implement JD creation and lifecycle
    - `POST /jobs` (create in `Draft`, unique id, creator + creation timestamp, all fields/enums, 1–20 required skills from taxonomy), `PUT /jobs/{id}` (creator/Admin only; audit before/after; existing applications remain associated), `POST /jobs/{id}/publish` (`Draft`→`Open`), `POST /jobs/{id}/close` (→`Closed`, closed indicator, reject new applications, no reopen); reject edit/publish/close by non-creator/non-Admin
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.12_

  - [ ]* 18.2 Write property test for JD status monotonicity
    - **Property 22: Job description status monotonicity**
    - **Validates: Requirements 6.6, 6.7**

  - [ ] 18.3 Implement JD browse, filter, and view endpoints
    - `GET /jobs` (Candidate: `Open` only, paginated 20/page, searchable by title/company/skills, AND filters for skills/location/work model/employment type/experience level), `GET /jobs/mine` (Senior: own any status + `Open`), Admin view all; `GET /jobs/{id}/applicants` (Senior own JDs limited fields, Admin full)
    - _Requirements: 6.9, 6.10, 6.11; 3.4_

  - [ ]* 18.4 Write property test for candidate browse filtering
    - **Property 23: Candidate browse filtering**
    - **Validates: Requirements 6.9, 6.10**

- [ ] 19. Self-service candidate applications
  - [ ] 19.1 Implement application submission with readiness gate and idempotency
    - `POST /jobs/{id}/apply`: enforce `Application-Ready` gate (naming unmet conditions), record candidate + jd + active CV snapshot + UTC timestamp, reject duplicate non-terminal application, reject apply to non-`Open` JD, enforce 20-per-24h rolling cap; in-app immediate + email ≤5min confirmation; notify Admin + JD creator; audit
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6, 7.8, 7.12, 7.13, 7.14_

  - [ ]* 19.2 Write property test for Application-Ready gate on submission
    - **Property 24: Application-Ready gate on submission**
    - **Validates: Requirements 7.1, 7.2**

  - [ ]* 19.3 Write property test for idempotent non-terminal application
    - **Property 25: Idempotent non-terminal application**
    - **Validates: Requirements 7.5, 7.6, 7.8**

  - [ ]* 19.4 Write property test for CV snapshot immutability on applications
    - **Property 27: CV snapshot immutability on applications**
    - **Validates: Requirements 7.3, 7.4**

  - [ ]* 19.5 Write property test for application rate cap
    - **Property 29: Application rate cap**
    - **Validates: Requirements 7.13**

  - [ ] 19.6 Implement withdrawal, listing, admin status, and close cascade
    - `POST /applications/{id}/withdraw` (own, only while `Submitted`/`Under Review` → `Withdrawn`), `GET /applications` (own list with status/title/company/date), `POST /admin/applications/{id}/status` (Admin set any status + audit); wire JD close to cascade non-terminal applications → `Closed`
    - _Requirements: 7.5, 7.7, 7.9, 7.10, 7.11_

  - [ ]* 19.7 Write property test for application withdrawal transition
    - **Property 26: Application withdrawal transition**
    - **Validates: Requirements 7.7**

  - [ ]* 19.8 Write property test for close cascade
    - **Property 28: Close cascade**
    - **Validates: Requirements 7.11**

- [ ] 20. Checkpoint - jobs and applications
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 21. Audit log search endpoint
  - [ ] 21.1 Implement Admin audit search endpoint
    - `GET /admin/audit`: parameterized filter query (actor, action, entity type/id, date range) with pagination, wired to the Audit Log Module `search`
    - _Requirements: 8.4_

- [ ] 22. Cross-cutting concerns and wiring
  - [ ] 22.1 Wire rate limiting, pagination, and HTTPS enforcement
    - Redis sliding-window limiters on login/registration/CV upload/application submission; enforce 20/page default and cap on list endpoints; enforce HTTPS-only at edge/app
    - _Requirements: Security & Performance constraints; 6.9, 8.4, 7.13_

  - [ ] 22.2 Implement localization/RTL and encryption-at-rest wiring
    - Per-account language preference (`ar`/`en`), i18n bundles, `dir="rtl"` via CSS logical properties; verify KMS-managed encryption at rest for national ID, residency proof, CV files; exclude encrypted fields from non-Admin responses
    - _Requirements: Localization & Security constraints; 2.16, 5.11_

  - [ ] 22.3 Integrate all modules end to end through the guard and event bus
    - Ensure every domain endpoint routes through the Authorization Guard and every write appends its audit entry in-transaction and emits post-commit events; confirm no orphaned modules
    - _Requirements: 3.1, 8.1; design "Request and Data Flow"_

  - [ ]* 22.4 Write smoke tests for one-time configuration
    - Encryption-at-rest enabled, audit retention ≥7 years policy, 30-min session expiry, MFA-required-for-Admin
    - _Requirements: Security & Data Retention constraints; 8.7_

- [ ] 23. Final checkpoint - full Phase 1 integration
  - Ensure all tests pass (property, unit, integration, smoke), ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional test sub-tasks and can be skipped for a faster MVP; core implementation tasks are never optional.
- Each task references specific requirement acceptance criteria and/or design Property numbers for traceability.
- Every property test uses fast-check at ≥100 iterations and is tagged `// Feature: hasoublabs-recruitment-platform, Property {number}: {property_text}`.
- Property-based tests validate universal properties; unit tests cover specific messages/transitions; integration tests cover email/DNS/scanner/object-storage/DB constraints; smoke tests cover one-time configuration.
- Checkpoints ensure incremental validation at logical phase boundaries.
- Phase 2 (Requirements 9–21) is out of scope; the event bus and service interfaces are the seams for later extension.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "2.2", "2.3", "2.4", "2.5", "2.6"] },
    { "id": 1, "tasks": ["4.1", "4.2", "6.1", "10.1", "14.1", "16.1"] },
    { "id": 2, "tasks": ["4.3", "4.4", "4.5", "4.6", "4.7", "6.2", "6.5", "6.6", "10.2", "13.1", "14.2", "14.3", "16.2"] },
    { "id": 3, "tasks": ["6.3", "7.1", "9.1", "13.2"] },
    { "id": 4, "tasks": ["6.4", "7.2", "7.3", "7.4", "7.5", "7.6", "9.2", "10.3", "14.4", "16.3"] },
    { "id": 5, "tasks": ["10.4", "14.5", "14.6", "14.7", "16.4", "16.5", "16.6", "18.1"] },
    { "id": 6, "tasks": ["10.5", "10.6", "10.9", "16.7", "16.8", "18.2", "18.3"] },
    { "id": 7, "tasks": ["10.7", "10.8", "11.1", "18.4", "19.1"] },
    { "id": 8, "tasks": ["11.2", "11.3", "11.4", "11.5", "11.6", "11.7", "19.2", "19.3", "19.4", "19.5", "19.6"] },
    { "id": 9, "tasks": ["19.7", "19.8", "21.1", "22.1", "22.2"] },
    { "id": 10, "tasks": ["22.3"] },
    { "id": 11, "tasks": ["22.4"] }
  ]
}
```

# PRD — Phase 1: Core Platform Foundation

## Overview

Phase 1 establishes the operational backbone of the HasoubLabs Recruitment Platform. By the end of this phase the team can register users, manage candidate profiles and CVs, post job descriptions, accept candidate applications, and maintain a tamper-evident audit trail — all without any AI or external integrations. This is the minimum viable platform: every subsequent phase depends on it.

**Covers requirements:** 1, 2, 3, 4, 5, 6, 14, 20

---

## Goals

- Replace spreadsheets and manual coordination for candidate intake and job posting.
- Give candidates self-service access to register, build their profile, upload their CV, and apply to jobs.
- Give admins full visibility and control over users, roles, and data.
- Establish data integrity guarantees and an audit trail from day one, so Phase 2 and Phase 3 can build on clean, auditable data.

## Non-Goals (deferred to later phases)

- AI-powered CV analysis, scoring, or tailoring (Phase 2)
- Candidate reviews and notes (Phase 2)
- Recruiter handoff packages (Phase 2)
- In-app chat (Phase 3)
- Email / WhatsApp integrations (Phase 3)
- Notifications beyond basic email confirmation on application (covered here only as a confirmation send)

---

## User Roles in Scope

| Role | Description |
|---|---|
| Admin | HasoubLabs employee. Full platform access. |
| Candidate | Arab student / graduate seeking employment. |
| Senior | Senior professional who posts job openings. |
| Recruiter | External recruiter. **No platform login.** Out of scope for Phase 1. |

---

## Requirements

---

### Requirement 1 — User Registration and Role Assignment

**User Story:** As an Admin, I want to register users and assign them appropriate roles, so that each person interacts with the platform according to their responsibilities and access level.

#### Acceptance Criteria

1. THE Platform SHALL support exactly three internal user roles: Admin, Candidate, and Senior.
2. WHEN a new user submits a registration request with all required fields provided and a valid email format, THE Platform SHALL assign a default role of Candidate until an Admin explicitly changes it.
3. WHEN an Admin assigns a role to a user, THE Platform SHALL record the role change with a timestamp and the identity of the Admin who made the change.
4. THE Platform SHALL prevent a user from holding more than one role simultaneously.
5. WHEN a user's role is changed, THE Platform SHALL revoke all permissions associated with the previous role and grant all permissions of the new role within the same request.
6. IF the permission update initiated by a role change fails to complete, THEN THE Platform SHALL revert the user to their previous role and return an error message indicating the role change could not be applied.
7. WHILE a user is not authenticated as an Admin, THE Platform SHALL deny access to the role management interface and return an error message indicating insufficient permissions.
8. THE Platform SHALL expose a role management interface accessible only to Admin-authenticated sessions.
9. IF a registration attempt is made with an email address already registered, THEN THE Platform SHALL reject the registration and return an error message identifying the email conflict.

---

### Requirement 2 — Geographic Verification

**User Story:** As an Admin, I want the platform to verify that all registering users are Israeli residents, so that we maintain the platform's intended geographic scope and comply with our operational requirements.

#### Acceptance Criteria

1. WHEN a user submits a registration form, THE Platform SHALL initiate a Geographic_Verification check before activating the account.
2. WHEN a user submits a registration form, THE Platform SHALL require the user to provide at least one of the following proof types of Israeli residency: Israeli phone number, Israeli national ID number, or Israeli address confirmation; and SHALL reject submission if none of the accepted proof types is provided.
3. IF Geographic_Verification fails for a registering user, THEN THE Platform SHALL reject account activation and notify the user with an error message indicating which residency proof requirement was not met and what accepted proof types are available.
4. WHILE a user's Geographic_Verification is pending, THE Platform SHALL restrict that user's access to all platform features except the verification submission interface; IF Geographic_Verification remains in a pending state for more than 72 hours after registration submission, THEN THE Platform SHALL automatically transition the verification status to failed and apply the rejection behavior defined in criterion 3.
5. WHEN an Admin submits a manual Geographic_Verification override for a specific user, THE Platform SHALL approve that user's Geographic_Verification, record the Admin's identity, a timestamp, and a non-empty reason of at least 10 characters.
6. WHEN the outcome of a Geographic_Verification check is determined (pass, fail, or manual override), THE Platform SHALL record the outcome and a timestamp against that user's record.
7. THE Platform SHALL prevent access to any platform feature, except the verification submission interface, for any user whose account does not have an associated Geographic_Verification record in a passing or manual-override state.

---

### Requirement 3 — Role-Based Access Control

**User Story:** As an Admin, I want each user role to have a clearly defined and enforced set of permissions, so that sensitive candidate data and platform operations are protected from unauthorized access.

#### Acceptance Criteria

1. THE Platform SHALL enforce a permission model where Admin, Candidate, and Senior roles have fixed, non-overlapping capabilities with no role-level overrides permitted.
2. THE Platform SHALL grant Admins access to all platform features, including user management, candidate profiles, all reviews, all notes, all applications, and system configuration (comprising platform settings, role assignments, and audit logs).
3. THE Platform SHALL grant Candidates access only to their own profile, their own CVs, their own Applications, AI-powered analysis and generation tools scoped exclusively to their own profile and CV data, and the in-app Chat.
4. THE Platform SHALL grant Seniors access to Job_Description posting, the Candidate list displaying only name, application status, and applied role per candidate (read-only), in-app Chat, and referral link generation.
5. WHEN a user attempts to access a resource outside their role's permissions, THE Platform SHALL deny the request and return an authorization error message indicating the action is not permitted.
6. IF a user attempts to access a resource outside their role's permissions, THEN THE Platform SHALL NOT return any data from the requested resource in the response.
7. THE Platform SHALL prevent Candidates from viewing other Candidates' profiles, CVs, reviews, or notes.
8. THE Platform SHALL prevent Seniors from viewing individual Candidate review timelines or evaluation notes of any level of detail.

---

### Requirement 4 — Candidate Profile Management

**User Story:** As a Candidate, I want to create and maintain a comprehensive profile, so that recruiters and admins can accurately evaluate my background and match me to suitable opportunities.

#### Acceptance Criteria

1. THE Platform SHALL allow a Candidate to create a profile containing: full name, email address, phone number, educational background (institution, degree, and graduation year), work experience (employer, role title, and duration), technical skills (list of up to 20 skills, each up to 50 characters), languages spoken (list of up to 10 languages), and a summary statement of up to 1000 characters.
2. WHILE a Candidate's account is active, THE Platform SHALL allow the Candidate to update any field of their own profile at any time.
3. WHEN a Candidate updates their profile, THE Platform SHALL record the update with a UTC timestamp accurate to the second.
4. THE Platform SHALL allow Admins and Seniors to view a Candidate's profile summary, where profile summary includes: full name, email address, phone number, technical skills, languages spoken, and summary statement.
5. THE Platform SHALL allow only Admins to view the complete Candidate profile, where the complete profile includes all fields defined in criterion 1, the full update history, assessment metrics, and internal notes.
6. THE Platform SHALL require that each Candidate profile contains at minimum: full name (1–100 characters), email address (valid format per RFC 5321), phone number (valid E.164 format), and at least one area of technical interest selected from the platform's defined skill set, before the profile is considered complete.
7. IF a Candidate attempts to submit an Application while their profile is incomplete, THEN THE Platform SHALL reject the submission and return an error message identifying each specific missing or invalid field by name.
8. WHEN a Candidate saves profile changes that do not satisfy the completeness criteria defined in criterion 6, THE Platform SHALL save the profile in a draft state and indicate to the Candidate which fields are missing or invalid.

---

### Requirement 5 — CV Upload and Version Control

**User Story:** As a Candidate, I want to upload new versions of my CV and have all prior versions retained, so that my evolution over time is tracked and historical context is never lost.

#### Acceptance Criteria

1. THE Platform SHALL allow a Candidate to upload a CV file only if it is in PDF format and does not exceed 10 MB in size.
2. WHEN a Candidate uploads a new CV, THE Platform SHALL store it as a new CV_Version and SHALL NOT delete or overwrite any prior CV_Version.
3. THE Platform SHALL associate each CV_Version with the Candidate's identity, a sequential version number starting at 1 and incrementing by 1 per upload, and an upload timestamp recorded in UTC.
4. THE Platform SHALL allow Admins to view the list of all CV_Versions for any Candidate, including each version's sequential number, upload timestamp, and file size.
5. THE Platform SHALL allow a Candidate to view their own CV_Version history, including each version's sequential number and upload timestamp, and download any prior version as the original PDF file.
6. THE Platform SHALL treat the most recently uploaded CV_Version as the active CV for AI analysis and scoring purposes, unless an Admin explicitly designates a different version as active.
7. IF an Admin designates a specific CV_Version as active, THEN THE Platform SHALL use that CV_Version for all subsequent AI analysis and scoring until a newer version is uploaded or a different version is designated.
8. IF a Candidate uploads a file that is not in PDF format or exceeds 10 MB, THEN THE Platform SHALL reject the upload without storing any data, and return an error message identifying whether the violation was file format, file size, or both.
9. FOR ALL CV_Versions stored, THE Platform SHALL guarantee that the stored file is byte-for-byte identical to the file uploaded by the Candidate, verified using a checksum computed at upload time and validated upon each retrieval.

---

### Requirement 6 — Job Description Posting

**User Story:** As a Senior, I want to post job descriptions on the platform, so that candidates can discover and apply to relevant opportunities.

#### Acceptance Criteria

1. THE Platform SHALL allow a Senior or Admin to create a Job_Description containing: a role title (maximum 150 characters), company name (maximum 150 characters), location (maximum 200 characters), employment type (one of: Full-time, Part-time, Contract, Freelance, or Internship), required skills (between 1 and 20 skills), experience level (one of: Junior, Mid, Senior, or Lead), and a description field (maximum 5000 characters).
2. WHEN a Job_Description is created, THE Platform SHALL assign it a unique identifier and record the creator's identity and a creation timestamp.
3. IF a user who is neither the creator of a Job_Description nor an Admin attempts to edit or close that Job_Description, THEN THE Platform SHALL reject the action and display an error indicating the user is not authorized to modify this posting.
4. WHEN a Job_Description is closed, THE Platform SHALL mark it with a closed status indicator visible in all listing and detail views.
5. WHEN a Job_Description is closed, THE Platform SHALL prevent any new Application from being submitted for it.
6. WHEN a Candidate views the Job_Descriptions list, THE Platform SHALL display all active Job_Descriptions in a paginated list (maximum 20 items per page) searchable by role title, company name, and required skills.
7. WHEN a Candidate applies one or more filters (skills, location, employment type, experience level), THE Platform SHALL display only the Job_Descriptions matching all selected filter criteria.
8. WHEN a Senior requests a shareable referral link for a Job_Description they created, THE Platform SHALL generate a unique link that expires 30 days after creation.
9. WHEN an unregistered Candidate completes registration using a valid referral link, THE Platform SHALL pre-associate that Candidate with the Senior who generated the link. IF the referral link has expired or the Candidate is already registered, THEN THE Platform SHALL complete the registration without applying the referral association and SHALL display a message indicating the referral could not be applied.

---

### Requirement 14 — Self-Service Candidate Application

**User Story:** As a Candidate, I want to independently apply for job openings on the platform, so that I can take ownership of my job search without requiring manual intervention from platform staff.

#### Acceptance Criteria

1. IF a Candidate's profile has all mandatory fields completed (full name, contact email, and at least one active CV_Version), THEN THE Platform SHALL allow that Candidate to submit an Application to any active Job_Description.
2. WHEN a Candidate submits an Application, THE Platform SHALL record: the Candidate's identity, the Job_Description identifier, the active CV_Version at time of submission, and a submission timestamp.
3. IF a Candidate attempts to submit an Application to a Job_Description for which they already have a recorded Application, THEN THE Platform SHALL reject the submission and inform the Candidate that they have already applied to that role.
4. THE Platform SHALL display to the Candidate a list of all their Applications, each showing one of the following statuses: "Submitted", "Under Review", "Forwarded to Recruiter", or "Closed".
5. THE Platform SHALL allow Admins to update the status of any Application to any of the defined statuses and SHALL record each status change with a timestamp and the Admin's identity.
6. WHEN a Job_Description is closed, THE Platform SHALL automatically transition all Applications for that Job_Description whose current status is "Submitted" or "Under Review" to "Closed" status.
7. WHEN a Candidate's Application is successfully submitted, THE Platform SHALL send a confirmation notification to the Candidate's registered email address within 5 minutes of submission.
8. FOR ALL submitted Applications, the CV_Version recorded at submission time SHALL remain immutable even if the Candidate uploads a newer CV_Version afterward.
9. IF a Candidate attempts to submit an Application and no active CV_Version exists on their profile, THEN THE Platform SHALL reject the submission and inform the Candidate that a CV must be uploaded before applying.

---

### Requirement 20 — Audit Trail and Data Integrity

**User Story:** As an Admin, I want a complete, tamper-evident audit trail of all significant platform actions, so that I can review decisions, resolve disputes, and demonstrate accountability for candidate data handling.

#### Acceptance Criteria

1. THE Platform SHALL record an audit log entry for every action that creates, modifies, or deletes a platform entity, including: user registration, role changes, CV uploads, profile updates, Application submissions, Review submissions, Note edits, Recruiter_Package sends, and email/WhatsApp communications.
2. EACH audit log entry SHALL contain: the identity of the actor, the action performed, the identifier of the affected entity, the previous field-level values of all modified fields (for modification actions), and a UTC timestamp with millisecond precision.
3. IF any user, including an Admin, attempts to modify or delete an audit log entry, THEN THE Platform SHALL reject the operation, return an error indicating the action is not permitted, and record the attempt as a new audit log entry identifying the actor and the targeted log entry identifier.
4. THE Platform SHALL allow Admins to search and filter the audit log by actor identity, action type, entity type, entity identifier, and date range.
5. IF a system operation fails partway through a multi-step transaction, THEN THE Platform SHALL roll back all partial changes so that every affected entity returns to its state prior to the operation, and SHALL record the failure as an audit log entry without persisting any intermediate entity state.
6. FOR every entity tracked in the audit log, applying the recorded field-level changes from all audit log entries for that entity in ascending timestamp order SHALL reproduce the current state of that entity.

---

## Cross-Cutting Constraints (apply to all phases)

### Security
- THE Platform SHALL enforce HTTPS for all communications between clients and the platform.
- THE Platform SHALL store passwords using a salted cryptographic hashing algorithm with a minimum of bcrypt cost factor 12 or equivalent.
- THE Platform SHALL enforce session expiration after 30 minutes of inactivity.

### Performance
- WHILE the platform is under normal operating load (up to 500 concurrent users), THE Platform SHALL respond to any user-initiated read or write request within 3 seconds.

### Availability
- THE Platform SHALL maintain availability of at least 99.5% measured monthly, excluding scheduled maintenance windows communicated at least 24 hours in advance.

### Localization
- THE Platform SHALL support Arabic and English in the user interface, with the language preference set per user account.
- WHEN content is submitted in Arabic, THE Platform SHALL store and display it in Arabic without transliteration or modification.

### Data Retention
- Candidate data (profiles, CVs, reviews, notes, messages) SHALL be retained for a minimum of 5 years after the Candidate's last activity, unless the Candidate explicitly requests deletion and applicable law permits it.

---

## Correctness Properties for Testing

### CV Version Control (Requirement 5)
- **Monotonic version numbers**: `∀ v₁, v₂ ∈ CV_Versions(candidate): upload_time(v₁) < upload_time(v₂) → version_number(v₁) < version_number(v₂)`
- **Upload integrity round-trip**: `retrieve(store(file)) == file`
- **Version count invariant**: After N successful uploads, the number of stored CV_Versions for that Candidate SHALL equal N.

### Application Deduplication (Requirement 14)
- **Idempotent application**: `submit(c, jd); submit(c, jd) → count(applications(c, jd)) == 1`

### Audit Log (Requirement 20)
- **Completeness**: For every entity modification, the audit log SHALL contain at least one entry referencing that entity and that modification type.
- **Immutability**: Reading the audit log at time T₂ > T₁ SHALL return all entries present at T₁, plus any new entries. No entry present at T₁ SHALL be absent at T₂.

---

## Dependencies and Exit Criteria

Phase 1 is considered complete when:
- All three user roles can register, be verified, and log in.
- Candidates can build a profile, upload CVs, and apply to job postings.
- Seniors can post and manage job descriptions.
- Admins can manage users, roles, application statuses, and view the full audit log.
- All acceptance criteria above pass in a staging environment.

Phase 2 cannot begin until Phase 1 exit criteria are met, because the AI engine and review system depend on clean candidate profiles, CVs, and job descriptions being in place.

# PRD — Phase 2: Intelligence & Engagement

## Overview

Phase 2 transforms the platform from a structured data store into an intelligent recruitment engine. It adds the AI subsystem, structured candidate evaluation, contextual notes, the recruiter handoff workflow, and the notification infrastructure. By the end of this phase, admins can score and rank candidates automatically, make informed forwarding decisions backed by reviews and notes, and send complete recruiter packages — all with timely notifications keeping every actor in the loop.

**Covers requirements:** 7, 8, 9, 10, 11, 12, 13, 15, 19

**Depends on:** Phase 1 (all requirements)

---

## Goals

- Eliminate manual candidate-to-job matching by scoring every candidate against every open JD automatically.
- Give candidates actionable, AI-generated feedback to improve their CV and LinkedIn profile.
- Give admins and seniors a structured, timestamped evaluation record for every candidate.
- Enable the recruiter handoff to be assembled and sent in one workflow, with full context attached.
- Keep all actors informed through a reliable, configurable notification system.

## Non-Goals (deferred to Phase 3)

- In-app chat between candidates and seniors (Phase 3)
- Email inbox synchronisation (Phase 3)
- WhatsApp messaging integration (Phase 3)

---

## User Roles in Scope

| Role | New capabilities added in Phase 2 |
|---|---|
| Admin | View/edit JD extractions, view candidate scores and ranking, submit and view all reviews, view all notes, compose and send recruiter packages, configure notification channels |
| Candidate | Request CV improvement analysis, run JD-specific CV tailoring, use LinkedIn enhancement feature, receive notifications |
| Senior | Submit reviews (own only visible), add and edit candidate notes, receive notifications |
| Recruiter | Receives recruiter packages by email (no platform login) |

---

## Requirements

---

### Requirement 7 — Automated JD Extraction

**User Story:** As an Admin, I want the platform to automatically extract structured requirements from job descriptions, so that candidate scoring and matching can be performed consistently without manual tagging.

#### Acceptance Criteria

1. WHEN a Job_Description is created or updated, THE AI_Engine SHALL initiate JD_Extraction on the description field within 10 seconds of the create or update event being persisted.
2. WHEN JD_Extraction is initiated, THE AI_Engine SHALL produce a JD_Extraction containing: a non-empty list of required technical skills, a list of preferred skills (which may be empty), a minimum experience level expressed as a whole number of years between 0 and 50 inclusive, and a non-empty set of domain keywords.
3. WHEN JD_Extraction is complete, THE Platform SHALL store the result linked to the specific Job_Description and record the extraction timestamp.
4. THE Platform SHALL allow Admins to view, edit, and override any content field in a JD_Extraction, where content fields are: required technical skills, preferred skills, minimum experience level, and domain keywords. System-managed fields (extraction timestamp, linked Job_Description identifier, and audit records) SHALL NOT be editable by Admins.
5. IF JD_Extraction does not complete within 60 seconds of initiation, or terminates with an error, THEN THE Platform SHALL display an in-platform notification to the Admin who created or last updated the Job_Description, log the failure with the Job_Description identifier and the reason for failure, and present a form allowing manual entry of all JD_Extraction content fields.
6. WHEN an Admin edits a JD_Extraction, THE Platform SHALL record the change with the Admin's identity and a timestamp, preserving the prior extraction result.
7. THE AI_Engine SHALL ensure that each term in the required technical skills and domain keywords fields of a JD_Extraction either appears verbatim in the associated Job_Description's description field, or is a recognised abbreviation or industry-standard synonym of a term that appears verbatim in that description field.

---

### Requirement 8 — Candidate Scoring and Ranking

**User Story:** As an Admin, I want each candidate to receive an automatic match score for each job description, so that I can quickly identify the most suitable candidates for a role without manually reviewing every profile.

#### Acceptance Criteria

1. WHEN a JD_Extraction is created or updated for a Job_Description, THE AI_Engine SHALL compute a Candidate_Score for every Candidate whose profile is complete, where a complete profile is defined as having at least one skill recorded and at least one CV_Version uploaded.
2. WHEN a Candidate updates their profile or uploads a new CV_Version, THE AI_Engine SHALL recompute that Candidate's Candidate_Score for all Job_Descriptions whose status is not closed or archived.
3. THE Candidate_Score SHALL be a numeric value on a scale of 0 to 100, computed from the degree of match between the Candidate's skills, years of experience, and education level recorded in their CV_Version and the corresponding fields extracted in the JD_Extraction.
4. WHEN an Admin views the candidate list for a specific Job_Description, THE Platform SHALL display Candidates ranked by Candidate_Score in descending order.
5. WHEN an Admin views a Candidate_Score, THE Platform SHALL expose the contributing factors comprising: matched skills (skills present in both the Candidate profile and the JD_Extraction), missing skills (skills required by the JD_Extraction absent from the Candidate profile), and experience gap (the difference in whole years between the years of experience required by the JD_Extraction and the years of experience recorded in the Candidate's CV_Version).
6. THE Platform SHALL NOT expose raw Candidate_Scores or rankings to Candidates.
7. IF a Candidate's profile is incomplete (does not have at least one skill recorded and at least one CV_Version uploaded), THEN THE AI_Engine SHALL NOT compute a Candidate_Score for that Candidate and THE Platform SHALL display an "incomplete profile" indicator for that Candidate in the ranking view.
8. THE AI_Engine SHALL assign a Candidate_Score greater than or equal to the score of any other Candidate scored against the same Job_Description whose skill set is a strict subset of the first Candidate's skills recorded in their CV_Version.
9. IF THE AI_Engine fails to compute a Candidate_Score for one or more Candidates during a scoring run, THEN THE Platform SHALL retain the most recently computed Candidate_Score for each affected Candidate, mark each such score as stale, and display an error indication to Admins identifying which Candidates could not be rescored.

---

### Requirement 9 — AI-Powered CV Improvement

**User Story:** As a Candidate, I want AI-generated suggestions to improve my CV's structure and content, so that I can present myself more effectively to recruiters and increase my chances of being selected.

#### Acceptance Criteria

1. THE Platform SHALL provide a CV improvement feature accessible to Candidates from their profile dashboard.
2. WHEN a Candidate requests CV analysis, THE AI_Engine SHALL analyze the active CV_Version and generate between 1 and 20 Profile_Suggestions, each targeting at least one of the following dimensions: structure, clarity, completeness, or impact of content.
3. THE AI_Engine SHALL return all Profile_Suggestions within 30 seconds of the Candidate's request when the platform is operating under normal load conditions.
4. EACH Profile_Suggestion SHALL identify the CV section it addresses by name and include a recommended action describing what to change or add, containing no fewer than 10 and no more than 300 characters.
5. THE Platform SHALL allow the Candidate to mark each Profile_Suggestion with exactly one of three statuses: accepted, dismissed, or saved for later.
6. THE Platform SHALL retain all Profile_Suggestions and their Candidate-assigned statuses for the duration of the Candidate's account.
7. IF the Candidate has no uploaded CV_Version, THEN THE Platform SHALL display a prompt directing the Candidate to upload a CV before the CV improvement feature becomes accessible.
8. THE Platform SHALL NOT modify the Candidate's stored CV or CV_Version in any way without the Candidate's explicit action.
9. IF the AI_Engine fails to return Profile_Suggestions within 30 seconds or encounters an error during analysis, THEN THE Platform SHALL display an error message indicating that the analysis could not be completed and allow the Candidate to retry without data loss.
10. IF the CV analysis request is submitted and Profile_Suggestions have already been generated for the current active CV_Version, THEN THE Platform SHALL display the existing Profile_Suggestions and indicate the date and time they were generated, without triggering a new analysis.

---

### Requirement 10 — Job-Specific CV Tailoring

**User Story:** As a Candidate, I want to receive targeted suggestions for adapting my CV to a specific job description, so that I can maximise my match score and relevance for roles I care about.

#### Acceptance Criteria

1. WHEN a Candidate selects a Job_Description with a completed JD_Extraction and requests tailoring, THE Platform SHALL initiate a tailoring session using the Candidate's most recently saved CV_Version.
2. WHEN a Candidate requests tailoring for a Job_Description, THE AI_Engine SHALL compare the active CV_Version against the JD_Extraction and generate targeted Profile_Suggestions covering identified skill gaps and terminology mismatches between the CV_Version and the JD_Extraction.
3. WHEN a tailoring session is initiated, THE AI_Engine SHALL return job-specific Profile_Suggestions within 30 seconds.
4. EACH tailoring Profile_Suggestion SHALL reference the specific JD requirement it addresses and include a rationale of no fewer than one sentence explaining why the change improves alignment with that JD requirement.
5. THE Platform SHALL allow the Candidate to run tailoring for multiple Job_Descriptions independently, storing each session's Profile_Suggestions separately and associated with their respective Job_Description, without overwriting results from other Job_Descriptions.
6. IF a Job_Description has no completed JD_Extraction, THEN THE Platform SHALL indicate this state to the Candidate and SHALL NOT offer the tailoring feature for that Job_Description until extraction is complete.
7. IF the AI_Engine does not return Profile_Suggestions within 30 seconds, THEN THE Platform SHALL terminate the tailoring session and display an error message indicating the request could not be completed, without modifying any previously stored Profile_Suggestions.
8. FOR ALL tailoring sessions submitted with the same CV_Version and the same JD_Extraction, THE AI_Engine SHALL return Profile_Suggestions that address the same set of JD requirements as identified in a prior session on those identical inputs.

---

### Requirement 11 — LinkedIn Profile Enhancement

**User Story:** As a Candidate, I want AI-generated suggestions to improve my LinkedIn profile, so that I am more visible to external recruiters and industry professionals.

#### Acceptance Criteria

1. THE Platform SHALL provide a LinkedIn enhancement feature accessible to Candidates from their profile dashboard.
2. WHEN a Candidate accesses the LinkedIn enhancement feature, THE Platform SHALL prompt the Candidate to either paste their LinkedIn profile text (up to 15,000 characters) or connect their LinkedIn account via OAuth; IF the OAuth connection attempt fails, THEN THE Platform SHALL display an error message indicating the connection could not be established and allow the Candidate to retry or switch to the paste input method.
3. WHEN LinkedIn profile content is submitted, THE AI_Engine SHALL analyze the content and produce Profile_Suggestions covering: headline, summary, experience descriptions, skills section, and profile completeness assessed as the presence or absence of each of those five sections.
4. EACH LinkedIn Profile_Suggestion SHALL specify the target section, a description of the identified gap or misalignment in the current content, and a recommended replacement or addition.
5. THE Platform SHALL allow the Candidate to mark each LinkedIn Profile_Suggestion as accepted, dismissed, or saved for later, consistent with the CV improvement workflow.
6. THE Platform SHALL NOT store the Candidate's raw LinkedIn profile content after the session in which it was submitted ends — defined as the point at which the Candidate navigates away from the enhancement feature or closes the session — unless the Candidate explicitly consents to storage before submission.
7. IF the Candidate does not provide LinkedIn content, THEN THE Platform SHALL NOT generate LinkedIn Profile_Suggestions for that session.
8. IF the AI_Engine fails to analyze the submitted LinkedIn profile content, THEN THE Platform SHALL display an error message indicating the analysis could not be completed and SHALL NOT store any partial Profile_Suggestions from that session.

---

### Requirement 12 — Candidate Review System

**User Story:** As an Admin or Senior, I want to submit structured evaluations of candidates at any point during the recruitment process, so that a complete and timestamped history of impressions is preserved for every candidate.

#### Acceptance Criteria

1. THE Platform SHALL allow Admins and Seniors to submit a Review for any Candidate at any stage of the recruitment pipeline.
2. A Review SHALL contain: the reviewer's identity, a UTC timestamp, a structured rating expressed as an integer from 1 to 5 across each of the following dimensions: technical ability, communication, culture fit, and overall impression; a free-text assessment of up to 2000 characters; and an optional association with a specific Job_Description.
3. WHEN a Review is submitted, THE Platform SHALL append it to the Candidate's Review_Timeline in chronological order.
4. IF a Review submission is missing any required field (rating dimensions, free-text assessment, or reviewer identity), THEN THE Platform SHALL reject the submission, return an error identifying each missing field, and not modify the Candidate's Review_Timeline.
5. THE Platform SHALL NOT allow a submitted Review to be edited or deleted; corrections SHALL be submitted as a new Review whose free-text assessment includes an explicit reference to the identifier of the Review being corrected.
6. THE Platform SHALL display the Review_Timeline for a Candidate in chronological order to Admins, including all Reviews regardless of which reviewer submitted them.
7. THE Platform SHALL allow Admins to filter a Candidate's Review_Timeline by reviewer identity, date range, and associated Job_Description.
8. THE Platform SHALL prevent Candidates from viewing their own Review_Timeline.
9. FOR ALL Reviews in a Candidate's Review_Timeline, the sequence of timestamps SHALL be strictly non-decreasing (append-only guarantee).
10. WHEN a Senior submits a Review, THE Platform SHALL record the Review and SHALL allow the Senior to view only their own submitted Reviews for that Candidate, but SHALL NOT display Reviews submitted by any other reviewer.

---

### Requirement 13 — Job-Specific Candidate Notes

**User Story:** As an Admin or Senior, I want to add notes specific to a candidate's application for a particular role, so that contextual observations are captured and available during the recruiter handoff.

#### Acceptance Criteria

1. THE Platform SHALL allow Admins and Seniors to create a Candidate_Note associated with a specific Candidate and a specific Job_Description.
2. A Candidate_Note SHALL contain: the author's identity, a creation timestamp, a last-modified timestamp, and free-text content of up to 2000 characters.
3. THE Platform SHALL allow the author of a Candidate_Note to edit its content, updating the last-modified timestamp on each edit.
4. IF a Candidate_Note edit submission results in empty or blank content, THEN THE Platform SHALL reject the edit and return an error indicating that note content cannot be empty.
5. THE Platform SHALL retain the full edit history of each Candidate_Note, where the edit history consists of a snapshot of the content before each edit together with the timestamp of that edit.
6. THE Platform SHALL allow Admins to view all Candidate_Notes for a Candidate across all Job_Descriptions.
7. THE Platform SHALL allow Seniors to view only Candidate_Notes they authored.
8. THE Platform SHALL prevent Candidates from viewing any Candidate_Notes associated with their profile.
9. IF a Job_Description is closed, THEN THE Platform SHALL retain all associated Candidate_Notes in read-only state, preventing edits by any user including the original author, and SHALL allow Admins and the authoring Senior to continue viewing them.

---

### Requirement 15 — Recruiter Handoff Workflow

**User Story:** As an Admin, I want to compile a candidate's complete profile, reviews, and notes into a Recruiter_Package and send it to an external recruiter, so that the recruiter receives everything needed to evaluate the candidate without additional back-and-forth.

#### Acceptance Criteria

1. THE Platform SHALL allow an Admin to initiate a Recruiter_Package for a specific Candidate and a specific Job_Description.
2. THE Platform SHALL automatically populate the Recruiter_Package with: the CV_Version recorded on the Candidate's Application for the selected Job_Description if an Application exists, otherwise the Candidate's currently active CV_Version, the Candidate's profile, the full Review_Timeline for that Candidate, all Candidate_Notes for that Candidate linked to the selected Job_Description, and the Candidate_Score for the Job_Description.
3. THE Platform SHALL allow the Admin to edit the contents of the Recruiter_Package before sending, including removing or annotating individual Reviews or Notes, without modifying the auto-populated version recorded at initiation.
4. WHEN an Admin edits the Recruiter_Package, THE Platform SHALL record each edit with a timestamp and the Admin's identity, and SHALL preserve the original auto-populated version as a separate read-only snapshot.
5. THE Platform SHALL allow the Admin to preview the Recruiter_Package before sending, rendering it in the same format that will be delivered to the Recruiter.
6. WHEN the Admin confirms sending, THE Platform SHALL deliver the Recruiter_Package to the Recruiter's email address associated with the Job_Description and record the delivery with a timestamp and the Admin's identity.
7. IF delivery of the Recruiter_Package to the Recruiter's email address fails, THEN THE Platform SHALL notify the Admin with an error message indicating the delivery failure, and the Recruiter_Package SHALL remain in an unsent state.
8. THE Platform SHALL retain a read-only archived copy of each sent Recruiter_Package indefinitely.
9. IF the Recruiter's email address is not set for a Job_Description, THEN THE Platform SHALL require the Admin to enter an email address in a valid format (local-part@domain.tld) before allowing the send action.
10. WHEN an Admin triggers a resend of a previously sent Recruiter_Package, THE Platform SHALL deliver the archived copy of that Recruiter_Package to the Recruiter's current email address and record the resend as a separate delivery event with a timestamp and the Admin's identity.

---

### Requirement 19 — Notifications and Alerts

**User Story:** As a user, I want to receive timely in-platform and channel-specific notifications about actions relevant to me, so that I stay informed about important updates without polling the platform manually.

#### Acceptance Criteria

1. WHEN a Candidate's Application status changes, a new Job_Description matching the Candidate's top skills is posted, a Chat_Message is received by the Candidate, or a CV improvement analysis is complete, THE Platform SHALL write an in-platform notification to that Candidate's notification queue within 10 seconds of the triggering event.
2. WHEN a new Candidate registers, a Candidate submits an Application, a JD_Extraction fails, or an inbound email is logged as unmatched, THE Platform SHALL write an in-platform notification to the Admin's notification queue within 10 seconds of the triggering event.
3. WHEN a Candidate applies to a Job_Description posted by a Senior, or a Chat_Message is received by a Senior, THE Platform SHALL write an in-platform notification to that Senior's notification queue within 10 seconds of the triggering event.
4. WHEN a notification is written to a recipient's notification queue, THE Platform SHALL record: the recipient's identity, the notification type, the identifier of the associated entity, and a UTC timestamp with millisecond precision.
5. THE Platform SHALL allow users to mark individual notifications as read and to permanently delete all notifications in their notification list in a single operation.
6. WHEN a notification is marked as read, THE Platform SHALL update its state to read within 2 seconds and SHALL NOT revert it to unread unless a new triggering event of the same type for the same entity occurs.
7. THE Platform SHALL allow Admins to configure which notification types are delivered via in-platform UI, email, or WhatsApp for each user role.
8. IF no channel configuration has been set for a user role, THEN THE Platform SHALL deliver notifications for that role exclusively via the in-platform UI.

> **Note:** Chat_Message notification triggers (criteria 1 and 3) are defined here but will only fire once Phase 3 (in-app chat) is delivered. The notification infrastructure itself must be in place from Phase 2 launch.

---

## Cross-Cutting Constraints

All constraints defined in Phase 1 apply. Additional constraint for Phase 2:

### AI Engine Performance
- THE AI_Engine SHALL complete CV analysis, tailoring, and JD extraction tasks within 30 seconds under normal operating conditions.
- All AI features SHALL degrade gracefully: timeouts and errors must surface to the user with a retry option and must never cause data loss.

### Recruiter Package Retention
- Recruiter_Package archives SHALL be retained indefinitely.

---

## Correctness Properties for Testing

### Candidate Scoring (Requirement 8)
- **Monotone skill superset**: For any two Candidates A and B scored against the same JD, if `skills(A) ⊇ skills(B)` then `score(A, JD) ≥ score(B, JD)`.
- **Score range invariant**: For all computed Candidate_Scores, the value SHALL be in [0, 100].
- **Score determinism**: Running the scoring algorithm twice with the same Candidate profile and JD_Extraction SHALL produce the same score.

### Review Timeline (Requirement 12)
- **Append-only ordering**: `∀ r₁, r₂ ∈ timeline: index(r₁) < index(r₂) → timestamp(r₁) ≤ timestamp(r₂)`
- **Count invariant after append**: After submitting N Reviews for a Candidate, `len(review_timeline(candidate)) == N`.

### JD Extraction Round-Trip (Requirement 7)
- **Keyword containment**: For all keywords in a completed JD_Extraction, each keyword SHALL appear in or be semantically derivable from the source Job_Description text.

---

## Dependencies and Exit Criteria

Phase 2 **requires** Phase 1 to be complete and stable in production before launch.

Phase 2 is considered complete when:
- JD extraction runs automatically on new/updated job descriptions and results are visible to admins.
- Candidate scoring is computed and the ranked list is visible per JD.
- Candidates can request CV improvement, job-specific tailoring, and LinkedIn enhancement.
- Admins and Seniors can submit reviews; Admins see the full timeline, Seniors see only their own.
- Admins and Seniors can create and edit job-scoped notes.
- Admins can assemble, preview, edit, and send a Recruiter_Package by email.
- In-platform notifications fire within SLA for all triggering events covered in Requirement 19.
- All acceptance criteria above pass in a staging environment.

Phase 3 can begin independently of Phase 2 completion for the integration scaffolding, but the notification channel configuration (Req 19.7) depends on Phase 2 being live.

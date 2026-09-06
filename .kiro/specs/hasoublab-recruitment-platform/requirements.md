# Requirements Document

## Introduction

The HasoubLabs Recruitment Platform is a greenfield web application designed to help the Hasoub Labs recruitment team efficiently discover, understand, manage, match, and support Arab students and graduates throughout their journey toward employment in Israel's high-tech industry.

The platform serves four distinct actors: internal admins/employees, candidates (Arab students and graduates), senior professionals who post jobs and act as mentors, <todo: delet>and external recruiters who receive consolidated candidate packages<todo:>. It replaces or reduces manual coordination (spreadsheets, email threads, manual WhatsApp messages) with structured workflows, AI-assisted tooling, and persistent records that accumulate value over time.

Key outcomes the platform must achieve:
- Reduce recruiter manual effort per candidate while improving decision quality
- Give candidates self-service access to AI-powered profile improvement tools
- Give seniors a lightweight way to post openings and connect with suitable candidates
- Provide a full audit trail of every candidate's journey — CVs, reviews, notes, communications — so no context is ever lost

---

## Glossary

- **Admin**: A HasoubLabs employee with full platform access. Responsible for overseeing candidates, managing users, and configuring platform settings.
- **Candidate**: An Arab student or graduate who registers on the platform to seek employment in Israel's high-tech industry.
- **Senior / Job_Poster**: A senior professional or mentor registered on the platform who can post job openings and share referral links.
- **Recruiter**: An external company recruiter who receives consolidated candidate profiles via email. The Recruiter does not have a platform login.
- **CV**: A candidate's resume document (PDF or equivalent).
- **CV_Version**: A specific historical snapshot of a Candidate's CV, stored immutably after upload.
- **Job_Description (JD)**: A structured or unstructured posting describing a role's requirements, responsibilities, and qualifications.
- **JD_Extraction**: The result of automated parsing of a Job_Description, producing a structured set of required skills, experience levels, and keywords.
- **Candidate_Score**: A numeric rank assigned to a Candidate for a specific Job_Description, derived from keyword matching between the Candidate's profile/CV and the JD_Extraction.
- **Review**: A structured evaluation of a Candidate by an Admin or Senior, associated with a timestamp and optionally with a specific Job_Description.
- **Review_Timeline**: The ordered, append-only sequence of all Reviews for a given Candidate.
- **Candidate_Note**: A free-text annotation added by an Admin or Senior for a Candidate in the context of a specific Job_Description.
- **Recruiter_Package**: A consolidated document (or structured email payload) containing a Candidate's profile, CV, Review_Timeline, and Candidate_Notes for a specific role, sent to an external Recruiter.
- **Chat_Message**: A message exchanged between a Candidate and a Senior via in-app messaging.
- **Geographic_Verification**: The process of confirming that a user is an Israeli resident before granting platform access.
- **AI_Engine**: The platform's AI subsystem responsible for CV improvement, JD-specific tailoring, LinkedIn suggestions, JD extraction, and keyword scoring.
- **Profile_Suggestion**: An AI-generated recommendation targeting a specific section of a Candidate's CV or LinkedIn profile.
- **Application**: A formal expression of interest by a Candidate for a specific Job_Description posted on the platform.
- **Platform**: The HasoubLabs Recruitment Platform web application described in this document.
- **System**: The Platform and all its subsystems acting as a whole.

---

## Requirements

---

### Requirement 1: User Registration and Role Assignment

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

### Requirement 2: Geographic Verification

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

### Requirement 3: Role-Based Access Control

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

### Requirement 4: Candidate Profile Management

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

### Requirement 5: CV Upload and Version Control

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

### Requirement 6: Job Description Posting

**User Story:** As a Senior, I want to post job descriptions on the platform, so that candidates can discover and apply to relevant opportunities and the platform can match candidates automatically.

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

### Requirement 7: Automated JD Extraction

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

### Requirement 8: Candidate Scoring and Ranking

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

### Requirement 9: AI-Powered CV Improvement

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

### Requirement 10: Job-Specific CV Tailoring

**User Story:** As a Candidate, I want to receive targeted suggestions for adapting my CV to a specific job description, so that I can maximize my match score and relevance for roles I care about.

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

### Requirement 11: LinkedIn Profile Enhancement

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

### Requirement 12: Candidate Review System

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

### Requirement 13: Job-Specific Candidate Notes

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

### Requirement 14: Self-Service Candidate Application

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

### Requirement 15: Recruiter Handoff Workflow

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

### Requirement 16: In-App Chat Between Seniors and Candidates

**User Story:** As a Candidate, I want to message senior professionals on the platform, so that I can ask for career guidance, referrals, and advice from people with industry experience.

#### Acceptance Criteria

1. THE Platform SHALL provide an in-app Chat feature allowing direct messaging between a Candidate and a Senior.
2. WHEN a Candidate initiates a Chat with a Senior, THE Platform SHALL create a Chat session between those two users if one does not already exist, or open the existing session if one does.
3. WHEN a Senior opts in to receiving chat requests and a Candidate initiates contact, THE Platform SHALL notify the Senior of the new Chat session.
4. WHEN a Chat_Message is sent, THE Platform SHALL store: sender identity, recipient identity, message content (up to 2000 characters), and a UTC send timestamp.
5. WHEN the recipient is online, THE Platform SHALL deliver a sent Chat_Message to the recipient's session within 5 seconds of send. WHEN the recipient is offline, THE Platform SHALL queue the message for delivery upon the recipient's next login, retaining the queued message for up to 30 days before expiry.
6. THE Platform SHALL allow a Senior to optionally reveal their professional email address within a Chat_Message, which THE Platform SHALL display as a clickable link to the Candidate.
7. THE Platform SHALL allow Admins to view Chat_Message histories for compliance and moderation purposes.
8. THE Platform SHALL prevent Candidates from initiating Chat sessions with other Candidates.
9. IF a user's account is deactivated, THEN THE Platform SHALL preserve all prior Chat_Message history in read-only form and SHALL prevent new messages from being sent or received on that account.

---

### Requirement 17: Email Synchronization with Admin Mailbox

**User Story:** As an Admin, I want the platform to synchronize with the core admin email inbox, so that all candidate-related correspondence is logged centrally and no communication is lost.

#### Acceptance Criteria

1. THE Platform SHALL integrate with a designated admin email account via IMAP/SMTP or a supported email API to send and receive emails on behalf of the Platform.
2. WHEN an outbound email is triggered by the Platform (e.g., Recruiter_Package delivery, notifications), THE Platform SHALL log the email with: recipient address, subject line, UTC timestamp, and delivery status (one of: sent, delivered, failed).
3. WHEN an inbound email is received on the synchronized admin mailbox, THE Platform SHALL attempt to associate it with a Candidate or Job_Description record by matching the sender's email address against registered Candidate email addresses and by matching any reference identifier present in the subject line or body.
4. WHEN an inbound email is successfully associated with a record, THE Platform SHALL append it to that record's communication log within 60 seconds of receipt.
5. THE Platform SHALL display the synchronized email log to Admins in an interface searchable by sender address, recipient address, subject line, and date range.
6. IF an inbound email cannot be associated with an existing record, THEN THE Platform SHALL log it as unmatched and send an in-platform notification to the designated Admin for manual review.
7. THE Platform SHALL NOT expose admin email credentials or mailbox content to Candidates or Seniors.
8. THE Platform SHALL allow Admins to compose and send emails directly from the Platform interface using the synchronized email account.

---

### Requirement 18: WhatsApp Integration

**User Story:** As an Admin, I want the platform to integrate with WhatsApp for candidate communications, so that I can reach candidates through their preferred messaging channel and log those interactions centrally.

#### Acceptance Criteria

1. THE Platform SHALL integrate with a WhatsApp Business API account to send and receive WhatsApp messages on behalf of the Platform.
2. THE Platform SHALL allow Admins to send WhatsApp messages to Candidates directly from the Platform interface, using the Candidate's registered phone number.
3. WHEN an outbound WhatsApp message is sent, THE Platform SHALL log: recipient phone number, message content, UTC timestamp, and delivery status (one of: sent, delivered, read, failed).
4. WHEN an inbound WhatsApp message is received, THE Platform SHALL match it to the corresponding Candidate record by the sender's phone number and append it to that Candidate's communication log.
5. IF an inbound WhatsApp message is received from a phone number not registered to any Candidate, THEN THE Platform SHALL log the message as unmatched and notify the designated Admin for manual review.
6. THE Platform SHALL display the WhatsApp communication log for a Candidate to Admins within the Candidate's profile view.
7. IF a WhatsApp message delivery fails, THEN THE Platform SHALL log the failure with an error reason and send an in-platform notification to the sending Admin.
8. THE Platform SHALL NOT expose WhatsApp API credentials or integration configuration to Candidates or Seniors.
9. THE Platform SHALL support sending pre-approved templated alert messages (e.g., application status updates, interview reminders) to Candidates via WhatsApp.

---

### Requirement 19: Notifications and Alerts

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

---

### Requirement 20: Audit Trail and Data Integrity

**User Story:** As an Admin, I want a complete, tamper-evident audit trail of all significant platform actions, so that I can review decisions, resolve disputes, and demonstrate accountability for candidate data handling.

#### Acceptance Criteria

1. THE Platform SHALL record an audit log entry for every action that creates, modifies, or deletes a platform entity, including: user registration, role changes, CV uploads, profile updates, Application submissions, Review submissions, Note edits, Recruiter_Package sends, and email/WhatsApp communications.
2. EACH audit log entry SHALL contain: the identity of the actor, the action performed, the identifier of the affected entity, the previous field-level values of all modified fields (for modification actions), and a UTC timestamp with millisecond precision.
3. IF any user, including an Admin, attempts to modify or delete an audit log entry, THEN THE Platform SHALL reject the operation, return an error indicating the action is not permitted, and record the attempt as a new audit log entry identifying the actor and the targeted log entry identifier.
4. THE Platform SHALL allow Admins to search and filter the audit log by actor identity, action type, entity type, entity identifier, and date range.
5. IF a system operation fails partway through a multi-step transaction, THEN THE Platform SHALL roll back all partial changes so that every affected entity returns to its state prior to the operation, and SHALL record the failure as an audit log entry without persisting any intermediate entity state.
6. FOR every entity tracked in the audit log, applying the recorded field-level changes from all audit log entries for that entity in ascending timestamp order SHALL reproduce the current state of that entity.

---

## Cross-Cutting Constraints

### Data Retention
- Candidate data (profiles, CVs, reviews, notes, messages) SHALL be retained for a minimum of 5 years after the Candidate's last activity, unless the Candidate explicitly requests deletion and applicable law permits it.
- Recruiter_Package archives SHALL be retained indefinitely.

### Performance
- WHILE the platform is under normal operating load (up to 500 concurrent users), THE Platform SHALL respond to any user-initiated read or write request within 3 seconds.
- THE AI_Engine SHALL complete CV analysis, tailoring, and JD extraction tasks within 30 seconds under normal operating conditions.

### Availability
- THE Platform SHALL maintain availability of at least 99.5% measured monthly, excluding scheduled maintenance windows communicated at least 24 hours in advance.

### Localization
- THE Platform SHALL support Arabic and English in the user interface, with the language preference set per user account.
- WHEN content is submitted in Arabic, THE Platform SHALL store and display it in Arabic without transliteration or modification.

### Security
- THE Platform SHALL enforce HTTPS for all communications between clients and the platform.
- THE Platform SHALL store passwords using a salted cryptographic hashing algorithm with a minimum of bcrypt cost factor 12 or equivalent.
- THE Platform SHALL enforce session expiration after 30 minutes of inactivity.

---

## Correctness Properties for Testing

The following properties are amenable to property-based testing and should be used to verify platform correctness:

### CV Version Control (Requirement 5)
- **Monotonic version numbers**: For all Candidates, the version number of each successive CV_Version upload SHALL be strictly greater than all prior version numbers. `∀ v₁, v₂ ∈ CV_Versions(candidate): upload_time(v₁) < upload_time(v₂) → version_number(v₁) < version_number(v₂)`
- **Upload integrity round-trip**: For any CV file uploaded, the file retrieved from storage SHALL be byte-for-byte identical to the file uploaded. `retrieve(store(file)) == file`
- **Version count invariant**: After N successful uploads, the number of stored CV_Versions for that Candidate SHALL equal N.

### Candidate Scoring (Requirement 8)
- **Monotone skill superset**: For any two Candidates A and B scored against the same Job_Description, if skills(A) ⊇ skills(B) then score(A, JD) ≥ score(B, JD).
- **Score range invariant**: For all computed Candidate_Scores, the value SHALL be in [0, 100].
- **Score determinism**: Running the scoring algorithm twice with the same Candidate profile and JD_Extraction SHALL produce the same score.

### Review Timeline (Requirement 12)
- **Append-only ordering**: For all Reviews in a Review_Timeline, timestamps SHALL be non-decreasing. `∀ r₁, r₂ ∈ timeline: index(r₁) < index(r₂) → timestamp(r₁) ≤ timestamp(r₂)`
- **Count invariant after append**: After submitting N Reviews for a Candidate, `len(review_timeline(candidate)) == N`.

### Application Deduplication (Requirement 14)
- **Idempotent application**: Submitting the same Candidate + Job_Description pair more than once SHALL always result in exactly one Application record. `submit(c, jd); submit(c, jd) → count(applications(c, jd)) == 1`

### JD Extraction Round-Trip (Requirement 7)
- **Keyword containment**: For all keywords in a completed JD_Extraction, each keyword SHALL appear in or be semantically derivable from the source Job_Description text.

### Audit Log (Requirement 20)
- **Completeness**: For every entity modification, the audit log SHALL contain at least one entry referencing that entity and that modification type.
- **Immutability**: Reading the audit log at time T₂ > T₁ SHALL return all entries that were present at T₁, plus any new entries. No entry present at T₁ SHALL be absent at T₂.

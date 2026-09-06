# PRD — Phase 3: Integrations & Operations

## Overview

Phase 3 closes the communication loop. It adds real-time in-app messaging between candidates and seniors, synchronises the admin email inbox so all candidate-related correspondence is logged centrally, and connects the WhatsApp Business API so admins can reach candidates through their most-used channel. After this phase the platform is the single source of truth for every interaction with a candidate — from initial contact to recruiter handoff.

**Covers requirements:** 16, 17, 18

**Depends on:** Phase 1 (all requirements), Phase 2 (notification infrastructure — Req 19)

---

## Goals

- Replace direct WhatsApp and email threads that currently happen outside the platform.
- Give seniors a structured, opt-in channel to offer mentorship without exposing personal contact details.
- Give admins a single inbox view that correlates every inbound email or WhatsApp message with the right candidate record automatically.
- Log every outbound and inbound communication with full delivery status so nothing is lost.

## Non-Goals

- All Phase 1 and Phase 2 functionality is assumed complete and out of scope here.
- Third-party video or voice calling.
- Automated response bots or AI-generated message suggestions (potential future phase).

---

## User Roles in Scope

| Role | New capabilities added in Phase 3 |
|---|---|
| Admin | View all chat histories (compliance), compose and send emails from the platform, send WhatsApp messages to candidates, view unified communication log per candidate |
| Candidate | Initiate and participate in chat sessions with Seniors, receive WhatsApp notifications and status updates |
| Senior | Opt in to receive chat requests, participate in chat sessions, optionally share professional email within chat |

---

## Requirements

---

### Requirement 16 — In-App Chat Between Seniors and Candidates

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

### Requirement 17 — Email Synchronisation with Admin Mailbox

**User Story:** As an Admin, I want the platform to synchronise with the core admin email inbox, so that all candidate-related correspondence is logged centrally and no communication is lost.

#### Acceptance Criteria

1. THE Platform SHALL integrate with a designated admin email account via IMAP/SMTP or a supported email API to send and receive emails on behalf of the Platform.
2. WHEN an outbound email is triggered by the Platform (e.g., Recruiter_Package delivery, notifications), THE Platform SHALL log the email with: recipient address, subject line, UTC timestamp, and delivery status (one of: sent, delivered, failed).
3. WHEN an inbound email is received on the synchronised admin mailbox, THE Platform SHALL attempt to associate it with a Candidate or Job_Description record by matching the sender's email address against registered Candidate email addresses and by matching any reference identifier present in the subject line or body.
4. WHEN an inbound email is successfully associated with a record, THE Platform SHALL append it to that record's communication log within 60 seconds of receipt.
5. THE Platform SHALL display the synchronised email log to Admins in an interface searchable by sender address, recipient address, subject line, and date range.
6. IF an inbound email cannot be associated with an existing record, THEN THE Platform SHALL log it as unmatched and send an in-platform notification to the designated Admin for manual review.
7. THE Platform SHALL NOT expose admin email credentials or mailbox content to Candidates or Seniors.
8. THE Platform SHALL allow Admins to compose and send emails directly from the Platform interface using the synchronised email account.

---

### Requirement 18 — WhatsApp Integration

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

## Cross-Cutting Constraints

All constraints from Phase 1 apply. Additional constraints for Phase 3:

### Real-Time Delivery (Requirement 16)
- Chat_Message delivery to an online recipient SHALL complete within 5 seconds. Implementations using WebSockets or server-sent events are preferred over polling.
- Offline message queuing SHALL retain messages for up to 30 days; expired messages SHALL be discarded without affecting message history already delivered.

### External API Resilience (Requirements 17 & 18)
- THE Platform SHALL handle transient failures from the email API and WhatsApp Business API with automatic retries (at least 3 attempts with exponential back-off) before marking delivery as failed.
- API credentials for email and WhatsApp SHALL be stored in a secrets management service, not in application configuration files or the database.
- THE Platform SHALL surface integration health status to Admins (connected / degraded / disconnected) in the system settings view.

### Compliance and Privacy
- Chat histories and communication logs are subject to the 5-year data retention policy defined in Phase 1.
- Admin email mailbox content and WhatsApp credentials SHALL NOT be accessible to Candidates or Seniors at any layer (API, UI, or database query).
- All communication logs SHALL be included in the audit trail (Req 20, Phase 1) with the same tamper-evident guarantees.

---

## Integration Prerequisites

Before Phase 3 can go live the following must be provisioned outside the platform codebase:

| Dependency | Owner | Notes |
|---|---|---|
| Admin email account with IMAP/SMTP or API access | HasoubLabs ops | Google Workspace or Microsoft 365 recommended |
| WhatsApp Business API account | HasoubLabs ops | Requires Meta Business verification; allow 2–4 weeks |
| Approved WhatsApp message templates | HasoubLabs ops | Required before templated alerts can be sent |
| Senior opt-in flow in product | Product/design | Seniors must explicitly enable chat availability |

---

## Dependencies and Exit Criteria

Phase 3 **requires** Phase 1 to be complete. The notification infrastructure from Phase 2 (Req 19) must be live before Phase 3 launches, because chat and email events generate notifications.

Phase 3 is considered complete when:
- Candidates can initiate and exchange messages with opted-in Seniors in real time.
- Offline messages are queued and delivered on next login within the 30-day window.
- Admins can view all chat histories.
- Inbound and outbound emails are logged against candidate/JD records automatically; unmatched emails alert the admin.
- Admins can compose and send emails from within the platform.
- Admins can send WhatsApp messages to candidates and view delivery status.
- Inbound WhatsApp messages are matched to candidate records automatically; unmatched messages alert the admin.
- Pre-approved WhatsApp templates can be sent for application status updates and interview reminders.
- Integration health status is visible in admin settings.
- All acceptance criteria above pass in a staging environment with live API credentials.

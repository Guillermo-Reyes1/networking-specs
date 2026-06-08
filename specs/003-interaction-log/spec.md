# Feature Specification: Interaction Log

**Feature Branch**: `003-interaction-log`

**Created**: 2026-06-07

**Status**: Draft

**Input**: User description: "Log follow-up interactions against an existing contact, recording date, type, and optional notes. Depends on Contact Management. All endpoints require a valid JWT access token."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Log an Interaction (Priority: P1)

A user who has just followed up with a contact wants to record that interaction. They supply the date it happened, the type of interaction, and optionally some notes, then save. This creates a timestamped entry in the contact's history.

**Why this priority**: Creating interactions is the foundation of the feature. Without it, there is no log to retrieve and no product value.

**Independent Test**: Can be fully tested by submitting a create request for a known contact and verifying the stored record. Delivers the primary product value — a persisted follow-up log entry.

**Acceptance Scenarios**:

1. **Given** an authenticated user with an existing contact, **When** they submit an interaction with a valid date and type, **Then** the interaction is created and a 201 response is returned.
2. **Given** an authenticated user with an existing contact, **When** they submit an interaction with a valid date, type, and notes, **Then** the interaction is created with all three fields stored and a 201 response is returned.
3. **Given** an authenticated user, **When** they submit an interaction with no date, **Then** the system rejects the request with a 400 response that identifies the missing field.
4. **Given** an authenticated user, **When** they submit an interaction with no type, **Then** the system rejects the request with a 400 response that identifies the missing field.
5. **Given** an authenticated user, **When** they submit an interaction against a contact ID that does not exist, **Then** the system returns 404.
6. **Given** an authenticated user, **When** they submit an interaction against a contact owned by a different user, **Then** the system returns 404 (no 403; contact existence must not be leaked).

---

### User Story 2 - Retrieve Interaction History (Priority: P2)

A user preparing for an outreach wants to review their history with a contact. They retrieve the full list of past interactions in chronological order so they can pick up the conversation naturally.

**Why this priority**: Retrieval is the value realized from logging. Without it, recorded interactions are invisible and useless.

**Independent Test**: Can be fully tested by retrieving interactions for a contact with existing records and verifying ordering; also verifying an empty list for a contact with no interactions.

**Acceptance Scenarios**:

1. **Given** a contact with multiple logged interactions, **When** the user retrieves the interaction list, **Then** interactions are returned in ascending chronological order (earliest date first).
2. **Given** a contact with no logged interactions, **When** the user retrieves the interaction list, **Then** an empty list is returned with a 200 response.
3. **Given** an authenticated user, **When** they request interactions for a contact that does not exist or belongs to another user, **Then** the system returns 404.

---

### User Story 3 - Correct or Remove an Interaction (Priority: P3)

A user who logged an interaction with an error — wrong date, wrong type, or a note that needs updating — wants to correct it. A user who logged an interaction by mistake wants to remove it entirely.

**Why this priority**: Corrections are a quality-of-life necessity but not required for the core logging and retrieval flows to work. P3 because the feature delivers value without it.

**Independent Test**: Can be fully tested by editing an interaction's fields and verifying the updated values are returned on retrieval; and by deleting an interaction and verifying it no longer appears in the list.

**Acceptance Scenarios**:

1. **Given** an authenticated user with an existing interaction on one of their contacts, **When** they submit an edit with updated date, type, or notes, **Then** the interaction is updated and a 200 response is returned with the updated record.
2. **Given** an authenticated user, **When** they submit an edit for an interaction that does not exist or belongs to another user's contact, **Then** the system returns 404.
3. **Given** an authenticated user with an existing interaction, **When** they delete it, **Then** the interaction is removed and no longer appears in the contact's interaction list; a 204 response is returned.
4. **Given** an authenticated user, **When** they attempt to delete an interaction that does not exist or belongs to another user's contact, **Then** the system returns 404.

---

### Edge Cases

- What happens when `date` is submitted in an invalid format? → 400 with a field-level error message.
- What happens when `notes` exceed the allowed length? → 400 with a field-level error message.
- What happens when `custom_type` is provided but `type` is not `"other"`? → 400 with a field-level error indicating `custom_type` is only valid when `type` is `"other"`.
- What happens when the `X-Sync-Version` header is absent or wrong? → Consistent with the behavior defined in the Contact Management spec.
- What happens when a contact ID is syntactically invalid (e.g., non-UUID)? → 404, consistent with the Contact Management spec.
- What happens when the JWT `sub` claim matches no user? → 401.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow an authenticated user to create an interaction record against one of their own contacts.
- **FR-002**: Each interaction record MUST store: `date` (required, calendar date), `type` (required, enumerated value), `notes` (optional, free text, nullable), and `custom_type` (optional, free text, nullable). `custom_type` is only permitted when `type` is `"other"`; it MUST be ignored or rejected when provided alongside any other type value.
- **FR-003**: System MUST reject any create request where `date` is absent, returning 400 with a field-level error identifying the missing field.
- **FR-004**: System MUST reject any create request where `type` is absent, returning 400 with a field-level error identifying the missing field.
- **FR-005**: System MUST return 404 when the target contact does not exist or is not owned by the authenticated user; contact ownership is derived from the JWT `sub` claim.
- **FR-006**: System MUST return all interactions for a contact sorted in ascending chronological order by date.
- **FR-007**: System MUST return 404 when interactions are requested for a contact that does not exist or is not owned by the authenticated user.
- **FR-008**: All endpoints MUST require a valid JWT access token in the `Authorization` header; absent or invalid tokens MUST receive 401.
- **FR-009**: All endpoints MUST require the `X-Sync-Version: 1` header.
- **FR-010**: System MUST NOT allow a user to create or retrieve interactions for contacts owned by another user.
- **FR-011**: System MUST allow an authenticated user to edit an existing interaction on one of their own contacts, updating any combination of `date`, `type`, `notes` or `custom_type`.
- **FR-012**: System MUST allow an authenticated user to delete an existing interaction on one of their own contacts. Soft-delete vs. hard-delete follows the strategy decided during the Contact Management specify/clarify cycle.

### Key Entities

- **Interaction**: A logged follow-up event tied to a specific contact. Attributes: system-assigned ID, contact reference, `date` (calendar date), `type` (enumerated), `notes` (free text, nullable), `custom_type` (free text, nullable — only populated when `type` is `"other"`), system-assigned creation timestamp.
- **Interaction Type**: An enumerated set of values representing the nature of a follow-up. Values: `call`, `email`, `meeting`, `coffee`, `message`, `other`. When `type` is `"other"`, the optional `custom_type` field accepts a free-text description of the interaction.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can log an interaction against a contact (date, type, optional notes) without confusion — task completion rate of 95% or higher on first attempt.
- **SC-002**: Interaction history for any contact is presented in strict ascending chronological order in 100% of retrievals.
- **SC-003**: 100% of create requests missing a required field receive a response that identifies the missing field by name.
- **SC-004**: 100% of requests targeting another user's contact receive 404 with no information about whether the contact exists.
- **SC-005**: Interaction history load time is imperceptible under normal use (equivalent to other list endpoints in the application).

## Assumptions

- **Date granularity** is calendar date (year-month-day), not datetime with time-of-day — consistent with `date met` and `follow-up date` in the contact model.
- **Interaction types** are a fixed enumerated set at launch: `call`, `email`, `meeting`, `coffee`, `message`, `other`. When `type` is `"other"`, the optional `custom_type` field accepts a free-text label. `custom_type` is only valid when `type` is `"other"`.
- **Pagination** of the interaction list follows the strategy decided during the Contact Management specify/clarify cycle and is applied consistently to all list endpoints.
- **Soft vs. hard delete** of interactions follows the deletion strategy decided during Contact Management.
- **Real-time broadcast** of interaction mutations is deferred to the sync feature and is out of scope here.
- **Contact Management** is fully specified and its open questions resolved before this feature proceeds to planning.
- **No bulk operations** are in scope (no bulk create or bulk delete).
- **No attachments or file uploads** per interaction are in scope.

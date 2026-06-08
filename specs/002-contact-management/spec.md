# Feature Specification: Contact Management

**Feature Branch**: `002-contact-management`

**Created**: 2026-06-07

**Status**: Draft

**Input**: User description: "Users must be able to create, read, update, and delete
contacts. This feature depends on User Authentication being fully specified. All endpoints
require a valid JWT access token."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Contact Creation (Priority: P1)

An authenticated user captures a new contact by providing a name and any optional fields.
Only the name is required; all other fields may be omitted. An attempt to create a contact
without a name is rejected with a clear validation error.

**Why this priority**: Contact creation is the core action of the product — everything
else (retrieval, update, delete) depends on contacts existing. It is also the most
time-sensitive flow: a student at a networking event needs to save a contact in seconds.

**Independent Test**: Can be fully tested by creating a contact with only a name,
creating one with all fields, and attempting creation without a name — all independently
of retrieval or update functionality.

**Acceptance Scenarios**:

1. **Given** an authenticated user and a request body containing only a valid name,
   **When** the user submits a create-contact request, **Then** the system creates the
   contact and returns the new contact record including its assigned identifier.
2. **Given** an authenticated user and a request body containing name plus all optional
   fields (company, role, where met, date met, follow-up date, notes, priority),
   **When** the user submits a create-contact request, **Then** the system creates the
   contact with all provided fields and returns the complete record.
3. **Given** an authenticated user and a request body with no name field (or an empty
   name), **When** the user submits a create-contact request, **Then** the system returns
   a clear validation error indicating that name is required and creates no contact.

---

### User Story 2 - Contact Retrieval (Priority: P2)

An authenticated user can browse all their contacts in a paginated list or look up a
specific contact by its identifier. Requesting a contact that does not exist returns a
clear 404 error.

**Why this priority**: Retrieval is needed to make captured contacts useful — the user
must be able to review and act on them. Without retrieval, creation has no value.

**Independent Test**: Can be fully tested against pre-seeded contacts by requesting the
list, requesting a specific contact by ID, and requesting a non-existent ID.

**Acceptance Scenarios**:

1. **Given** an authenticated user with one or more existing contacts, **When** the user
   requests their contact list, **Then** the system returns a paginated response containing
   their contacts and the information needed to retrieve the next page.
2. **Given** an authenticated user and the identifier of an existing contact, **When** the
   user requests that contact, **Then** the system returns the full contact record.
3. **Given** an authenticated user and an identifier that does not correspond to any
   contact, **When** the user requests that contact, **Then** the system returns a 404
   error.

---

### User Story 3 - Contact Update (Priority: P3)

An authenticated user can modify any subset of a contact's fields. Fields not included in
the update request remain unchanged. Attempting to update a contact that does not exist
returns a 404 error.

**Why this priority**: Updates allow the user to correct or enrich contact records after
capture. This is important for follow-up planning but is not blocking for the core capture
flow (P1) or review flow (P2).

**Independent Test**: Can be fully tested against a pre-existing contact by sending
partial updates and verifying unchanged fields are preserved, then attempting to update a
non-existent ID.

**Acceptance Scenarios**:

1. **Given** an authenticated user and the identifier of an existing contact, **When** the
   user submits an update request with one or more fields, **Then** the system updates
   exactly those fields and returns the full contact record with all other fields
   unchanged.
2. **Given** an authenticated user and an identifier that does not correspond to any
   contact, **When** the user submits an update request, **Then** the system returns a
   404 error.

---

### User Story 4 - Contact Deletion (Priority: P4)

An authenticated user can permanently remove a contact. Attempting to delete a contact
that does not exist returns a 404 error.

**Why this priority**: Deletion is the final CRUD operation. It is less frequently needed
than creation or retrieval and does not block any other story.

**Independent Test**: Can be fully tested by deleting an existing contact and verifying it
is no longer accessible, then attempting to delete a non-existent ID.

**Acceptance Scenarios**:

1. **Given** an authenticated user and the identifier of an existing contact, **When** the
   user submits a delete request, **Then** the system removes the contact and the contact
   is no longer returned by any retrieval endpoint.
2. **Given** an authenticated user and an identifier that does not correspond to any
   contact, **When** the user submits a delete request, **Then** the system returns a
   404 error.

---

### Edge Cases

- What happens when `priority` is provided with a value other than `hot`, `warm`, or
  `cold`? System MUST return a 400 validation error.
- What happens when `date_met` or `follow_up_date` is provided in an invalid date format?
  System MUST return a 400 validation error.
- What happens when `follow_up_date` is explicitly set to null in an update? System MUST
  clear the field (not ignore the null).
- What happens when an empty paginated list is returned (no contacts yet)? System MUST
  return an empty results array with valid pagination metadata.
- What happens when the authenticated user has contacts but requests a page beyond the
  last? System MUST return an empty results array.
- What happens when a contact ID in a URL path is malformed (not a valid UUID)? System
  MUST return a 400 or 404 error.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow an authenticated user to create a contact providing only
  the `name` field.
- **FR-002**: System MUST allow an authenticated user to create a contact with any
  combination of optional fields alongside `name`.
- **FR-003**: System MUST reject contact creation when `name` is absent or empty and
  return a field-level validation error.
- **FR-004**: System MUST return a paginated list of contacts belonging to the
  authenticated user using offset-based pagination. The response MUST include the current
  page, page size, and total contact count.
- **FR-005**: System MUST return a single contact by its unique identifier when that
  contact belongs to the authenticated user.
- **FR-006**: System MUST return a 404 error when a requested contact identifier does not
  exist or does not belong to the authenticated user.
- **FR-007**: System MUST allow an authenticated user to partially update a contact —
  only the fields present in the request are modified; all other fields remain unchanged.
- **FR-008**: System MUST return a 404 error when attempting to update a contact that does
  not exist.
- **FR-009**: System MUST allow an authenticated user to permanently delete a contact by
  its identifier (hard-delete). The contact record MUST be removed from the database and
  MUST NOT be returned by any subsequent retrieval endpoint.
- **FR-010**: System MUST return a 404 error when attempting to delete a contact that does
  not exist.
- **FR-011**: System MUST validate the `priority` field against the allowed values
  (`hot`, `warm`, `cold`) when provided; any other value MUST be rejected with a 400
  error.
- **FR-012**: System MUST validate date fields (`date_met`, `follow_up_date`) for correct
  format when provided; malformed values MUST be rejected with a 400 error.
- **FR-013**: All contact endpoints MUST require a valid JWT access token; requests
  without a valid token MUST receive a 401 error.
- **FR-014**: All contact endpoints MUST require the `X-Sync-Version: 1` header; requests
  with a missing or unrecognized version MUST receive an appropriate error.
- **FR-015**: A null value for `follow_up_date` in an update request MUST explicitly clear
  that field (not be ignored).

### Key Entities

- **Contact**: Represents a person met at a networking event. Key attributes: unique
  identifier, owning user (derived from JWT), name (required), company (nullable), role
  (nullable), where met (nullable), date met (nullable date), follow-up date (nullable
  date), notes (nullable), priority (nullable; one of hot/warm/cold), creation timestamp,
  last-updated timestamp.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A contact with only a name can be created in under 2 seconds from the moment
  the user submits the request.
- **SC-002**: The full contact list loads within 3 seconds regardless of total contact
  count.
- **SC-003**: Retrieving a single contact by identifier succeeds in under 1 second.
- **SC-004**: A partial update to a contact completes in under 2 seconds and returns the
  correct full record with unchanged fields preserved.
- **SC-005**: After deletion, the contact is no longer returned by any retrieval endpoint
  in the same session.
- **SC-006**: Validation errors for missing name or invalid field values are returned
  immediately with a message actionable enough for the user to correct their input without
  support.

## Assumptions

- The authenticated user can only access their own contacts. The server derives contact
  ownership from the JWT `sub` claim; no explicit owner field is accepted in request
  bodies.
- Priority defaults to null (unset) when not provided at creation. There is no implicit
  default value.
- The partial-update operation uses PATCH semantics: only keys present in the request body
  are updated. A key absent from the body is not touched; a key explicitly set to null
  clears the field.
- Real-time broadcast of contact mutations (create, update, delete) to connected clients
  is a responsibility of the sync feature, which is specified separately. This spec covers
  the REST API surface only.
- Contact list ordering defaults to most-recently-created first. Sort order is not
  configurable in this feature version.
- User Authentication (`001-user-auth`) is fully specified and its contract is stable.
  This feature depends on the JWT access token mechanism defined there.

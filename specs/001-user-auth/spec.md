# Feature Specification: User Authentication

**Feature Branch**: `001-user-auth`

**Created**: 2026-06-04

**Status**: Draft

**Input**: User description: "Users must be able to register with email and password, log in,
maintain a session via JWT access tokens and refresh tokens, and log out. This is the
foundation all other features depend on."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Account Registration (Priority: P1)

A new user creates an account by providing their email address and a password. On success
they receive an access token and a refresh token they can use immediately. If the email is
already taken they receive a clear error message.

**Why this priority**: Every other feature requires an authenticated user. Without
registration the product cannot be used at all. This is the entry point.

**Independent Test**: Can be fully tested by submitting a registration request and
verifying the token response, then submitting a duplicate email registration and verifying
the error — no other feature needs to be implemented.

**Acceptance Scenarios**:

1. **Given** a valid, previously unregistered email and a password meeting minimum
   requirements, **When** the user submits a registration request, **Then** the system
   returns an access token and a refresh token, and the account is created.
2. **Given** an email address already associated with an existing account, **When** the
   user submits a registration request, **Then** the system returns a clear error
   indicating the email is already in use and issues no tokens.

---

### User Story 2 - Login (Priority: P2)

A registered user provides their email and password to receive a fresh access token and
refresh token. Incorrect credentials result in a rejection with no token issued.

**Why this priority**: Users need a way back in after their session expires. Login is the
daily-use entry point once the account exists.

**Independent Test**: Can be fully tested with a pre-existing account by submitting login
requests with correct and incorrect credentials and verifying the responses independently
of any other feature.

**Acceptance Scenarios**:

1. **Given** a registered email and the correct password, **When** the user submits a
   login request, **Then** the system returns a new access token and a new refresh token.
2. **Given** a registered email and an incorrect password, **When** the user submits a
   login request, **Then** the system returns a 401 status and issues no tokens.

---

### User Story 3 - Session Maintenance and Logout (Priority: P3)

An authenticated user makes requests using their access token. When the access token
expires, a new one is obtained using the refresh token without the user needing to log in
again. When the user explicitly logs out, the refresh token is invalidated server-side and
cannot be reused.

**Why this priority**: Session continuity and logout are security requirements, but they
depend on a working login flow (P2). They are grouped because they share the same token
lifecycle.

**Independent Test**: Can be fully tested by obtaining tokens via login, making an
authenticated request with a valid access token, simulating expiry and verifying refresh,
then logging out and verifying the refresh token is rejected.

**Acceptance Scenarios**:

1. **Given** a valid, non-expired access token, **When** the user makes an authenticated
   request, **Then** the request succeeds.
2. **Given** an expired access token and a valid, non-invalidated refresh token, **When**
   the user (or the client acting on their behalf) submits the refresh token, **Then** the
   system issues a new access token without requiring the user to re-enter credentials.
3. **Given** a valid refresh token, **When** the user submits a logout request, **Then**
   the refresh token is invalidated server-side and cannot be used again.
4. **Given** an invalidated refresh token (after logout), **When** a request is made using
   that token, **Then** the system returns a 401 status.

---

### Edge Cases

- What happens when a registration request omits the email or password field entirely?
  System MUST return a 400 with a field-level error.
- What happens when a password does not meet the minimum length requirement on
  registration? System MUST return a 400 with a message indicating the password
  requirement.
- What happens when the X-Sync-Version header is missing or incorrect? System MUST return
  a response indicating a version mismatch.
- What happens when a refresh token that never existed is submitted? System MUST return a
  401.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow a user to create an account using a unique email address
  and a password.
- **FR-002**: System MUST reject registration when the supplied email address is already
  associated with an existing account and return a meaningful error.
- **FR-003**: System MUST issue a JWT access token and a refresh token upon successful
  registration.
- **FR-004**: System MUST allow a registered user to authenticate using their email address
  and password.
- **FR-005**: System MUST reject login attempts with incorrect credentials and return a 401
  status without issuing any token.
- **FR-006**: System MUST issue a new JWT access token and a new refresh token upon
  successful login.
- **FR-007**: System MUST accept and authorize requests that carry a valid, non-expired
  access token.
- **FR-008**: System MUST issue a new access token in exchange for a valid, non-invalidated
  refresh token when the access token has expired.
- **FR-009**: System MUST invalidate a refresh token server-side when the user logs out.
- **FR-010**: System MUST reject any request that presents an invalidated refresh token
  and return a 401 status.
- **FR-011**: Access tokens MUST expire after 1 hour.
- **FR-012**: Refresh tokens MUST expire after 30 days.
- **FR-013**: All API requests MUST include the `X-Sync-Version` header; requests with a
  missing or unrecognized version MUST receive an appropriate error response.
- **FR-014**: Passwords MUST meet a minimum length requirement. Passwords that do not meet
  this requirement MUST be rejected at registration time with a descriptive error.
- **FR-015**: Passwords MUST be stored in a non-reversible hashed form; the plaintext
  password MUST NOT be persisted or logged at any point.

### Key Entities

- **User**: Represents a registered account. Key attributes: unique email address, hashed
  password, account creation timestamp.
- **RefreshToken**: Represents an active session credential. Key attributes: token value
  (opaque or structured), association with a User, expiry timestamp, invalidation flag
  (set on logout or explicit revocation).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new user completes account registration and holds usable tokens within
  3 seconds of submitting their credentials.
- **SC-002**: A returning user completes login and holds usable tokens within 2 seconds of
  submitting their credentials.
- **SC-003**: An authenticated request with a valid access token succeeds without any
  additional user action required.
- **SC-004**: Token refresh is transparent — the user is never prompted to re-enter
  credentials while their refresh token remains valid and non-expired.
- **SC-005**: After logout, the user's session is fully invalidated; no further
  authenticated requests succeed using the previous tokens.
- **SC-006**: Invalid or malformed registration and login inputs produce clear, actionable
  error messages that allow the user to correct their input without contacting support.

## Assumptions

- Password minimum length is 8 characters. This is a widely accepted baseline and was not
  specified otherwise.
- Email format is validated to standard RFC format at registration and login. No email
  verification step (confirmation link) is required — this was not mentioned and is out of
  scope for a single-user portfolio project.
- Login failure responses (wrong password, non-existent email) return the same 401 status
  and a generic message to avoid revealing whether a given email is registered.
- Tokens are returned in the response body. How each client stores them is
  platform-specific and outside the scope of this shared specification, per the
  constitution's platform-agnostic rule.
- All auth endpoints require the `X-Sync-Version` header consistent with the rest of the
  API, even though auth itself has no versioned data model today.
- The interaction log and contact features are not yet specified; this specification covers
  only the auth surface.

# Research: User Authentication

**Feature**: `001-user-auth` | **Date**: 2026-06-04

## Decision 1: Refresh Token Strategy

**Decision**: Opaque random token stored in the database, with rotation on every use.

**Rationale**: The spec requires server-side invalidation on logout (FR-009, FR-010). A
stateless JWT refresh token cannot be individually invalidated without a blocklist, which
recreates the database lookup anyway. Storing a random opaque token is simpler and
correct. On each successful refresh, the old token is invalidated and a new one is issued,
limiting the impact of a stolen token to the window before next use.

**Alternatives considered**:
- *JWT refresh token*: Stateless, no DB lookup on refresh. Rejected — cannot invalidate
  on logout without a blocklist table, which negates the stateless benefit.
- *Rotating JWT + blocklist*: Equivalent security to opaque token but more moving parts.
  Rejected.

---

## Decision 2: Password Storage

**Decision**: Passwords MUST be hashed using an adaptive, salted hashing algorithm before
storage. Plaintext passwords MUST never be persisted or logged. The specific algorithm and
cost factor are platform-specific implementation details defined in each platform overlay.

**Rationale**: FR-015 mandates non-reversible hashed storage. An adaptive algorithm
ensures the cost of brute-force attacks scales with hardware improvements over time.
Automatic salting prevents rainbow table attacks.

---

## Decision 3: API Versioning Header Value

**Decision**: `X-Sync-Version: 1` for the initial release.

**Rationale**: The constitution mandates the `X-Sync-Version` header on all requests but
does not specify the initial value. Starting at `1` is conventional. The server MUST
reject requests with a missing or unrecognized version with a structured error response.
Version bumps are reserved for breaking changes to request or response shapes.

---

## Decision 4: Login Failure Response Strategy

**Decision**: Incorrect credentials (wrong password or unrecognized email) return the same
401 status with a generic message.

**Rationale**: Returning distinct errors for "email not found" vs "wrong password" would
allow an attacker to enumerate registered email addresses. A generic response prevents
this. Documented in auth-api.md under POST /api/auth/login.

---

## Resolved Unknowns

| Unknown | Resolution |
|---------|------------|
| Refresh token type | Opaque random string, stored in DB |
| Refresh token rotation | Yes — old token invalidated on each use |
| Password storage | Adaptive salted hash; algorithm is platform-specific |
| X-Sync-Version initial value | `1` |
| Login failure response | Generic 401 for both wrong password and unrecognized email |
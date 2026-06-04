# Data Model: User Authentication

**Feature**: `001-user-auth` | **Date**: 2026-06-04

## Entities

### User

Represents a registered account. One User exists per email address.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUID | PRIMARY KEY, auto-generated | Stable identifier; never reused |
| `email` | String | UNIQUE, NOT NULL, max 254 chars | Stored lowercase; indexed |
| `password_hash` | String | NOT NULL | bcrypt output, cost 12; plaintext never stored |
| `created_at` | Timestamp (UTC) | NOT NULL, default now() | Immutable after creation |

**Validation rules**:
- `email` must match RFC 5321 format; validated at the application layer before storage
- `password` (input) must be ≥ 8 characters; hashed immediately, hash stored
- `email` comparison is case-insensitive at the application layer (normalize to lowercase
  before lookup and storage)

---

### RefreshToken

Represents a single active refresh credential for a User. Multiple rows may exist per User
(one per active device session), but logout invalidates the specific token presented.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUID | PRIMARY KEY, auto-generated | |
| `user_id` | UUID | NOT NULL, FK → User.id | Cascade delete if User is deleted |
| `token` | String | UNIQUE, NOT NULL | Cryptographically random; see note on storage |
| `expires_at` | Timestamp (UTC) | NOT NULL | Set to `created_at + 30 days` at issuance |
| `invalidated_at` | Timestamp (UTC) | NULLABLE | NULL = active; non-NULL = revoked (set on logout) |
| `created_at` | Timestamp (UTC) | NOT NULL, default now() | |

**Notes on token storage**:
The `token` field stores the raw random string returned to the client. An alternative is
to store a hash of the token (as with passwords), making a database breach less useful to
an attacker. For this portfolio project, storing the raw token is acceptable; hashing the
refresh token is a recommended hardening step for a production system.

**State machine**:

```
Issued → Active (invalidated_at IS NULL, expires_at in future)
Active → Expired (expires_at in past, no action needed — treated as invalid on lookup)
Active → Revoked (logout sets invalidated_at = now())
Active → Rotated (refresh call: old row revoked, new row issued)
```

---

## Entity Relationships

```
User 1 ──── N RefreshToken
```

One User can have multiple active refresh tokens (one per device/session). Logout revokes
the specific token presented; it does not wipe all tokens for the user (single-device
logout). A "log out everywhere" capability is not in scope for this feature.

---

## Indexes

| Table | Column(s) | Type | Reason |
|-------|-----------|------|--------|
| `users` | `email` | UNIQUE | Uniqueness enforcement + login lookup |
| `refresh_tokens` | `token` | UNIQUE | Lookup by token value on refresh/logout |
| `refresh_tokens` | `user_id` | Index | FK traversal (future: list sessions) |
| `refresh_tokens` | `expires_at` | Index | Cleanup queries for expired rows |

---

## Access Token (JWT — not stored)

The access token is a signed JWT and is **not** stored in the database. Its claims are
verified by checking the signature against the server secret and the `exp` claim against
the current time.

| Claim | Value | Notes |
|-------|-------|-------|
| `sub` | User.id (UUID string) | Subject — identifies the authenticated user |
| `email` | User.email | Included for convenience; reduces DB lookups per request |
| `iat` | Unix timestamp | Issued-at time |
| `exp` | iat + 3600 | Expires 1 hour after issuance |

**Signing algorithm**: HS256 (HMAC-SHA256) with a server-side secret. For a portfolio
project this is sufficient. RS256 (asymmetric) is recommended if tokens must be verified
by third-party services.

# API Contract: User Authentication

**Feature**: `001-user-auth` | **Date**: 2026-06-04
**Base URL**: `/api/auth`

## Common Rules

All endpoints:
- Accept and return `Content-Type: application/json`
- REQUIRE the `X-Sync-Version: 1` header
- Return a structured error body on all non-2xx responses

### Missing or Unsupported X-Sync-Version

**Response**: `400 Bad Request`

```json
{
  "error": "UNSUPPORTED_VERSION",
  "message": "Missing or unsupported X-Sync-Version header."
}
```

---

## POST /api/auth/register

Register a new user account.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Body**:

```json
{
  "email": "user@example.com",
  "password": "minimum8chars"
}
```

| Field | Type | Rules |
|-------|------|-------|
| `email` | string | Valid RFC 5321 email format. Required. |
| `password` | string | Minimum 8 characters. Required. |

### Responses

**201 Created** — Registration successful, tokens issued.

```json
{
  "access_token": "<jwt>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "<opaque-random-string>"
}
```

| Field | Notes |
|-------|-------|
| `access_token` | Signed JWT. Include as `Authorization: Bearer <token>` on subsequent requests. |
| `token_type` | Always `"Bearer"`. |
| `expires_in` | Seconds until access token expires (3600 = 1 hour). |
| `refresh_token` | Opaque string. Store securely; use to obtain a new access token after expiry. |

---

**400 Bad Request** — Validation failure.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "email": "Invalid email format.",
    "password": "Password must be at least 8 characters."
  }
}
```

`fields` contains only the fields that failed validation. Fields that passed are omitted.

---

**409 Conflict** — Email already registered.

```json
{
  "error": "EMAIL_ALREADY_EXISTS",
  "message": "An account with this email address already exists."
}
```

---

## POST /api/auth/login

Authenticate an existing user.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Body**:

```json
{
  "email": "user@example.com",
  "password": "their-password"
}
```

### Responses

**200 OK** — Login successful, tokens issued.

```json
{
  "access_token": "<jwt>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "<opaque-random-string>"
}
```

Same shape as `POST /api/auth/register` 201 response.

---

**400 Bad Request** — Missing or malformed request body.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "email": "Required.",
    "password": "Required."
  }
}
```

---

**401 Unauthorized** — Incorrect credentials (wrong password or unrecognized email).
The message is intentionally generic to avoid revealing whether the email is registered.

```json
{
  "error": "INVALID_CREDENTIALS",
  "message": "Email or password is incorrect."
}
```

---

## POST /api/auth/refresh

Exchange a valid refresh token for a new access token. The presented refresh token is
invalidated and a new one is issued (token rotation).

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Body**:

```json
{
  "refresh_token": "<opaque-random-string>"
}
```

### Responses

**200 OK** — New tokens issued. The old refresh token is now invalid.

```json
{
  "access_token": "<new-jwt>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "<new-opaque-random-string>"
}
```

The client MUST replace its stored refresh token with the new one.

---

**401 Unauthorized** — Refresh token is missing, expired, or has been revoked.

```json
{
  "error": "INVALID_REFRESH_TOKEN",
  "message": "Refresh token is invalid, expired, or has been revoked."
}
```

---

## POST /api/auth/logout

Invalidate the current refresh token server-side. The client should also discard both
tokens locally.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Body**:

```json
{
  "refresh_token": "<opaque-random-string>"
}
```

The `refresh_token` must be provided so the server knows exactly which session to revoke.
The access token in the Authorization header identifies the user.

### Responses

**204 No Content** — Refresh token successfully invalidated.

No response body.

---

**401 Unauthorized** — Access token is missing or invalid.

```json
{
  "error": "UNAUTHORIZED",
  "message": "A valid access token is required."
}
```

---

## Authenticated Request Error (General)

Any endpoint that requires authentication (Authorization header) returns the following when
the token is missing, malformed, or expired:

**401 Unauthorized**:

```json
{
  "error": "UNAUTHORIZED",
  "message": "A valid access token is required."
}
```

---

## Error Code Reference

| Code | HTTP Status | Trigger |
|------|-------------|---------|
| `UNSUPPORTED_VERSION` | 400 | Missing or unrecognized `X-Sync-Version` header |
| `VALIDATION_ERROR` | 400 | Request body fails field-level validation |
| `INVALID_CREDENTIALS` | 401 | Wrong email or password at login |
| `UNAUTHORIZED` | 401 | Missing, expired, or malformed access token |
| `INVALID_REFRESH_TOKEN` | 401 | Refresh token missing, expired, or revoked |
| `EMAIL_ALREADY_EXISTS` | 409 | Registration with an already-used email |

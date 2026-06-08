# API Contract: Contact Management

**Feature**: `002-contact-management` | **Date**: 2026-06-07
**Base URL**: `/api/contacts`

## Common Rules

All endpoints:
- Accept and return `Content-Type: application/json`
- REQUIRE `Authorization: Bearer <access_token>` (JWT from auth feature)
- REQUIRE the `X-Sync-Version: 1` header
- Return a structured error body on all non-2xx responses
- Scope all contact access to the authenticated user (via JWT `sub` claim)

### Contact Object (shared response shape)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Jane Smith",
  "company": "Acme Corp",
  "role": "Software Engineer",
  "where_met": "HackNYU 2026",
  "date_met": "2026-02-14",
  "follow_up_date": "2026-02-21",
  "notes": "Interested in backend roles. Connected on LinkedIn.",
  "priority": "hot",
  "created_at": "2026-02-14T18:30:00Z",
  "updated_at": "2026-02-14T18:30:00Z"
}
```

Nullable fields (`company`, `role`, `where_met`, `date_met`, `follow_up_date`, `notes`,
`priority`) appear in every response — as `null` when not set, never omitted.

---

## GET /api/contacts

Retrieve a paginated list of the authenticated user's contacts, sorted most-recently-
created first.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Query parameters**:

| Parameter | Type | Default | Rules |
|-----------|------|---------|-------|
| `page` | integer | `1` | Must be ≥ 1 |
| `page_size` | integer | `20` | Must be between 1 and 100 inclusive |

### Responses

**200 OK**

```json
{
  "data": [
    { /* Contact Object */ },
    { /* Contact Object */ }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 47,
    "total_pages": 3
  }
}
```

An empty list (no contacts yet, or page beyond last) returns `"data": []` with valid
pagination metadata (`total: 0`, `total_pages: 0` or correct count).

---

**400 Bad Request** — Invalid query parameters.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "page_size": "Must be between 1 and 100."
  }
}
```

---

## POST /api/contacts

Create a new contact for the authenticated user.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Body**:

```json
{
  "name": "Jane Smith",
  "company": "Acme Corp",
  "role": "Software Engineer",
  "where_met": "HackNYU 2026",
  "date_met": "2026-02-14",
  "follow_up_date": "2026-02-21",
  "notes": "Interested in backend roles.",
  "priority": "hot"
}
```

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `name` | string | Yes | Non-empty, non-whitespace-only. Max 255 chars. |
| `company` | string | No | Max 255 chars. Omit or `null` to leave unset. |
| `role` | string | No | Max 255 chars. |
| `where_met` | string | No | Max 500 chars. |
| `date_met` | string | No | `YYYY-MM-DD` format. |
| `follow_up_date` | string | No | `YYYY-MM-DD` format. |
| `notes` | string | No | No length limit. |
| `priority` | string | No | One of `hot`, `warm`, `cold`. |

### Responses

**201 Created** — Contact created. Returns the full Contact Object.

```json
{ /* Contact Object */ }
```

---

**400 Bad Request** — Validation failure.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "name": "Name is required.",
    "priority": "Must be one of: hot, warm, cold.",
    "date_met": "Must be a valid date in YYYY-MM-DD format."
  }
}
```

`fields` contains only the failing fields.

---

## GET /api/contacts/:id

Retrieve a single contact by its identifier.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameter**: `:id` — UUID of the contact.

### Responses

**200 OK** — Returns the full Contact Object.

```json
{ /* Contact Object */ }
```

---

**404 Not Found** — Contact does not exist or does not belong to the authenticated user.

```json
{
  "error": "NOT_FOUND",
  "message": "Contact not found."
}
```

---

## PATCH /api/contacts/:id

Partially update a contact. Only fields present in the request body are modified.
Fields absent from the body remain unchanged. An explicit `null` for a nullable field
clears that field.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameter**: `:id` — UUID of the contact.

**Body** (all fields optional; any subset may be provided):

```json
{
  "name": "Jane Smith-Updated",
  "follow_up_date": null,
  "priority": "warm"
}
```

| Field | Rules |
|-------|-------|
| `name` | Non-empty, non-whitespace-only. Max 255 chars. |
| `company` | Max 255 chars. Explicit `null` clears the field. |
| `role` | Max 255 chars. Explicit `null` clears the field. |
| `where_met` | Max 500 chars. Explicit `null` clears the field. |
| `date_met` | `YYYY-MM-DD`. Explicit `null` clears the field. |
| `follow_up_date` | `YYYY-MM-DD`. Explicit `null` clears the field. |
| `notes` | No length limit. Explicit `null` clears the field. |
| `priority` | One of `hot`, `warm`, `cold`. Explicit `null` clears the field. |

### Responses

**200 OK** — Returns the full updated Contact Object (all fields, including unchanged ones).

```json
{ /* Contact Object */ }
```

---

**400 Bad Request** — Validation failure on a provided field.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "priority": "Must be one of: hot, warm, cold."
  }
}
```

---

**404 Not Found** — Contact does not exist or does not belong to the authenticated user.

```json
{
  "error": "NOT_FOUND",
  "message": "Contact not found."
}
```

---

## DELETE /api/contacts/:id

Permanently delete a contact (hard-delete). The contact is removed from the database and
cannot be recovered.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameter**: `:id` — UUID of the contact.

### Responses

**204 No Content** — Contact permanently deleted. No response body.

---

**404 Not Found** — Contact does not exist or does not belong to the authenticated user.

```json
{
  "error": "NOT_FOUND",
  "message": "Contact not found."
}
```

---

## Error Code Reference

| Code | HTTP Status | Trigger |
|------|-------------|---------|
| `UNSUPPORTED_VERSION` | 400 | Missing or unrecognized `X-Sync-Version` header |
| `VALIDATION_ERROR` | 400 | Request body or query params fail field-level validation |
| `UNAUTHORIZED` | 401 | Missing, expired, or malformed access token |
| `NOT_FOUND` | 404 | Contact does not exist or belongs to a different user |

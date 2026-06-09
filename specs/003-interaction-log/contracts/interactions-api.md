# API Contract: Interaction Log

**Feature**: `003-interaction-log` | **Date**: 2026-06-07
**Base URL**: `/api/contacts/:contact_id/interactions`

## Common Rules

All endpoints:
- Accept and return `Content-Type: application/json`
- REQUIRE `Authorization: Bearer <access_token>` (JWT from auth feature)
- REQUIRE the `X-Sync-Version: 1` header
- Return a structured error body on all non-2xx responses
- Verify contact ownership before processing the interaction: the contact identified by `:contact_id` must exist and belong to the authenticated user (JWT `sub` claim). If not, return 404.

### Interaction Object (shared response shape)

```json
{
  "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
  "contact_id": "550e8400-e29b-41d4-a716-446655440000",
  "date": "2026-06-07",
  "type": "coffee",
  "notes": "Discussed potential collaboration on open-source project.",
  "custom_type": null,
  "created_at": "2026-06-07T14:22:00Z",
  "updated_at": "2026-06-07T14:22:00Z"
}
```

`custom_type` appears in every response — as `null` when `type` is not `"other"`, never omitted.
`updated_at` appears in every response and is never null. It equals `created_at` for interactions that have never been patched. Added by `004-cross-device-sync` for conflict detection during offline queue replay.

---

## POST /api/contacts/:contact_id/interactions

Log a new interaction against the specified contact.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameter**: `:contact_id` — UUID of the contact.

**Body**:

```json
{
  "date": "2026-06-07",
  "type": "coffee",
  "notes": "Discussed potential collaboration on open-source project."
}
```

With `custom_type` (only when `type` is `"other"`):

```json
{
  "date": "2026-06-07",
  "type": "other",
  "custom_type": "Hackathon mentoring session",
  "notes": "Helped them debug their MVP during finals."
}
```

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `date` | string | Yes | `YYYY-MM-DD` format. Must be a valid calendar date. |
| `type` | string | Yes | One of: `call`, `email`, `meeting`, `coffee`, `message`, `other`. |
| `notes` | string | No | Free text. Omit or `null` to leave unset. |
| `custom_type` | string | No | Max 255 chars. Only permitted when `type` is `"other"`. Providing it with any other type returns 400. |

### Responses

**201 Created** — Interaction logged. Returns the full Interaction Object.

```json
{
  "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
  "contact_id": "550e8400-e29b-41d4-a716-446655440000",
  "date": "2026-06-07",
  "type": "coffee",
  "notes": "Discussed potential collaboration on open-source project.",
  "custom_type": null,
  "created_at": "2026-06-07T14:22:00Z",
  "updated_at": "2026-06-07T14:22:00Z"
}
```

---

**400 Bad Request** — Validation failure.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "date": "Date is required.",
    "type": "Must be one of: call, email, meeting, coffee, message, other."
  }
}
```

`fields` contains only the failing fields. For `custom_type` misuse:

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "custom_type": "custom_type is only valid when type is \"other\"."
  }
}
```

---

**401 Unauthorized** — Missing or invalid access token.

```json
{
  "error": "UNAUTHORIZED",
  "message": "A valid access token is required."
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

## GET /api/contacts/:contact_id/interactions

Retrieve all interactions for the specified contact in ascending chronological order.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameter**: `:contact_id` — UUID of the contact.

No query parameters. All interactions are returned in a single response.

### Responses

**200 OK** — Returns an array of Interaction Objects sorted by `date` ascending, then `created_at` ascending for ties.

```json
{
  "data": [
    {
      "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
      "contact_id": "550e8400-e29b-41d4-a716-446655440000",
      "date": "2026-02-14",
      "type": "email",
      "notes": "Sent intro email.",
      "custom_type": null,
      "created_at": "2026-02-14T09:00:00Z",
      "updated_at": "2026-02-14T09:00:00Z"
    },
    {
      "id": "8c4f0a23-2b3c-4d5e-9f5a-667788990011",
      "contact_id": "550e8400-e29b-41d4-a716-446655440000",
      "date": "2026-06-07",
      "type": "coffee",
      "notes": "Met at campus café.",
      "custom_type": null,
      "created_at": "2026-06-07T14:22:00Z",
      "updated_at": "2026-06-07T14:22:00Z"
    }
  ]
}
```

An empty list (no interactions yet) returns `"data": []`.

---

**401 Unauthorized** — Same shape as POST 401.

---

**404 Not Found** — Contact does not exist or does not belong to the authenticated user.

```json
{
  "error": "NOT_FOUND",
  "message": "Contact not found."
}
```

---

## PATCH /api/contacts/:contact_id/interactions/:id

Edit an existing interaction. Only fields present in the request body are modified.
Absent fields remain unchanged. An explicit `null` for `notes` or `custom_type` clears that field.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameters**:
- `:contact_id` — UUID of the contact.
- `:id` — UUID of the interaction.

**Body** (all fields optional; any subset may be provided):

```json
{
  "date": "2026-06-08",
  "notes": null
}
```

| Field | Rules |
|-------|-------|
| `date` | `YYYY-MM-DD`. Must be a valid calendar date. |
| `type` | One of: `call`, `email`, `meeting`, `coffee`, `message`, `other`. |
| `notes` | Free text. Explicit `null` clears the field. |
| `custom_type` | Max 255 chars. Explicit `null` clears the field. Only valid when resulting `type` (after update) is `"other"`. Providing it when the final type is not `"other"` returns 400. If `type` is changed away from `"other"` and `custom_type` is not provided, the server automatically clears `custom_type` to null. |

### Responses

**200 OK** — Returns the full updated Interaction Object (all fields).

```json
{
  "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
  "contact_id": "550e8400-e29b-41d4-a716-446655440000",
  "date": "2026-06-08",
  "type": "coffee",
  "notes": null,
  "custom_type": null,
  "created_at": "2026-06-07T14:22:00Z",
  "updated_at": "2026-06-08T10:00:00Z"
}
```

---

**400 Bad Request** — Validation failure on a provided field.

```json
{
  "error": "VALIDATION_ERROR",
  "fields": {
    "type": "Must be one of: call, email, meeting, coffee, message, other."
  }
}
```

---

**401 Unauthorized** — Same shape as POST 401.

---

**404 Not Found** — Contact or interaction does not exist, or does not belong to the authenticated user.

```json
{
  "error": "NOT_FOUND",
  "message": "Interaction not found."
}
```

---

## DELETE /api/contacts/:contact_id/interactions/:id

Permanently delete an interaction (hard-delete). The row is removed and cannot be recovered.

### Request

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | Yes |
| `X-Sync-Version` | `1` | Yes |

**Path parameters**:
- `:contact_id` — UUID of the contact.
- `:id` — UUID of the interaction.

### Responses

**204 No Content** — Interaction permanently deleted. No response body.

---

**401 Unauthorized** — Same shape as POST 401.

---

**404 Not Found** — Contact or interaction does not exist, or does not belong to the authenticated user.

```json
{
  "error": "NOT_FOUND",
  "message": "Interaction not found."
}
```

---

## Error Code Reference

| Code | HTTP Status | Trigger |
|------|-------------|---------|
| `UNSUPPORTED_VERSION` | 400 | Missing or unrecognized `X-Sync-Version` header |
| `VALIDATION_ERROR` | 400 | Request body fails field-level validation |
| `UNAUTHORIZED` | 401 | Missing, expired, or malformed access token |
| `NOT_FOUND` | 404 | Contact or interaction does not exist, or belongs to a different user |

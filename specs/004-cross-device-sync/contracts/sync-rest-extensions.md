# Contract: Sync REST Extensions

**Feature**: `004-cross-device-sync` | **Date**: 2026-06-08

This document describes additions to the existing REST API contracts required by the offline sync feature. These are extensions to the Contact Management API (`002-contact-management`) and Interaction Log API (`003-interaction-log`) — the base endpoints, request shapes, and success responses are unchanged.

---

## New Request Header: `If-Match`

### Purpose

Enables optimistic concurrency control during offline queue replay. The client includes the `updated_at` timestamp of the entity as it was recorded at enqueue time. The server compares this against the current `updated_at` value. If they differ, a conflict is detected and a 409 response is returned instead of applying the mutation.

### Applicability

The `If-Match` header is used on **PATCH and DELETE** requests only, and only during **offline queue replay**. It is optional on interactive (online) requests.

| Header | Format | Example |
|--------|--------|---------|
| `If-Match` | ISO 8601 UTC timestamp string | `If-Match: "2026-06-08T10:00:00Z"` |

The value MUST be quoted (as per RFC 7232 ETag syntax).

### Behavior by Scenario

| `If-Match` present | Current `updated_at` matches | Outcome |
|--------------------|------------------------------|---------|
| Yes | Yes | Mutation applied normally; 200 or 204 returned |
| Yes | No | Conflict detected; 409 returned with current entity state |
| No | N/A | Mutation applied unconditionally (no conflict check) |

**The absence of `If-Match` is the "force overwrite" path** — used when the user explicitly chooses "local wins" after a conflict prompt. The server applies the mutation without checking `updated_at`.

---

## New Response: 409 Conflict

Returned on PATCH and DELETE requests when an `If-Match` header is present and the current `updated_at` of the entity does not match the provided value.

### Response Body

```json
{
  "error": "CONFLICT",
  "message": "The entity has been modified since your last read.",
  "server_version": { }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `error` | String | Always `"CONFLICT"` |
| `message` | String | Human-readable description |
| `server_version` | Object | The full current entity as stored on the server (same shape as the normal GET/PATCH response for that entity) |

### Contact Conflict Example

PATCH `/api/contacts/550e8400-e29b-41d4-a716-446655440000` with `If-Match: "2026-06-08T10:00:00Z"` when the server's current `updated_at` is `"2026-06-08T11:30:00Z"`:

```json
{
  "error": "CONFLICT",
  "message": "The entity has been modified since your last read.",
  "server_version": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "aaaabbbb-cccc-dddd-eeee-ffffaaaabbbb",
    "name": "Jordan Lee",
    "company": "Acme Corp",
    "role": "Senior Engineer",
    "where_met": "TechConf 2026",
    "date_met": "2026-06-08",
    "follow_up_date": "2026-07-01",
    "notes": "Promoted last month.",
    "priority": "hot",
    "created_at": "2026-06-08T10:00:00Z",
    "updated_at": "2026-06-08T11:30:00Z"
  }
}
```

### Interaction Conflict Example

PATCH `/api/contacts/:contact_id/interactions/7b3e9f12-...` with `If-Match: "2026-06-08T12:00:00Z"` when the server's current `updated_at` is `"2026-06-08T13:45:00Z"`:

```json
{
  "error": "CONFLICT",
  "message": "The entity has been modified since your last read.",
  "server_version": {
    "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
    "contact_id": "550e8400-e29b-41d4-a716-446655440000",
    "date": "2026-06-08",
    "type": "meeting",
    "notes": "Changed to an in-person meeting.",
    "custom_type": null,
    "created_at": "2026-06-08T12:00:00Z",
    "updated_at": "2026-06-08T13:45:00Z"
  }
}
```

---

## Conflict Resolution Flows

### User Chooses "Server Wins"

1. Client receives 409 — `server_version` contains the current server state.
2. Client presents user with local (queued) version and `server_version`.
3. User selects server version.
4. Client discards the queued entry.
5. Client updates its local state with the `server_version` from the 409 body.
6. **No additional HTTP request required.**

### User Chooses "Local Wins"

1. Client receives 409 — `server_version` contains the current server state.
2. Client presents user with local (queued) version and `server_version`.
3. User selects local version.
4. Client resubmits the original PATCH or DELETE request **without the `If-Match` header**.
5. Server applies the mutation unconditionally and returns the normal 200 or 204 response.
6. Client removes the entry from the offline queue.

---

## New Response: `updated_at` in Interaction Objects

As documented in `data-model.md`, the `interactions` table gains an `updated_at` column. All Interaction API responses that return an interaction object now include `updated_at`. This field was not present in the original Interaction Log contract (`003-interaction-log/contracts/interactions-api.md`).

**Updated Interaction Object shape** (additions highlighted):

```json
{
  "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
  "contact_id": "550e8400-e29b-41d4-a716-446655440000",
  "date": "2026-06-07",
  "type": "coffee",
  "notes": "Discussed potential collaboration.",
  "custom_type": null,
  "created_at": "2026-06-07T14:22:00Z",
  "updated_at": "2026-06-08T09:00:00Z"
}
```

`updated_at` is always present and never null. It equals `created_at` for interactions that have never been patched.

---

## Error Code Reference (additions to existing tables)

| Code | HTTP Status | Trigger |
|------|-------------|---------|
| `CONFLICT` | 409 | `If-Match` header present and entity `updated_at` does not match |

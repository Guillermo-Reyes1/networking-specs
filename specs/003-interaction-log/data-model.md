# Data Model: Interaction Log

**Feature**: `003-interaction-log` | **Date**: 2026-06-07

## Entities

### Interaction

Represents a single logged follow-up event against a contact. Each interaction belongs to
exactly one contact (and transitively to one user).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUID | PRIMARY KEY, auto-generated | Stable identifier; never reused |
| `contact_id` | UUID | NOT NULL, FK → contacts.id | CASCADE DELETE when contact is deleted |
| `date` | Date | NOT NULL | ISO 8601 calendar date (`YYYY-MM-DD`); no time component |
| `type` | Enum | NOT NULL | One of: `call`, `email`, `meeting`, `coffee`, `message`, `other` |
| `notes` | Text | NULLABLE | Free-text; no length limit enforced at DB level |
| `custom_type` | String | NULLABLE, max 255 chars | Only populated when `type = "other"`; null for all other type values |
| `created_at` | Timestamp (UTC) | NOT NULL, default now() | Immutable after creation; used as tie-breaker for sort |
| `updated_at` | Timestamp (UTC) | NOT NULL, default now() | Updated on every PATCH; managed by ORM lifecycle hook or database trigger. Added by `004-cross-device-sync` for conflict detection. |

**Validation rules**:
- `date` must be a valid calendar date in `YYYY-MM-DD` format; required on create.
- `type` must be one of the six enumerated values; required on create.
- `custom_type` is only permitted when `type` is `"other"`. Providing `custom_type` alongside any other type value returns 400.
- On PATCH: if `type` is set to a non-`"other"` value and `custom_type` is not explicitly provided, the server MUST clear `custom_type` to null.
- All string fields are trimmed of leading/trailing whitespace before storage.
- `updated_at` is managed automatically; it MUST NOT be accepted as a client-supplied field on create or update requests.

**Deletion**: Hard-delete. The row is removed permanently with no recovery path.

**PATCH null semantics**: An explicit `null` for `notes` or `custom_type` clears the field. An absent key leaves the field unchanged.

---

## Entity Relationships

```
User 1 ──── N Contact 1 ──── N Interaction
```

One User owns zero or more Contacts. One Contact has zero or more Interactions. Deleting
a Contact cascades to delete all of its Interactions (database FK cascade). Deleting a
User cascades to delete all of their Contacts, which in turn cascades to delete all
associated Interactions.

---

## Indexes

| Table | Column(s) | Type | Reason |
|-------|-----------|------|--------|
| `interactions` | `contact_id` | Index | Required for all list and lookup queries scoped to a contact |
| `interactions` | `contact_id`, `date`, `created_at` | Composite index | List sort: ascending `date`, tie-break ascending `created_at` |
| `interactions` | `id` | PRIMARY KEY (unique) | Already exists as PK; used for single-interaction lookups |

---

## Interaction Type Enum

| Value | Meaning |
|-------|---------|
| `call` | Phone or video call |
| `email` | Email exchange |
| `meeting` | In-person or scheduled meeting |
| `coffee` | Informal coffee chat |
| `message` | Text, DM, or chat message |
| `other` | Any other interaction type; `custom_type` field names it |

---

## Sort Order

Interactions are always returned in ascending chronological order:

1. `date` ASC (earliest date first)
2. `created_at` ASC (tie-breaker: earlier-logged entry first for same date)

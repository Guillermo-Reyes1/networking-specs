# Data Model: Contact Management

**Feature**: `002-contact-management` | **Date**: 2026-06-07

## Entities

### Contact

Represents a person met at a networking event. Each contact belongs to exactly one user.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUID | PRIMARY KEY, auto-generated | Stable identifier; never reused |
| `user_id` | UUID | NOT NULL, FK → users.id | Derived from JWT `sub` at creation; cascade delete if user deleted |
| `name` | String | NOT NULL, max 255 chars | The only required field; minimum 1 non-whitespace character |
| `company` | String | NULLABLE, max 255 chars | |
| `role` | String | NULLABLE, max 255 chars | |
| `where_met` | String | NULLABLE, max 500 chars | Free-text description of where/event |
| `date_met` | Date | NULLABLE | ISO 8601 calendar date (YYYY-MM-DD); no time component |
| `follow_up_date` | Date | NULLABLE | ISO 8601 calendar date; may be cleared by explicit null in PATCH |
| `notes` | Text | NULLABLE | Free-text; no length limit enforced at DB level |
| `priority` | Enum | NULLABLE | Allowed values: `hot`, `warm`, `cold` |
| `created_at` | Timestamp (UTC) | NOT NULL, default now() | Immutable after creation; used for default sort |
| `updated_at` | Timestamp (UTC) | NOT NULL, default now() | Updated on every PATCH; managed by ORM or trigger |

**Validation rules**:
- `name` must be non-empty and non-whitespace-only; validated at the application layer.
- `priority` must be one of `hot`, `warm`, `cold` when provided; null is allowed.
- `date_met` and `follow_up_date` must be valid calendar dates in `YYYY-MM-DD` format.
- All string fields are trimmed of leading/trailing whitespace before storage.

**Deletion**: Hard-delete only. The row is removed permanently. No `deleted_at` column.

**PATCH null semantics**: For nullable fields, an explicit `null` in the PATCH body clears
the field. An absent key leaves the field unchanged. Implementations MUST distinguish
between a missing key and an explicit `null` at the deserialization layer.

---

## Entity Relationships

```
User 1 ──── N Contact
```

One User owns zero or more Contacts. Deleting the User cascades to delete all their
Contacts (handled at the database FK level).

---

## Indexes

| Table | Column(s) | Type | Reason |
|-------|-----------|------|--------|
| `contacts` | `user_id` | Index | Required for all list and lookup queries |
| `contacts` | `user_id`, `created_at` | Composite index | Default sort (`created_at DESC`) scoped to user |
| `contacts` | `id` | PRIMARY KEY (unique) | Already exists as PK; used for single-contact lookups |

---

## Pagination Model

The list endpoint uses offset-based pagination. The server always returns:

```json
{
  "data": [ /* array of Contact objects */ ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 47,
    "total_pages": 3
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `page` | Integer | Current page number (1-indexed) |
| `page_size` | Integer | Contacts returned on this page (may be less on last page) |
| `total` | Integer | Total contact count for the user |
| `total_pages` | Integer | Ceiling of `total / page_size` |

Default: `page=1`, `page_size=20`. Maximum `page_size=100`. Requesting a page beyond
`total_pages` returns an empty `data` array with valid pagination metadata.

---

## Priority Enum

| Value | Meaning |
|-------|---------|
| `hot` | High-priority follow-up; contact is actively being pursued |
| `warm` | Medium-priority; in-progress relationship |
| `cold` | Low-priority or dormant contact |
| `null` | Not set; no priority assigned |

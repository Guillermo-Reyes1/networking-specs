# Research: Contact Management

**Feature**: `002-contact-management` | **Date**: 2026-06-07

## Decision 1: Deletion Strategy

**Decision**: Hard-delete — the contact row is permanently removed from the database.

**Rationale**: For a single-user portfolio project with no audit requirement, hard-delete
is simpler and correct. Soft-delete adds a `deleted_at` column and requires every query
to filter `WHERE deleted_at IS NULL`, which is easy to forget and introduces bugs. There
is no stated requirement for contact recovery or audit trail.

**Alternatives considered**:
- *Soft-delete*: Non-destructive; contact recoverable from DB. Rejected — adds query
  complexity to every read path, no user-facing recovery feature is in scope, and
  constitution Principle IV mandates simplicity.

---

## Decision 2: Pagination Strategy

**Decision**: Offset-based pagination — `page` (1-indexed) and `page_size` query
parameters. Response includes `pagination.page`, `pagination.page_size`,
`pagination.total`, and `pagination.total_pages`.

**Default**: `page=1`, `page_size=20`. Maximum `page_size` is 100.

**Rationale**: For a single-user personal CRM, the contact list will realistically contain
tens to low hundreds of contacts. Offset pagination is simpler to implement, reason about,
and consume. The limitation of offset pagination (duplicates/skips under concurrent
inserts) is negligible at this scale and user count (1 user).

**Alternatives considered**:
- *Cursor-based*: Consistent under concurrent mutations, better for infinite scroll.
  Rejected — unnecessary complexity for this scope; constitution Principle IV.

---

## Decision 3: Partial Update Semantics (PATCH)

**Decision**: PATCH semantics — only keys present in the request body are updated. Keys
absent from the body are left unchanged. A key explicitly set to `null` for a nullable
field clears that field.

**Rationale**: The spec explicitly requires "other fields unchanged" (FR-007). PATCH is
the correct HTTP method for partial updates. PUT (full replacement) would require the
client to re-send all existing values to avoid accidental data loss, which creates
unnecessary round-trips especially on mobile with limited bandwidth.

**Null handling**: The distinction between "absent key" (leave unchanged) and "explicit
null" (clear the field) is important for `follow_up_date`. The spec calls this out
explicitly (FR-015). Implementations must inspect the parsed body to distinguish the two
cases.

---

## Decision 4: Contact Ownership via JWT

**Decision**: Contacts are scoped to the authenticated user via the JWT `sub` claim.
No `user_id` is accepted in request bodies. All queries filter by `user_id = jwt.sub`.

**Rationale**: Single-user product. The server derives ownership from the trusted JWT
rather than accepting a user identifier from the client, which would be a security
vulnerability (IDOR — Insecure Direct Object Reference). All CRUD operations silently
scope to the current user; attempting to access another user's contact (if other users
existed) returns a 404, not a 403, to avoid leaking resource existence.

---

## Decision 5: Sort Order for Contact List

**Decision**: Default sort is `created_at DESC` (most-recently-created first). Sort order
is not configurable in this feature version.

**Rationale**: Most recently added contacts are the most relevant at a networking event
or immediately after. A fixed sort avoids the API surface of sort parameters, consistent
with constitution Principle IV.

---

## Resolved Unknowns

| Unknown | Resolution |
|---------|------------|
| Deletion strategy | Hard-delete; row permanently removed |
| Pagination strategy | Offset-based; `page` + `page_size` + `total` + `total_pages` |
| Default page size | 20; maximum 100 |
| Partial update method | PATCH; absent key = no-op, explicit null = clear |
| Contact ownership | JWT `sub` claim; not accepted in body |
| Sort order | `created_at DESC`; not configurable |
| Priority default | `null` when omitted at creation |
| 404 vs 403 for wrong-user contact | 404 (avoids IDOR leak) |

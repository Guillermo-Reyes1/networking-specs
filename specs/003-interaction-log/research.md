# Research: Interaction Log

**Feature**: `003-interaction-log` | **Date**: 2026-06-07

## Decision 1: Deletion Strategy

**Decision**: Hard-delete — the interaction row is permanently removed from the database.

**Rationale**: Inherited from the Contact Management decision (see `002-contact-management/research.md`). No audit requirement exists. Soft-delete adds query complexity to every read path with no user-facing benefit, violating constitution Principle IV.

**Alternatives considered**:
- *Soft-delete*: Rejected for the same reasons as in Contact Management — unnecessary complexity, no recovery feature in scope.

---

## Decision 2: Pagination for Interaction List

**Decision**: No pagination — the list endpoint returns all interactions for a contact in a single response.

**Rationale**: A contact's interaction history is bounded by the number of times one person has followed up with another. In a personal CRM used by a single student, this will realistically be in the low tens. Applying offset pagination here adds API surface (query params, pagination envelope) with no practical benefit at this scale. Constitution Principle IV mandates simplicity over features.

**Alternatives considered**:
- *Offset pagination (same as contacts list)*: Rejected — the contact list has unbounded growth (a user may have hundreds of contacts); interaction counts per contact are self-limiting. Different scale → different tradeoff.

---

## Decision 3: URL Structure (Nested Resource)

**Decision**: Interactions are nested under their contact: `/api/contacts/:contact_id/interactions[/:id]`.

**Rationale**: Interactions are not addressable without knowing their parent contact. Nesting encodes this ownership relationship in the URL and makes the 404 behavior for a non-existent or foreign contact unambiguous — the server checks contact ownership before looking for the interaction.

**Alternatives considered**:
- *Flat resource (`/api/interactions/:id`)*: Rejected — requires a separate ownership lookup and loses the natural URL hierarchy. Adds complexity with no benefit.

---

## Decision 4: Interaction Type Enum

**Decision**: Enumerated type field with an escape hatch. Values: `call`, `email`, `meeting`, `coffee`, `message`, `other`. When `type` is `"other"`, the optional `custom_type` field (free text, max 255 chars) may be provided to name the interaction type.

**Rationale**: A fixed, short enum covers the vast majority of networking follow-ups. The `"other"` + `custom_type` escape hatch prevents edge cases from becoming blockers for the user without introducing an open-ended field that bypasses validation. The enum is not user-extensible.

**Constraints**:
- `custom_type` MUST be null when `type` is anything other than `"other"`. Providing it alongside a non-`"other"` type returns 400.
- On PATCH: if `type` is changed from `"other"` to another value, `custom_type` is automatically cleared to null.

---

## Decision 5: Sort Order for Interaction List

**Decision**: Ascending by `date` (earliest date first). Tie-broken by `created_at` ascending (earlier-logged entry first).

**Rationale**: The spec explicitly requires chronological order. Ascending (oldest first) reads like a conversation history — natural for reviewing how a relationship developed. Tie-breaking on `created_at` gives a stable, deterministic order for interactions logged on the same day.

---

## Decision 6: PATCH Semantics (Edit Interaction)

**Decision**: Mirrors Contact Management PATCH semantics — only keys present in the request body are modified. Absent keys are unchanged. An explicit `null` on `notes` or `custom_type` clears that field.

**Special case**: If the PATCH body sets `type` to a non-`"other"` value and does not explicitly set `custom_type`, the server MUST automatically clear `custom_type` to null. This prevents stale custom type strings from persisting after the type changes.

---

## Decision 7: Interaction Ownership Verification

**Decision**: Ownership is verified in two steps: (1) look up the contact by `contact_id` scoped to `user_id = jwt.sub`; (2) look up the interaction by `id` scoped to `contact_id`. Both lookups return 404 on failure (no 403) to avoid IDOR leaks.

**Rationale**: Inherited from Contact Management Decision 4. The server must never reveal whether a resource exists to a non-owner.

---

## Resolved Unknowns

| Unknown | Resolution |
|---------|------------|
| Deletion strategy | Hard-delete; row permanently removed |
| Interaction list pagination | None — return all interactions in one response |
| URL structure | Nested: `/api/contacts/:contact_id/interactions[/:id]` |
| Interaction type values | `call`, `email`, `meeting`, `coffee`, `message`, `other` |
| Custom type field | Allowed only when `type = "other"`; rejected (400) otherwise |
| Sort order | Ascending by `date`; tie-break by `created_at` ascending |
| PATCH semantics | Same as Contact Management; absent key = no-op, explicit null = clear |
| `custom_type` on type change | Auto-cleared to null when type changes away from `"other"` |
| Ownership verification | Two-step: contact lookup → interaction lookup; both return 404 on miss |

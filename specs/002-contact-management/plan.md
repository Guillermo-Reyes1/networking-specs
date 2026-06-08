# Implementation Plan: Contact Management

**Branch**: `002-contact-management` | **Date**: 2026-06-07 | **Spec**: [spec.md](spec.md)

## Summary

Define the REST API layer for contact CRUD: create, list (offset-paginated), get by ID,
partial update (PATCH), and permanent hard-delete. All endpoints require JWT
authentication. Contacts are owned by the authenticated user via the JWT `sub` claim.
Real-time broadcast of mutations is deferred to the sync feature. Client-side storage and
offline queue handling are platform-specific and out of scope.

## Shared Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| Spec | `specs/002-contact-management/spec.md` | Acceptance scenarios and functional requirements |
| API Contract | `specs/002-contact-management/contracts/contacts-api.md` | Shared endpoint definitions consumed by all platforms |
| Data Model | `specs/002-contact-management/data-model.md` | Canonical entity definitions |
| Research | `specs/002-contact-management/research.md` | Cross-platform decisions and rationale |

## Resolved Open Questions

| Question | Resolution |
|----------|------------|
| Deletion strategy | Hard-delete — row permanently removed from DB (FR-009) |
| Pagination strategy | Offset-based — `page`, `page_size`, `total`, `total_pages` (FR-004) |
| Default page size | 20 contacts per page; maximum 100 |
| Contact ownership | Derived from JWT `sub` claim; not accepted in request body |
| Default sort order | Most-recently-created first (`created_at DESC`) |
| Priority default | `null` when not provided at creation |
| Partial update semantics | PATCH — only keys present in body are modified |
| Null in update | Explicit `null` for a nullable field clears it; absent key leaves it unchanged |
| Wrong-user contact access | Returns 404 (not 403) to avoid IDOR information leak |

## Platform Overlay Note

Implementation details — runtime, framework, ORM, request validation library, and source
code structure — are platform-specific. They MUST be defined in each platform overlay
(`networking-web`, `networking-ios`) and MUST NOT be specified here.

## Constitution Check

| Principle | Verdict | Notes |
|-----------|---------|-------|
| I. Speed of Capture First | ✅ PASS | POST /api/contacts requires only `name`. No multi-step flow. Server must return the created contact quickly so the mobile client can confirm the save. |
| II. Offline-First on Mobile | ✅ PASS | Mutations are enqueued locally on iOS; the server processes them on reconnect. This spec covers server-side REST only. Offline queue and conflict resolution are iOS-client concerns. |
| III. Real-Time Cross-Device Sync | ⚠️ NOTE | REST mutations defined here. Broadcasting change events via Socket.io is deferred to the sync feature spec. Not a violation — sync is a separate, later feature. |
| IV. Simplicity Over Features | ✅ PASS | Hard-delete over soft-delete. Offset pagination over cursor. No sort/filter options. Single user. |
| V. API-Driven Data Model | ✅ PASS | All mutations via shared REST endpoints. Contact fields are canonical per constitution. No client-specific endpoints. |

**All gates pass. Principle III sync broadcast deferred to sync feature — not a violation.**

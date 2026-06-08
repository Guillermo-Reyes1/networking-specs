# Implementation Plan: Interaction Log

**Branch**: `003-interaction-log` | **Date**: 2026-06-07 | **Spec**: [spec.md](spec.md)

## Summary

Define the REST API layer for interaction CRUD: create, list (full, chronological), edit
(PATCH), and permanent hard-delete. Interactions are nested under their parent contact at
`/api/contacts/:contact_id/interactions`. All endpoints require JWT authentication and
verify contact ownership before processing. The interaction type is an enumerated field
with an `"other"` escape hatch that unlocks a free-text `custom_type`. Real-time
broadcast of mutations is deferred to the sync feature.

## Shared Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| Spec | `specs/003-interaction-log/spec.md` | Acceptance scenarios and functional requirements |
| API Contract | `specs/003-interaction-log/contracts/interactions-api.md` | Shared endpoint definitions consumed by all platforms |
| Data Model | `specs/003-interaction-log/data-model.md` | Canonical entity definitions |
| Research | `specs/003-interaction-log/research.md` | Cross-platform decisions and rationale |
| Quickstart | `specs/003-interaction-log/quickstart.md` | End-to-end validation scenarios |

## Resolved Open Questions

| Question | Resolution |
|----------|------------|
| Append-only vs. edit/delete | Full edit (PATCH) and hard-delete — user confirmed (Option C) |
| Deletion strategy | Hard-delete — row permanently removed (inherited from Contact Management) |
| Interaction list pagination | None — all interactions returned in one response |
| URL structure | Nested: `/api/contacts/:contact_id/interactions[/:id]` |
| Interaction type enum | `call`, `email`, `meeting`, `coffee`, `message`, `other` |
| `custom_type` field | Only valid when `type = "other"`; rejected (400) otherwise |
| Sort order | `date` ASC, `created_at` ASC tie-breaker |
| PATCH semantics | Absent key = no-op; explicit null = clear (mirrors Contact Management) |
| `custom_type` on type change | Auto-cleared to null when type changes away from `"other"` |
| Ownership verification | Two-step: contact ownership → interaction ownership; both return 404 on miss |

## Platform Overlay Note

Implementation details — runtime, framework, ORM, request validation library, and source
code structure — are platform-specific. They MUST be defined in each platform overlay
(`networking-web`, `networking-ios`) and MUST NOT be specified here.

## Technical Context

**Language/Version**: Platform-specific — defined in platform overlays.

**Primary Dependencies**: Platform-specific — defined in platform overlays.

**Storage**: PostgreSQL. `interactions` table with FK to `contacts.id` (CASCADE DELETE).

**Testing**: Platform-specific — defined in platform overlays.

**Target Platform**: Multi-platform. REST API consumed by iOS and web clients.

**Project Type**: Shared, platform-agnostic REST API spec.

**Performance Goals**: Interaction history load is imperceptible under normal use (SC-005). Logging an interaction is a fast single-record insert (SC-001).

**Constraints**: JWT access token required on all endpoints. `X-Sync-Version: 1` header required. Contact ownership verified before interaction access. Hard-delete only.

**Scale/Scope**: Single user, personal CRM. Interaction count per contact: realistically low tens.

## Constitution Check

*Evaluated before Phase 0 research and re-checked after Phase 1 design. Must pass all gates.*

| Principle | Verdict | Notes |
|-----------|---------|-------|
| I. Speed of Capture First | ✅ PASS | POST requires only `date` and `type` — two fields. No confirmation dialogs, multi-step flows, or wizards in scope. The mobile client can log an interaction in a single tap. |
| II. Offline-First on Mobile | ✅ PASS | This spec covers server-side REST only. Mobile offline queue and conflict resolution for interaction mutations are iOS-platform concerns handled in the platform overlay. |
| III. Real-Time Cross-Device Sync | ⚠️ NOTE | REST mutations defined here. Broadcasting change events via Socket.io deferred to the sync feature spec — same as Contact Management. Not a violation. |
| IV. Simplicity Over Features | ✅ PASS | No pagination on interaction list. Hard-delete. Fixed type enum (no user-extensible values). Single-step nested URL. No attachments, no bulk ops. |
| V. API-Driven Data Model | ✅ PASS | All mutations via shared REST endpoints. Interaction fields are canonical. Ownership derived from JWT `sub`. No client-specific endpoints. |

**All gates pass. Principle III sync broadcast deferred to sync feature — not a violation.**

## Project Structure

### Documentation (this feature)

```text
specs/003-interaction-log/
├── plan.md              # This file
├── spec.md              # Feature spec
├── research.md          # Decisions and rationale
├── data-model.md        # Canonical entity definitions
├── quickstart.md        # End-to-end validation scenarios
├── contracts/
│   └── interactions-api.md   # Shared REST API contract
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Generated by /speckit-tasks (not yet created)
```

### Source Code

Platform-specific. Defined in platform overlays. Not specified here.

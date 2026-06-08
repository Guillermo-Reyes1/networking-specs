---
description: "Task list for Interaction Log feature implementation"
---

# Tasks: Interaction Log

**Input**: Design documents from `specs/003-interaction-log/`

**Prerequisites**: plan.md ✅ | spec.md ✅ | research.md ✅ | data-model.md ✅ | contracts/interactions-api.md ✅

**Dependency**: This feature depends on `specs/002-contact-management/` being fully
implemented. The `contacts` table, JWT middleware, and `X-Sync-Version` middleware
introduced in auth and contact management are reused here. If those features are not yet
implemented on the target platform, complete them first.

**Platform note**: Source code structure and file paths are platform-specific and defined
in each platform overlay. Tasks reference logical components and design documents.

**Tests**: Not requested. Test tasks are excluded.

**Organization**: Tasks are grouped by user story. Each story is independently
implementable against the interactions table once the foundation is in place.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (no dependency on an incomplete task)
- **[Story]**: User story this task belongs to (US1–US3)

---

## Phase 1: Setup

**Purpose**: Platform-level project initialization. Skip if already completed for auth or
contact management features.

- [ ] T001 Confirm api project is initialized with correct runtime and framework per platform overlay (may already be done from 001-user-auth or 002-contact-management)
- [ ] T002 Confirm PostgreSQL database connection is configured and reachable
- [ ] T003 [P] Confirm schema migration tooling is set up per platform overlay
- [ ] T004 [P] Confirm environment variables (`DATABASE_URL`, `JWT_SECRET`, `PORT`) are configured

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: `interactions` table, indexes, reused middleware, and shared validators that
all three user stories depend on. No user story work may begin until this phase is
complete.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 Define `interactions` table schema per `data-model.md` Interaction entity: `id` (UUID PK auto-generated), `contact_id` (UUID NOT NULL FK → contacts.id CASCADE DELETE), `date` (NOT NULL DATE), `type` (NOT NULL ENUM: `call`, `email`, `meeting`, `coffee`, `message`, `other`), `notes` (nullable TEXT), `custom_type` (nullable VARCHAR 255 — only populated when `type = "other"`), `created_at` (Timestamp UTC default now)
- [ ] T006 Add indexes per `data-model.md` Indexes section: index on `contact_id`; composite index on `(contact_id, date, created_at)` for the sorted list query
- [ ] T007 Run migrations and verify `interactions` table exists with correct columns, constraints, FK cascade, and indexes
- [ ] T008 [P] Confirm or reuse JWT access token authentication middleware: extract `Authorization: Bearer <token>`, verify signature and expiry, attach `sub` to request context; return `401 UNAUTHORIZED` if missing/invalid/expired (reuse from 001-user-auth if available)
- [ ] T009 [P] Confirm or reuse `X-Sync-Version: 1` header validation middleware: return `400 UNSUPPORTED_VERSION` on missing or unrecognized version (reuse from 001-user-auth or 002-contact-management if available)
- [ ] T010 [P] Implement contact ownership verification helper: given `contact_id` from path and `user_id` from JWT `sub`, query `contacts` table with `WHERE id = :contact_id AND user_id = :user_id`; return `404 NOT_FOUND` with `{ "error": "NOT_FOUND", "message": "Contact not found." }` if the contact does not exist or belongs to a different user (prevents IDOR), per `research.md` Decision 7 and FR-005
- [ ] T011 [P] Implement `type` enum validation utility: accepts `call`, `email`, `meeting`, `coffee`, `message`, `other`; rejects any other value with `400 VALIDATION_ERROR` and a descriptive `fields.type` error, per FR-004 and `data-model.md` Interaction Type Enum
- [ ] T012 [P] Implement `custom_type` constraint validator: if `custom_type` is present and non-null in the request body, verify that `type` (the final resolved type after the update) is `"other"`; if not, return `400 VALIDATION_ERROR` with `fields.custom_type: "custom_type is only valid when type is \"other\"."`, per FR-002 and `contracts/interactions-api.md` POST 400 shape
- [ ] T013 [P] Implement `custom_type` auto-clear rule: on PATCH, if `type` is being changed to a non-`"other"` value and `custom_type` is not explicitly included in the request body, the update MUST set `custom_type = null` in the database write, per `research.md` Decision 6 and `contracts/interactions-api.md` PATCH field rules

**Checkpoint**: Foundation ready — user story implementation can now begin.

---

## Phase 3: User Story 1 — Log an Interaction (Priority: P1) 🎯 MVP

**Goal**: Authenticated user logs an interaction against one of their contacts with `date`
and `type` required; `notes` and `custom_type` optional. Missing required fields return
`400 VALIDATION_ERROR`. Logging against a non-existent or foreign contact returns `404`.
Success returns `201` with the full Interaction Object.

**Independent Test**: `quickstart.md` Scenarios 1–6 (create happy paths and error cases)

### Implementation for User Story 1

- [ ] T014 [P] [US1] Implement `POST /api/contacts/:contact_id/interactions` route wired to auth middleware (T008) and version middleware (T009), per `contracts/interactions-api.md`
- [ ] T015 [US1] Implement contact ownership check on create: call T010 helper with `:contact_id` and JWT `sub`; propagate `404 NOT_FOUND` if check fails (depends on T014)
- [ ] T016 [US1] Implement request body validation on create: `date` required (`YYYY-MM-DD` calendar date format); `type` required via T011; `custom_type` constraint via T012; return `400 VALIDATION_ERROR` with `fields` map identifying each failing field, per FR-003, FR-004 and `contracts/interactions-api.md` POST 400 shape (depends on T015)
- [ ] T017 [US1] Implement interaction creation: insert `interactions` row with validated `contact_id`, `date`, `type`, `notes` (null if absent), `custom_type` (null unless `type = "other"`), and auto-generated `id` and `created_at`; return `201` with full Interaction Object per `contracts/interactions-api.md`, per FR-001 and FR-002 (depends on T016)

**Checkpoint**: `POST /api/contacts/:contact_id/interactions` fully functional. Run `quickstart.md` Scenarios 1–6 before proceeding.

---

## Phase 4: User Story 2 — Retrieve Interaction History (Priority: P2)

**Goal**: Authenticated user retrieves all interactions for one of their contacts in
ascending chronological order. An empty list returns `{ "data": [] }`. A non-existent or
foreign contact returns `404`.

**Independent Test**: `quickstart.md` Scenarios 7–9 (list happy path, empty list, 404 for
foreign contact) — requires at least one interaction from US1 or pre-seeded data for
ordering verification

### Implementation for User Story 2

- [ ] T018 [P] [US2] Implement `GET /api/contacts/:contact_id/interactions` route wired to auth middleware (T008) and version middleware (T009), per `contracts/interactions-api.md`
- [ ] T019 [US2] Implement contact ownership check on list: call T010 helper with `:contact_id` and JWT `sub`; propagate `404 NOT_FOUND` if check fails (depends on T018)
- [ ] T020 [US2] Implement interaction list query: `SELECT * FROM interactions WHERE contact_id = :contact_id ORDER BY date ASC, created_at ASC`; return `{ "data": [...] }` with all Interaction Objects per `contracts/interactions-api.md`; return `{ "data": [] }` (not 404) when no interactions exist for the contact, per FR-006 and `data-model.md` Sort Order (depends on T019)

**Checkpoint**: `GET /api/contacts/:contact_id/interactions` fully functional. Run `quickstart.md` Scenarios 7–9 before proceeding.

---

## Phase 5: User Story 3 — Correct or Remove an Interaction (Priority: P3)

**Goal**: Authenticated user edits an existing interaction (any combination of `date`,
`type`, `notes`, `custom_type`) or deletes it entirely. Non-existent interaction or
foreign contact/interaction returns `404`. Edit returns `200` with the full updated
Interaction Object. Delete returns `204` with no body.

**Independent Test**: `quickstart.md` Scenarios 10–16 (edit happy paths, auto-clear of
`custom_type`, edit 404, delete happy path, delete 404)

### Implementation for User Story 3

- [ ] T021 [P] [US3] Implement `PATCH /api/contacts/:contact_id/interactions/:id` route wired to auth middleware (T008) and version middleware (T009), per `contracts/interactions-api.md`
- [ ] T022 [US3] Implement two-step ownership check for PATCH: (1) call T010 helper for contact ownership; (2) query `interactions WHERE id = :id AND contact_id = :contact_id`; return `404 NOT_FOUND` with `{ "error": "NOT_FOUND", "message": "Interaction not found." }` if either check fails, per `research.md` Decision 7 (depends on T021)
- [ ] T023 [US3] Implement PATCH field validation: for each key present in the request body, validate `date` (YYYY-MM-DD if provided), `type` via T011 (if provided), `custom_type` via T012 using the final resolved type after the update; return `400 VALIDATION_ERROR` on failure (depends on T022)
- [ ] T024 [US3] Implement PATCH update: apply only keys present in body — absent key is a no-op, explicit `null` on `notes` or `custom_type` clears the field; invoke T013 auto-clear rule when `type` changes away from `"other"`; return `200` with the full updated Interaction Object (all fields, including unchanged ones), per FR-011 and `research.md` Decision 6 (depends on T023)
- [ ] T025 [P] [US3] Implement `DELETE /api/contacts/:contact_id/interactions/:id` route wired to auth middleware (T008) and version middleware (T009), per `contracts/interactions-api.md`
- [ ] T026 [US3] Implement two-step ownership check for DELETE: (1) call T010 helper for contact ownership; (2) query `interactions WHERE id = :id AND contact_id = :contact_id`; return `404 NOT_FOUND` with `{ "error": "NOT_FOUND", "message": "Interaction not found." }` if either check fails (depends on T025)
- [ ] T027 [US3] Implement hard-delete: `DELETE FROM interactions WHERE id = :id AND contact_id = :contact_id`; return `204 No Content` with no response body, per FR-012 and `research.md` Decision 1 (depends on T026)

**Checkpoint**: `PATCH` and `DELETE` fully functional. Run `quickstart.md` Scenarios 10–16 before proceeding.

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: End-to-end validation and contract conformance across all three stories.

- [ ] T028 [P] Verify all error response shapes match the Error Code Reference table in `contracts/interactions-api.md` (`UNSUPPORTED_VERSION`, `VALIDATION_ERROR`, `UNAUTHORIZED`, `NOT_FOUND`) — check error body structure and HTTP status code for each code
- [ ] T029 [P] Confirm `GET /api/contacts/:contact_id/interactions` returns `{ "data": [] }` (not 404) when the contact exists but has no interactions, per `quickstart.md` Scenario 8
- [ ] T030 [P] Confirm PATCH auto-clears `custom_type` to `null` when `type` is changed from `"other"` to any other value and `custom_type` is not explicitly provided in the body, per `quickstart.md` Scenario 12 and `research.md` Decision 6
- [ ] T031 [P] Confirm deleting a contact (via `DELETE /api/contacts/:id`) cascades and removes all associated interactions from the `interactions` table, per `data-model.md` Entity Relationships FK CASCADE rule
- [ ] T032 Run all validation scenarios in `quickstart.md` end-to-end in sequence (Scenarios 1 → 17)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — confirm or complete immediately
- **Foundational (Phase 2)**: Depends on Phase 1 + 001-user-auth and 002-contact-management implemented — **BLOCKS all user stories**
- **US1 (Phase 3)**: Depends on Phase 2 completion
- **US2 (Phase 4)**: Depends on Phase 2 completion — can run in parallel with US1 after Phase 2; requires at least one interaction for ordering verification
- **US3 (Phase 5)**: Depends on Phase 2 completion — PATCH and DELETE routes are independent of each other within the phase; depends on existing interactions for quickstart validation
- **Polish (Phase N)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Can start after Phase 2 — no dependency on US2 or US3
- **US2 (P2)**: Can start after Phase 2 — reads interactions, no dependency on US1 beyond needing data to validate ordering
- **US3 (P3)**: Can start after Phase 2 — operates on existing interactions; PATCH and DELETE routes can be implemented in parallel

### Within Each User Story

- Route handler first (T014/T018/T021/T025), then ownership check, then validation, then persistence
- T021 (PATCH route) and T025 (DELETE route) within US3 are independent — implement in parallel

### Parallel Opportunities

- Phase 2: T008, T009, T010, T011, T012, T013 are all independent utilities — run in parallel after T007
- US1 and US2: T014 (POST route) and T018 (GET route) are independent — implement in parallel if staffed
- US3: T021 (PATCH route) and T025 (DELETE route) are independent — implement in parallel

---

## Parallel Execution Example: Phase 2 (Foundational)

```
After T007 (migrations verified), launch in parallel:
  Task: "Confirm/reuse JWT auth middleware (T008)"
  Task: "Confirm/reuse X-Sync-Version middleware (T009)"
  Task: "Implement contact ownership verification helper (T010)"
  Task: "Implement type enum validator (T011)"
  Task: "Implement custom_type constraint validator (T012)"
  Task: "Implement custom_type auto-clear rule (T013)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (confirm prereqs)
2. Complete Phase 2: Foundational (CRITICAL — blocks everything)
3. Complete Phase 3: User Story 1 (log an interaction)
4. **STOP and VALIDATE**: Run `quickstart.md` Scenarios 1–6
5. Demo: authenticated user can log an interaction against a contact

### Incremental Delivery

1. Setup + Foundational → infrastructure ready
2. US1 complete → `POST` works → validate independently (MVP)
3. US2 complete → `GET` (chronological list) works → validate independently
4. US3 complete → `PATCH` + `DELETE` work → validate end-to-end
5. Polish → all contracts verified, cascade delete confirmed, edge cases confirmed

---

## Notes

- [P] tasks = independent — no incomplete predecessors, different logical components
- [Story] label maps each task to its user story for traceability back to `spec.md`
- File paths are defined per platform overlay, not in this shared spec
- T008 and T009 reference middleware from 001-user-auth — reuse if already implemented
- T010 references the `contacts` table from 002-contact-management — that feature must be fully implemented before any interaction endpoint can verify ownership
- Hard-delete means no `deleted_at` column on interactions — do not add one
- Verify each checkpoint before moving to the next phase

---
description: "Task list for Contact Management feature implementation"
---

# Tasks: Contact Management

**Input**: Design documents from `specs/002-contact-management/`

**Prerequisites**: plan.md ✅ | spec.md ✅ | research.md ✅ | data-model.md ✅ | contracts/contacts-api.md ✅

**Dependency**: This feature depends on `specs/001-user-auth/` being fully implemented.
The JWT access token middleware introduced in auth (T022 of 001-user-auth tasks) is reused
here. If the auth feature middleware is not yet available, implement it per the 001-user-auth
contracts before proceeding to Phase 2.

**Platform note**: Source code structure and file paths are platform-specific and defined in
each platform overlay. Tasks reference logical components and design documents.

**Tests**: Not requested. Test tasks are excluded.

**Organization**: Tasks are grouped by user story. Each story is independently implementable
against the contacts table once the foundation is in place.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (no dependency on an incomplete task)
- **[Story]**: User story this task belongs to (US1–US4)

---

## Phase 1: Setup

**Purpose**: Platform-level project initialization. Skip if already completed for auth feature.

- [ ] T001 Confirm api/ project is initialized with correct runtime and framework per platform overlay (may already be done from 001-user-auth)
- [ ] T002 Confirm PostgreSQL database connection is configured and reachable
- [ ] T003 [P] Confirm schema migration tooling is set up per platform overlay
- [ ] T004 [P] Confirm environment variables (`DATABASE_URL`, `JWT_SECRET`, `PORT`) are configured

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Contacts table, indexes, and shared validation utilities that ALL user stories
depend on. No user story work may begin until this phase is complete.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 Define `contacts` table schema per `data-model.md` Contact entity: `id` (UUID PK auto-generated), `user_id` (UUID NOT NULL FK → users.id cascade delete), `name` (NOT NULL max 255), `company` (nullable max 255), `role` (nullable max 255), `where_met` (nullable max 500), `date_met` (nullable DATE), `follow_up_date` (nullable DATE), `notes` (nullable TEXT), `priority` (nullable ENUM: hot/warm/cold), `created_at` (default now), `updated_at` (default now)
- [ ] T006 Add indexes per `data-model.md` Indexes section: index on `user_id`, composite index on `(user_id, created_at)` for sorted list queries
- [ ] T007 Run migrations and verify `contacts` table exists with correct columns, constraints, and indexes
- [ ] T008 [P] Confirm or implement JWT access token authentication middleware: extract `Authorization: Bearer <token>`, verify signature and expiry, attach `sub` to request context; return `401 UNAUTHORIZED` if missing/invalid/expired (reuse from 001-user-auth if available, per `contracts/auth-api.md`)
- [ ] T009 [P] Confirm or implement `X-Sync-Version: 1` header validation middleware: return `400 UNSUPPORTED_VERSION` on missing or unrecognized version (reuse from 001-user-auth if available)
- [ ] T010 [P] Implement `priority` enum validation utility: accepts `hot`, `warm`, `cold`, or `null`; rejects any other value with a descriptive error, per FR-011
- [ ] T011 [P] Implement date format validation utility: validates `YYYY-MM-DD` calendar date format for `date_met` and `follow_up_date` fields, per FR-012
- [ ] T012 [P] Implement PATCH null-semantics helper: at the request body deserialization layer, distinguish an absent key (no-op) from an explicit `null` (clear the field), per `research.md` Decision 3 and FR-015

**Checkpoint**: Foundation ready — user story implementation can now begin.

---

## Phase 3: User Story 1 — Contact Creation (Priority: P1) 🎯 MVP

**Goal**: Authenticated user creates a contact with only name (or with all fields). Missing
name returns `400 VALIDATION_ERROR`. Successful creation returns the full Contact Object.

**Independent Test**: `quickstart.md` Scenario 1 (Steps 1a, 1b, 1c)

### Implementation for User Story 1

- [ ] T013 [P] [US1] Implement `POST /api/contacts` route wired to auth middleware (T008) and version middleware (T009), per `contracts/contacts-api.md`
- [ ] T014 [US1] Implement name validation: non-empty, non-whitespace-only, max 255 chars; return `400 VALIDATION_ERROR` with `fields.name` error on failure, per FR-003 (depends on T013)
- [ ] T015 [US1] Implement optional field validation on creation: `priority` via T010, `date_met` and `follow_up_date` via T011; return `400 VALIDATION_ERROR` with field-level errors on failure, per FR-011, FR-012 (depends on T013)
- [ ] T016 [US1] Implement contact creation: derive `user_id` from JWT `sub` claim, insert `contacts` row with all validated fields (`priority` and nullable dates default to `null` when absent), return `201` with full Contact Object per `contracts/contacts-api.md` (depends on T014, T015)

**Checkpoint**: `POST /api/contacts` fully functional. Run `quickstart.md` Scenario 1 before proceeding.

---

## Phase 4: User Story 2 — Contact Retrieval (Priority: P2)

**Goal**: Authenticated user lists all contacts with offset pagination, or fetches a single
contact by ID. Non-existent or wrong-user contact returns `404 NOT_FOUND`.

**Independent Test**: `quickstart.md` Scenario 2 (Steps 2a, 2b, 2c) — requires at least one contact from US1 or pre-seeded data

### Implementation for User Story 2

- [ ] T017 [P] [US2] Implement `GET /api/contacts` route wired to auth middleware (T008) and version middleware (T009), per `contracts/contacts-api.md`
- [ ] T018 [US2] Implement pagination parameter parsing: `page` (integer ≥ 1, default `1`) and `page_size` (integer 1–100, default `20`); return `400 VALIDATION_ERROR` on invalid values, per FR-004 and `research.md` Decision 2 (depends on T017)
- [ ] T019 [US2] Implement contact list query: `SELECT … WHERE user_id = JWT.sub ORDER BY created_at DESC LIMIT page_size OFFSET (page-1)*page_size`; also query total count; return `{ data: [...], pagination: { page, page_size, total, total_pages } }` per `data-model.md` Pagination Model (depends on T018)
- [ ] T020 [P] [US2] Implement `GET /api/contacts/:id` route wired to auth middleware (T008) and version middleware (T009), per `contracts/contacts-api.md`
- [ ] T021 [US2] Implement single contact lookup: query `WHERE id = :id AND user_id = JWT.sub`; return `200` with Contact Object if found; return `404 NOT_FOUND` if row absent or belongs to a different user (prevents IDOR), per FR-005, FR-006, `research.md` Decision 4 (depends on T020)

**Checkpoint**: `GET /api/contacts` and `GET /api/contacts/:id` fully functional. Run `quickstart.md` Scenario 2 before proceeding.

---

## Phase 5: User Story 3 — Contact Update (Priority: P3)

**Goal**: Authenticated user partially updates any contact field. Only supplied fields
change; all others remain unchanged. Explicit `null` for a nullable field clears it.
Non-existent contact returns `404 NOT_FOUND`.

**Independent Test**: `quickstart.md` Scenario 3 (Steps 3a, 3b)

### Implementation for User Story 3

- [ ] T022 [P] [US3] Implement `PATCH /api/contacts/:id` route wired to auth middleware (T008) and version middleware (T009), per `contracts/contacts-api.md`
- [ ] T023 [US3] Implement contact existence check: query `WHERE id = :id AND user_id = JWT.sub`; return `404 NOT_FOUND` if not found, per FR-008 (depends on T022)
- [ ] T024 [US3] Implement partial field validation: for each key present in the PATCH body, validate — `name` (non-empty), `priority` via T010, date fields via T011; return `400 VALIDATION_ERROR` on failure (depends on T023)
- [ ] T025 [US3] Implement PATCH update: apply only keys present in body using null-semantics helper (T012) — explicit `null` clears nullable field, absent key is a no-op; set `updated_at = now()`; return `200` with the full updated Contact Object (all fields, including unchanged ones), per FR-007, FR-015 (depends on T024)

**Checkpoint**: `PATCH /api/contacts/:id` fully functional. Run `quickstart.md` Scenario 3 before proceeding.

---

## Phase 6: User Story 4 — Contact Deletion (Priority: P4)

**Goal**: Authenticated user permanently deletes a contact (hard-delete). Contact is
immediately absent from all retrieval endpoints. Non-existent contact returns `404 NOT_FOUND`.

**Independent Test**: `quickstart.md` Scenario 4 (Steps 4a, 4b, 4c)

### Implementation for User Story 4

- [ ] T026 [P] [US4] Implement `DELETE /api/contacts/:id` route wired to auth middleware (T008) and version middleware (T009), per `contracts/contacts-api.md`
- [ ] T027 [US4] Implement contact existence check for delete: query `WHERE id = :id AND user_id = JWT.sub`; return `404 NOT_FOUND` if not found, per FR-010 (depends on T026)
- [ ] T028 [US4] Implement hard-delete: `DELETE FROM contacts WHERE id = :id AND user_id = JWT.sub`; return `204 No Content` with no body, per FR-009 and `research.md` Decision 1 (depends on T027)

**Checkpoint**: `DELETE /api/contacts/:id` fully functional. Run `quickstart.md` Scenario 4 (including Steps 4b and 4c to confirm permanent deletion) before proceeding.

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: End-to-end validation and contract conformance across all four stories.

- [ ] T029 [P] Verify all error response shapes match the Error Code Reference table in `contracts/contacts-api.md` (`UNSUPPORTED_VERSION`, `VALIDATION_ERROR`, `UNAUTHORIZED`, `NOT_FOUND`)
- [ ] T030 [P] Confirm `GET /api/contacts` returns `{ data: [], pagination: { page: 1, page_size: 20, total: 0, total_pages: 0 } }` when the user has no contacts (spec.md edge case)
- [ ] T031 [P] Confirm requesting a `page` beyond `total_pages` returns `{ data: [] }` with valid pagination metadata (not 404), per `data-model.md` Pagination Model
- [ ] T032 Run all validation scenarios in `quickstart.md` end-to-end in sequence (Scenarios 1 → 5)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — confirm or complete immediately
- **Foundational (Phase 2)**: Depends on Phase 1 + 001-user-auth implemented — **BLOCKS all user stories**
- **US1 (Phase 3)**: Depends on Phase 2 completion
- **US2 (Phase 4)**: Depends on Phase 2 completion — can run in parallel with US1 after Phase 2; requires at least one contact for quickstart validation
- **US3 (Phase 5)**: Depends on Phase 2 completion — can run in parallel with US1/US2 after Phase 2
- **US4 (Phase 6)**: Depends on Phase 2 completion — can run in parallel with US1/US2/US3 after Phase 2
- **Polish (Phase N)**: Depends on all desired user stories being complete

### Within Each User Story

- Route handler first (Txx3), then validation (Txx4/Txx5), then persistence/response (Txx6)
- T014 and T015 within US1 are independent of each other (parallel)
- T017 and T020 within US2 are independent (different routes)

### Parallel Opportunities

- Phase 2: T008, T009, T010, T011, T012 are all independent utilities — run in parallel
- US1 and US2: T013 (POST) and T017/T020 (GET routes) are independent — implement in parallel if staffed
- US3 and US4: T022 (PATCH) and T026 (DELETE) are independent — implement in parallel
- All Phase N tasks are independent of each other

---

## Parallel Execution Example: Phase 2 (Foundational)

```
After T007 (migrations verified), launch in parallel:
  Task: "Confirm/implement JWT auth middleware (T008)"
  Task: "Confirm/implement X-Sync-Version middleware (T009)"
  Task: "Implement priority enum validator (T010)"
  Task: "Implement date format validator (T011)"
  Task: "Implement PATCH null-semantics helper (T012)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (confirm prereqs)
2. Complete Phase 2: Foundational (CRITICAL — blocks everything)
3. Complete Phase 3: User Story 1 (contact creation)
4. **STOP and VALIDATE**: Run `quickstart.md` Scenario 1
5. Demo: authenticated user can capture a contact

### Incremental Delivery

1. Setup + Foundational → infrastructure ready
2. US1 complete → `POST /api/contacts` works → validate independently
3. US2 complete → `GET /api/contacts` + `GET /api/contacts/:id` work → validate independently
4. US3 complete → `PATCH /api/contacts/:id` works → validate independently
5. US4 complete → `DELETE /api/contacts/:id` works → validate end-to-end
6. Polish → all contracts verified, edge cases confirmed

---

## Notes

- [P] tasks = independent — no incomplete predecessors, different logical components
- [Story] label maps each task to its user story for traceability back to `spec.md`
- File paths are defined per platform overlay, not in this shared spec
- T008 and T009 reference middleware already defined in 001-user-auth — reuse if the auth feature is already implemented on the target platform
- Verify each checkpoint before moving to the next phase
- Hard-delete means no `deleted_at` column — do not add one

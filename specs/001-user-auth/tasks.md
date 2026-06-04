---
description: "Task list for User Authentication feature implementation"
---

# Tasks: User Authentication

**Input**: Design documents from `specs/001-user-auth/`

**Prerequisites**: plan.md ✅ | spec.md ✅ | research.md ✅ | data-model.md ✅ | contracts/auth-api.md ✅

**Platform note**: Source code structure and file paths are platform-specific and defined in
each platform overlay (`networking-web`, `networking-ios`). Tasks below reference logical
components and design documents; the implementing platform maps these to concrete file paths.

**Tests**: Not requested. Test tasks are excluded.

**Organization**: Tasks are grouped by user story. Each story is independently implementable
and testable using the scenarios in `quickstart.md`.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (no dependency on an incomplete task)
- **[Story]**: User story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup

**Purpose**: Platform-level project initialization and shared infrastructure scaffolding.

- [ ] T001 Initialize the API project in the `api/` directory using the platform overlay's chosen runtime and framework
- [ ] T002 Configure PostgreSQL database connection via environment variable (`DATABASE_URL`)
- [ ] T003 [P] Set up schema migration tooling per platform overlay
- [ ] T004 [P] Configure environment variable management (`DATABASE_URL`, `JWT_SECRET`, `PORT`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Database schema, shared token utilities, and version middleware that ALL user
stories depend on. No user story work may begin until this phase is complete.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 Define `users` table schema per `data-model.md` User entity: fields `id` (UUID PK), `email` (unique, not null, max 254), `password_hash` (not null), `created_at` (default now)
- [ ] T006 Define `refresh_tokens` table schema per `data-model.md` RefreshToken entity: fields `id` (UUID PK), `user_id` (FK → users.id, cascade delete), `token` (unique, not null), `expires_at` (not null), `invalidated_at` (nullable), `created_at` (default now)
- [ ] T007 Add indexes per `data-model.md` Indexes section: unique on `users.email`, unique on `refresh_tokens.token`, index on `refresh_tokens.user_id`, index on `refresh_tokens.expires_at`
- [ ] T008 Run migrations and verify both tables exist with correct columns and indexes
- [ ] T009 [P] Implement JWT access token utilities: sign (produces HS256 JWT with claims `sub`, `email`, `iat`, `exp = iat+3600`) and verify (checks signature and `exp`), per `data-model.md` Access Token section
- [ ] T010 [P] Implement password utilities: hash input using adaptive salted algorithm (cost ≥ 12 if bcrypt), compare plaintext against stored hash, per FR-015 — no plaintext ever logged or stored
- [ ] T011 [P] Implement `X-Sync-Version` header validation middleware: reject requests with missing or unrecognized version with `400 UNSUPPORTED_VERSION`, per `contracts/auth-api.md` Common Rules
- [ ] T012 [P] Generate cryptographically random token utility (for refresh token issuance)

**Checkpoint**: Foundation ready — user story implementation can now begin.

---

## Phase 3: User Story 1 — Account Registration (Priority: P1) 🎯 MVP

**Goal**: New user submits email + password and receives access token + refresh token.
Duplicate email returns `409 EMAIL_ALREADY_EXISTS`.

**Independent Test**: quickstart.md Validation Scenario 1 (Steps 1a, 1b, 1c)

### Implementation for User Story 1

- [ ] T013 [P] [US1] Implement `POST /api/auth/register` route, wired to version middleware (T011), per `contracts/auth-api.md`
- [ ] T014 [US1] Implement email normalization (lowercase) and RFC 5321 format validation; return `400 VALIDATION_ERROR` with field-level errors on failure, per `contracts/auth-api.md` and `data-model.md` validation rules (depends on T013)
- [ ] T015 [US1] Implement email uniqueness check: query `users` table by normalized email; return `409 EMAIL_ALREADY_EXISTS` if found, per FR-002 (depends on T014)
- [ ] T016 [US1] Implement password length validation (≥ 8 chars); return `400 VALIDATION_ERROR` on failure, per FR-014 and edge cases in spec.md (depends on T013)
- [ ] T017 [US1] Implement registration completion: hash password (T010), insert `users` row, generate refresh token (T012), insert `refresh_tokens` row with `expires_at = now() + 30 days`, sign access token (T009), return `201` with token payload per `contracts/auth-api.md` (depends on T015, T016)

**Checkpoint**: `POST /api/auth/register` fully functional. Run quickstart.md Scenario 1 before proceeding.

---

## Phase 4: User Story 2 — Login (Priority: P2)

**Goal**: Registered user submits credentials and receives a fresh token pair. Wrong
credentials return `401 INVALID_CREDENTIALS`.

**Independent Test**: quickstart.md Validation Scenario 2 (Steps 2a, 2b)

### Implementation for User Story 2

- [ ] T018 [P] [US2] Implement `POST /api/auth/login` route, wired to version middleware (T011), per `contracts/auth-api.md`
- [ ] T019 [US2] Implement credential validation: require both `email` and `password` fields; return `400 VALIDATION_ERROR` if either is missing (depends on T018)
- [ ] T020 [US2] Implement credential verification: normalize email, look up user in `users` table, compare password hash using T010; return generic `401 INVALID_CREDENTIALS` for any mismatch (wrong password OR unrecognized email), per research.md Decision 4 (depends on T019)
- [ ] T021 [US2] Implement login token issuance: generate refresh token (T012), insert `refresh_tokens` row with `expires_at = now() + 30 days`, sign access token (T009), return `200` with token payload per `contracts/auth-api.md` (depends on T020)

**Checkpoint**: `POST /api/auth/login` fully functional. Run quickstart.md Scenario 2 before proceeding.

---

## Phase 5: User Story 3 — Session Maintenance and Logout (Priority: P3)

**Goal**: Valid access token authorizes requests. Expired access token + valid refresh token
yields new token pair (rotation). Logout invalidates the refresh token server-side;
subsequent use returns `401`.

**Independent Test**: quickstart.md Validation Scenario 3 (Steps 3a–3e)

### Implementation for User Story 3

- [ ] T022 [P] [US3] Implement access token authentication middleware: extract `Authorization: Bearer <token>` header, verify JWT using T009, attach decoded claims to request context; return `401 UNAUTHORIZED` if missing/invalid/expired, per `contracts/auth-api.md` Authenticated Request Error section
- [ ] T023 [P] [US3] Implement `POST /api/auth/refresh` route, wired to version middleware (T011), per `contracts/auth-api.md`
- [ ] T024 [US3] Implement refresh token exchange: look up `token` in `refresh_tokens` table, reject if `invalidated_at IS NOT NULL` or `expires_at` is past with `401 INVALID_REFRESH_TOKEN`; on success: set `invalidated_at = now()` on old row, insert new `refresh_tokens` row, sign new access token (T009), return `200` with new token pair per `contracts/auth-api.md` (depends on T023, research.md Decision 1)
- [ ] T025 [P] [US3] Implement `POST /api/auth/logout` route, protected by authentication middleware (T022), wired to version middleware (T011), per `contracts/auth-api.md`
- [ ] T026 [US3] Implement logout: look up `refresh_token` body field in `refresh_tokens` table, set `invalidated_at = now()`, return `204 No Content`, per FR-009, FR-010 (depends on T025)

**Checkpoint**: All US3 acceptance scenarios functional. Run quickstart.md Scenario 3 (all steps) before proceeding.

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Validation that all stories compose correctly and all contracts are honoured.

- [ ] T027 [P] Verify all error response shapes match the Error Code Reference table in `contracts/auth-api.md` (6 error codes across all 4 endpoints)
- [ ] T028 [P] Verify `X-Sync-Version` header enforcement on all four endpoints by running quickstart.md Scenario 4
- [ ] T029 [P] Confirm no plaintext password appears in any API response body or server log output, per FR-015
- [ ] T030 Run all validation scenarios in `quickstart.md` end-to-end in sequence (Scenarios 1 → 4)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — **BLOCKS all user stories**
- **US1 (Phase 3)**: Depends on Phase 2 completion (T005–T012)
- **US2 (Phase 4)**: Depends on Phase 2 completion — can run in parallel with US1 after Phase 2
- **US3 (Phase 5)**: Depends on Phase 2 completion — can run in parallel with US1/US2 after Phase 2
- **Polish (Phase N)**: Depends on all desired user stories being complete

### Within Each User Story

- Setup route handler (T013, T018, T023, T025) before logic tasks
- Validation before persistence before token issuance
- Authentication middleware (T022) before logout route (T025)

### Parallel Opportunities

- Within Phase 2: T009, T010, T011, T012 are all independent (different utilities)
- US1 and US2 are independent after Phase 2 — different endpoints, no shared mutable state
- T022, T023, T025 within US3 are independent (different routes/middleware)
- All Phase N tasks are independent of each other

---

## Parallel Execution Example: Phase 2 (Foundational)

```
Launch in parallel after T008 (migrations verified):
  Task: "Implement JWT sign/verify utilities (T009)"
  Task: "Implement password hash/compare utilities (T010)"
  Task: "Implement X-Sync-Version middleware (T011)"
  Task: "Implement random token generator (T012)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks everything)
3. Complete Phase 3: User Story 1 (registration)
4. **STOP and VALIDATE**: Run quickstart.md Scenario 1
5. Demo: new user can register and receive usable tokens

### Incremental Delivery

1. Setup + Foundational → infrastructure ready
2. US1 complete → `POST /api/auth/register` works → validate independently
3. US2 complete → `POST /api/auth/login` works → validate independently
4. US3 complete → refresh + logout work → validate independently
5. Polish → all contracts verified end-to-end

---

## Notes

- [P] tasks = independent — different logical components, no incomplete predecessors
- [Story] label maps each task to its user story for traceability back to spec.md
- File paths are defined per platform overlay, not in this shared spec
- Verify each checkpoint before moving to the next phase
- Stop at each checkpoint to validate the story independently before continuing

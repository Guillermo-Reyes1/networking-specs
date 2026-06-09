---
description: "Task list for Cross-Device Sync feature implementation"
---

# Tasks: Cross-Device Sync

**Input**: Design documents from `specs/004-cross-device-sync/`

**Prerequisites**: plan.md ✅ | spec.md ✅ | research.md ✅ | data-model.md ✅ | contracts/websocket-events.md ✅ | contracts/sync-rest-extensions.md ✅

**Dependency**: This feature depends on `001-user-auth`, `002-contact-management`, and
`003-interaction-log` being fully implemented on the target platform. JWT middleware,
`X-Sync-Version` middleware, and all contact and interaction REST endpoints are reused
here. Complete those features before starting this one.

**Platform note**: Source code structure and file paths are platform-specific and defined
in each platform overlay. Tasks reference logical components and design documents.

**Tests**: Not requested. Test tasks are excluded.

**Organization**: Tasks are grouped by user story. Each story is independently
implementable and testable once the foundation is in place.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (no dependency on an incomplete task)
- **[Story]**: User story this task belongs to (US1–US6)

---

## Phase 1: Setup

**Purpose**: Confirm all upstream features are in place. Skip any item already completed.

- [ ] T001 Confirm `001-user-auth` is fully implemented on the target platform: JWT access tokens issued, auth middleware reusable
- [ ] T002 [P] Confirm `002-contact-management` is fully implemented: all contact REST endpoints (POST, GET list, GET single, PATCH, DELETE) working
- [ ] T003 [P] Confirm `003-interaction-log` is fully implemented: all interaction REST endpoints working
- [ ] T004 [P] Confirm Socket.io (or equivalent WebSocket library per platform overlay) is available as a dependency

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Schema migration, core sync infrastructure, and shared utilities that every
user story depends on. No user story work may begin until this phase is complete.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 Add `updated_at` column to the `interactions` table per `data-model.md` Interaction amendment: NOT NULL Timestamp UTC, default now(), automatically updated on every PATCH by ORM lifecycle hook or database trigger (same pattern as `contacts.updated_at`)
- [ ] T006 Run migration and verify `interactions` table has the `updated_at` column with correct default and auto-update behavior
- [ ] T007 Update all Interaction API response serializers to include `updated_at` in the response body, positioned after `created_at`, per `contracts/sync-rest-extensions.md` Updated Interaction Object shape — affects POST 201, GET 200 array items, and PATCH 200 responses
- [ ] T008 [P] Initialize the Socket.io server instance co-located with the existing REST API server; configure it to share the same HTTP listener
- [ ] T009 [P] Implement device ID extraction: on every contact and interaction mutation request, read the originating device UUID from the request (e.g., a dedicated request header); validate it is a valid UUID; make it available to the broadcast step — this value is embedded in every sync event envelope per `contracts/websocket-events.md` Shared Envelope `device_id` field
- [ ] T010 [P] Implement sync event emitter utility: given `(user_id, event_name, device_id, data)`, emit a Socket.io event to the room keyed by `user_id` (a UUID string) with the envelope `{ "version": "1", "event": event_name, "device_id": device_id, "data": data }` per `contracts/websocket-events.md` Shared Envelope and `research.md` Decision 2 and 3

**Checkpoint**: Foundation ready — schema migration applied, Socket.io running, emitter utility available. User story implementation can now begin.

---

## Phase 3: User Story 1 — WebSocket Authentication (Priority: P1) 🎯 MVP

**Goal**: Clients with a valid JWT are admitted to the Socket.io server and placed into
their user room. Clients without a valid JWT are rejected at the handshake.

**Independent Test**: `quickstart.md` Scenarios 1–3 (accepted with valid JWT, rejected with no token, rejected with invalid/expired token)

- [ ] T011 [US1] Implement Socket.io connection handler: on connection attempt, extract the JWT from the Socket.io handshake auth object (`handshake.auth.token`) per `contracts/websocket-events.md` Establishing a Connection
- [ ] T012 [US1] Implement JWT validation inside the connection handler: verify the token using the same JWT verification logic as the REST auth middleware (T001); reject the connection if the token is missing, expired, or invalid per FR-006 and `contracts/websocket-events.md` Connection Outcomes (depends on T011)
- [ ] T013 [US1] Implement user room assignment: on successful JWT validation, join the connection to a Socket.io room identified by the JWT `sub` claim (user UUID); per `research.md` Decision 2 (depends on T012)
- [ ] T014 [US1] Implement mid-session expiry handling: if the JWT expires while the connection is active, the server closes the socket; per `contracts/websocket-events.md` Disconnection (depends on T013)

**Checkpoint**: `quickstart.md` Scenarios 1–3 pass. WebSocket authentication is independently functional.

---

## Phase 4: User Story 2 — Server Broadcasts Mutation Events (Priority: P2)

**Goal**: Every successful contact and interaction REST mutation triggers a correctly
shaped sync event broadcast to all authenticated connections in the user's room.

**Independent Test**: `quickstart.md` Scenarios 4–8 (contact:created, contact:updated, contact:deleted, interaction events, no event on failed mutation) — requires US1 complete so connections can be established

- [ ] T015 [P] [US2] Add broadcast after POST /api/contacts succeeds: after database commit (2xx response), invoke T010 emitter with event `"contact:created"` and the full Contact Object, including `updated_at`, as `data`; per FR-001 and `contracts/websocket-events.md` contact:created
- [ ] T016 [P] [US2] Add broadcast after PATCH /api/contacts/:id succeeds: after database commit, invoke T010 emitter with event `"contact:updated"` and the full updated Contact Object as `data`; per FR-002 and `contracts/websocket-events.md` contact:updated
- [ ] T017 [P] [US2] Add broadcast after DELETE /api/contacts/:id succeeds: after database commit, invoke T010 emitter with event `"contact:deleted"` and tombstone `{ "id": "<uuid>" }` as `data`; per FR-003 and `contracts/websocket-events.md` contact:deleted
- [ ] T018 [P] [US2] Add broadcast after POST /api/contacts/:contact_id/interactions succeeds: invoke T010 emitter with event `"interaction:created"` and full Interaction Object (now including `updated_at` from T007) as `data`; per FR-004 and `contracts/websocket-events.md` interaction:created
- [ ] T019 [P] [US2] Add broadcast after PATCH /api/contacts/:contact_id/interactions/:id succeeds: invoke T010 emitter with event `"interaction:updated"` and full updated Interaction Object as `data`; per FR-004 and `contracts/websocket-events.md` interaction:updated
- [ ] T020 [P] [US2] Add broadcast after DELETE /api/contacts/:contact_id/interactions/:id succeeds: invoke T010 emitter with event `"interaction:deleted"` and tombstone `{ "id": "<uuid>" }` as `data`; per FR-004 and `contracts/websocket-events.md` interaction:deleted
- [ ] T021 [US2] Verify broadcast gates: confirm that T010 is only invoked inside the 2xx success path of each mutation handler — routes returning 4xx or 5xx MUST NOT emit an event; per FR-005 (depends on T015–T020)

**Checkpoint**: `quickstart.md` Scenarios 4–8 pass. All six mutation types broadcast correct events.

---

## Phase 5: User Story 3 — Real-Time Update Received by Client (Priority: P2)

**Goal**: A connected client applies incoming sync events to its local state without user
action. An event carrying an unrecognized `version` triggers a user warning and is
discarded rather than applied.

**Independent Test**: `quickstart.md` Scenarios 4–8 observed from the receiving client, plus Scenario 15 (unrecognized version) — requires US1 (connection) and US2 (server broadcasts)

- [ ] T022 [US3] Implement client-side Socket.io event listener registration for all six event names: `contact:created`, `contact:updated`, `contact:deleted`, `interaction:created`, `interaction:updated`, `interaction:deleted`; per `contracts/websocket-events.md` Event Names
- [ ] T023 [US3] Implement version check as the first step in every event handler: read the `version` field from the event envelope; if not `"1"` (or any value the client explicitly supports), display a user-visible warning and return immediately — MUST NOT mutate local state; per FR-009, SC-006, and `contracts/websocket-events.md` Unrecognized Version Handling (depends on T022)
- [ ] T024 [P] [US3] Implement contact event handlers (after version check passes): on `contact:created` or `contact:updated`, merge the full Contact Object from `data` into local contact state; on `contact:deleted`, remove the contact identified by `data.id` from local state; per `contracts/websocket-events.md` contact events and FR-006 (depends on T023)
- [ ] T025 [P] [US3] Implement interaction event handlers (after version check passes): on `interaction:created` or `interaction:updated`, merge the full Interaction Object from `data` into local interaction state for the relevant contact; on `interaction:deleted`, remove the interaction identified by `data.id`; per `contracts/websocket-events.md` interaction events (depends on T023)
- [ ] T026 [US3] Implement optional self-origin skip: compare the event envelope `device_id` against the local device UUID (from T009); if they match, the client MAY skip re-processing the event (the local state is already current from the REST response); per `research.md` Decision 3 and `data-model.md` Sync Event (depends on T024, T025)

**Checkpoint**: `quickstart.md` Scenarios 4–8 pass from the receiving client's perspective. Scenario 15 (unrecognized version warning) passes.

---

## Phase 6: User Story 4 — Offline Queue and Reconnect Replay (Priority: P3)

**Goal**: Mutations made while offline are enqueued locally and survive app restarts. On
reconnect, all queued entries are submitted to the REST API in FIFO order without user
action. Successful submissions remove the entry from the queue.

**Independent Test**: `quickstart.md` Scenarios 9–10 (queue persists after restart, replay on reconnect in order)

- [ ] T027 [US4] Implement the offline queue persistent store: the store must satisfy the Offline Queue Entry schema from `data-model.md` — each entry has `id` (UUID), `entity_type`, `mutation_type`, `entity_id`, `payload`, `entity_updated_at` (nullable), and `enqueued_at`; the store MUST survive app restarts; platform overlay defines the storage engine
- [ ] T028 [US4] Implement connectivity detection: monitor network state; when the device goes offline, route all contact and interaction mutation attempts to T027 instead of the REST API; per FR-012 (depends on T027)
- [ ] T029 [US4] Implement queue entry creation: when a mutation is intercepted while offline, create a queue entry with `entity_updated_at` set to the entity's current `updated_at` timestamp (null for `created` entries where no entity exists yet), `enqueued_at` set to current time, and `payload` set to the full request body; per `research.md` Decision 8 and FR-012 (depends on T028)
- [ ] T030 [US4] Implement reconnect trigger: when connectivity is restored, automatically initiate queue replay without requiring any user action; per FR-013 (depends on T028)
- [ ] T031 [US4] Implement FIFO queue replay: iterate queue entries in ascending `enqueued_at` order; for each entry, construct and submit the correct REST request (derived from `entity_type` and `mutation_type`) with the stored `payload` and `X-Sync-Version: 1` header; per FR-013 and `research.md` Decision 8 (depends on T030)
- [ ] T032 [US4] Implement queue entry removal on success: when a REST submission returns 2xx, remove the corresponding queue entry from the store; per `data-model.md` Offline Queue Entry Lifecycle step 3 (depends on T031)

**Checkpoint**: `quickstart.md` Scenarios 9–10 pass. Queue persists across restart; entries submitted and removed on reconnect.

---

## Phase 7: User Story 5 — Conflict Resolution (Priority: P3)

**Goal**: PATCH and DELETE submissions during replay include `If-Match`. The server
returns 409 with `server_version` when the entity has changed. The user explicitly
chooses local or server version; no silent overwrite ever occurs.

**Independent Test**: `quickstart.md` Scenarios 11–14 (conflict detected, server wins, local wins, per-entry isolation) — requires US4 complete

- [ ] T033 [US5] Implement `If-Match` header injection on PATCH and DELETE submissions during replay: read `entity_updated_at` from the queue entry; add `If-Match: "<entity_updated_at>"` to the request (RFC 7232 quoted format); POST (create) entries MUST NOT include `If-Match`; per `contracts/sync-rest-extensions.md` New Request Header and `research.md` Decision 4 (depends on T031)
- [ ] T034 [US5] Implement server-side `If-Match` check on PATCH /api/contacts/:id: if `If-Match` header is present, compare its value against the contact's current `updated_at`; if they differ, return 409 with body `{ "error": "CONFLICT", "message": "...", "server_version": <full Contact Object> }`; if `If-Match` is absent, apply the mutation unconditionally (no conflict check); per `contracts/sync-rest-extensions.md` 409 response shape and `research.md` Decision 4 and 5
- [ ] T035 [P] [US5] Implement server-side `If-Match` check on DELETE /api/contacts/:id: same logic as T034 — if present and mismatched, return 409 with `server_version` containing the current Contact Object; if absent, apply unconditionally; per `contracts/sync-rest-extensions.md` and `research.md` Decision 4
- [ ] T036 [P] [US5] Implement server-side `If-Match` check on PATCH /api/contacts/:contact_id/interactions/:id: compare `If-Match` against the interaction's current `updated_at`; if mismatched, return 409 with `server_version` containing the full Interaction Object; absent = unconditional write; per `contracts/sync-rest-extensions.md`
- [ ] T037 [P] [US5] Implement server-side `If-Match` check on DELETE /api/contacts/:contact_id/interactions/:id: same as T036 but for interaction deletes; per `contracts/sync-rest-extensions.md`
- [ ] T038 [US5] Implement client-side conflict handler: when a 409 response is received during replay, hold the queue entry (do not remove it); extract `server_version` from the response body; present the user with both versions side-by-side; require an explicit choice before proceeding; the handler MUST NOT auto-resolve or timeout to a default; per FR-015, SC-004, and `contracts/sync-rest-extensions.md` Conflict Resolution Flows (depends on T033)
- [ ] T039 [US5] Implement "server wins" resolution: when the user selects the server version, discard (remove) the queue entry from the store; update local state with `server_version` from the 409 body; no additional HTTP request is made; per FR-016 and `contracts/sync-rest-extensions.md` User Chooses "Server Wins" (depends on T038)
- [ ] T040 [US5] Implement "local wins" resolution: when the user selects the local version, resubmit the original PATCH or DELETE request from the queue entry WITHOUT the `If-Match` header (unconditional write); on 2xx, remove the queue entry; per FR-016 and `contracts/sync-rest-extensions.md` User Chooses "Local Wins" (depends on T038)
- [ ] T041 [US5] Verify per-entry conflict isolation: a 409 on one queue entry MUST NOT block replay of subsequent entries; after the user resolves each conflict (T039 or T040), replay continues with the remaining entries; per FR-017 and `spec.md` User Story 5 AS 5 (depends on T039, T040)

**Checkpoint**: `quickstart.md` Scenarios 11–14 pass. Conflicts surface correctly; both resolution paths work; subsequent entries replay after resolution.

---

## Phase 8: User Story 6 — Retry and Error Surfacing (Priority: P4)

**Goal**: Non-conflict failures (network errors, 5xx) during replay are retried with
exponential backoff at 1 s, 2 s, 4 s, 8 s. After four consecutive failures the error is
shown to the user and automatic retrying stops.

**Independent Test**: `quickstart.md` Scenarios 16–17 (correct retry delays, error surfaced after fourth failure)

- [ ] T042 [US6] Implement retry counter per queue entry: track consecutive failed non-conflict attempts; increment on each failure; reset to zero on success (2xx) or when the entry transitions to conflict-pending (409); per FR-018
- [ ] T043 [US6] Implement exponential backoff scheduler: after attempt N fails (1-indexed), schedule attempt N+1 after `2^(N-1)` seconds — giving delays 1 s, 2 s, 4 s, 8 s for attempts 1–4; per FR-018 and `spec.md` Decisions (retry sequence: 1 s, 2 s, 4 s, 8 s) (depends on T042)
- [ ] T044 [US6] Implement error surfacing after four failures: if the 4th retry also fails, display a user-visible error message identifying the failed mutation; cease all automatic retrying for that queue entry; the entry remains in the queue until the user takes action; per FR-019 and SC-005 (depends on T043)

**Checkpoint**: `quickstart.md` Scenarios 16–17 pass. Retry timing is correct; error surface after the fourth failure is confirmed.

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Contract conformance, end-to-end validation, and edge case verification.

- [ ] T045 [P] Verify all six broadcast event envelopes match `contracts/websocket-events.md` examples exactly: `version: "1"`, correct `event` name, `device_id` present as a UUID, correct `data` shape (full object for creates/updates, tombstone `{ "id": "..." }` for deletes)
- [ ] T046 [P] Verify all four 409 response bodies match `contracts/sync-rest-extensions.md` exactly: `error: "CONFLICT"`, human-readable `message`, `server_version` contains full entity with correct fields including `updated_at`
- [ ] T047 [P] Confirm POST (create) replay entries never send `If-Match`: inspect queue replay path and verify that create-type entries produce requests without the `If-Match` header
- [ ] T048 [P] Confirm all Interaction API responses now include `updated_at` per `contracts/sync-rest-extensions.md` Updated Interaction Object shape — verify against the three affected response shapes (POST 201, GET 200 items, PATCH 200)
- [ ] T049 [P] Confirm Socket.io connections from a second user's JWT cannot receive another user's events: verify room isolation by connecting two accounts and confirming cross-user event leakage does not occur
- [ ] T050 Run all 17 validation scenarios in `quickstart.md` end-to-end in sequence; confirm every scenario produces its expected outcome before marking the feature complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — confirm prereqs immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — **BLOCKS all user stories**
- **US1 (Phase 3)**: Depends on Phase 2
- **US2 (Phase 4)**: Depends on Phase 2 and US1 (clients must connect to receive events)
- **US3 (Phase 5)**: Depends on US1 (connection) and US2 (server broadcasts to receive)
- **US4 (Phase 6)**: Depends on Phase 2; US1 needed for replay to trigger events visible on other clients
- **US5 (Phase 7)**: Depends on US4 (offline replay path must be implemented first)
- **US6 (Phase 8)**: Depends on US4 (retry wraps the replay submission loop)
- **Polish (Phase N)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Can start immediately after Phase 2 — no story dependencies
- **US2 (P2)**: Can start after Phase 2; US1 must be complete for end-to-end testing
- **US3 (P2)**: Depends on US1 + US2 — client can only receive what the server emits
- **US4 (P3)**: Can start after Phase 2 — queue logic is independent of broadcast
- **US5 (P3)**: Depends on US4 — conflict handling lives inside the replay path
- **US6 (P4)**: Depends on US4 — retry lives inside the replay submission loop

### Within Each User Story

- US2: T015–T020 (all broadcast additions) are fully parallel; T021 (verification) gates after all six
- US3: T024 (contact handlers) and T025 (interaction handlers) are parallel; both depend on T023 (version check)
- US5: T035, T036, T037 are parallel server-side additions; T038 (client conflict handler) gates before T039 and T040; T041 (isolation verification) gates after T039 and T040
- US4 and US5/US6 can be worked in parallel by separate developers once Phase 2 is done

### Parallel Opportunities

- Phase 2: T008, T009, T010 can run in parallel after T007 (schema verified)
- Phase 4 (US2): T015–T020 are all independent; launch together
- Phase 5 (US3): T024 and T025 are independent; launch together
- Phase 7 (US5): T035, T036, T037 (server-side checks) are independent; launch together
- Phase N: T045, T046, T047, T048, T049 are all independent; run together

---

## Parallel Execution Example: Phase 4 (US2 — Server Broadcasts)

```
After Phase 2 completes and US1 checkpoint passes, launch all broadcast tasks together:

  Task T015: Add broadcast after POST /api/contacts succeeds (contact:created)
  Task T016: Add broadcast after PATCH /api/contacts/:id succeeds (contact:updated)
  Task T017: Add broadcast after DELETE /api/contacts/:id succeeds (contact:deleted)
  Task T018: Add broadcast after POST interaction succeeds (interaction:created)
  Task T019: Add broadcast after PATCH interaction succeeds (interaction:updated)
  Task T020: Add broadcast after DELETE interaction succeeds (interaction:deleted)

Then run T021 to verify no broadcast fires on failed mutations.
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (confirm prereqs)
2. Complete Phase 2: Foundational (CRITICAL — blocks everything)
3. Complete Phase 3: User Story 1 (WebSocket authentication)
4. **STOP and VALIDATE**: `quickstart.md` Scenarios 1–3
5. Demo: WebSocket connections accepted with valid JWT, rejected without

### Incremental Delivery

1. Setup + Foundational → Socket.io running, schema migrated
2. US1 complete → connections accepted/rejected → validate (`quickstart.md` 1–3)
3. US2 complete → server broadcasts all six event types → validate (4–8)
4. US3 complete → client applies events, warns on version mismatch → validate (4–8 from receiver, 15)
5. US4 complete → offline queue persists + replays → validate (9–10)
6. US5 complete → conflicts detected and resolved → validate (11–14)
7. US6 complete → retry backoff + error surfacing → validate (16–17)
8. Polish → full `quickstart.md` run (all 17 scenarios)

### Parallel Team Strategy

With two developers after Phase 2:

- **Developer A**: US1 → US2 → US3 (server-side broadcast pipeline)
- **Developer B**: US4 → US5 → US6 (client-side offline + conflict pipeline)

Both pipelines are independently testable; they integrate at the checkpoint where Developer B's replay triggers Developer A's broadcasts on other connected clients.

---

## Notes

- [P] tasks = independent — different logical components, no incomplete predecessors
- [Story] label maps each task to its user story for traceability back to `spec.md`
- File paths are defined per platform overlay, not in this shared spec
- T005 and T006 are the only server-side schema changes — all other sync logic is application-layer
- `If-Match` (T033–T037) applies to PATCH and DELETE replay only; POST (create) entries MUST NOT send it
- T039 (server wins) requires no HTTP request — `server_version` is already in the 409 body
- T040 (local wins) resubmits the original payload without `If-Match` — the existing PATCH/DELETE handler already handles unconditional writes
- Hard-delete tombstones for sync events (`{ "id": "..." }`) are consistent with the hard-delete strategy from 002-contact-management
- Do not add `deleted_at` or event log tables — no server-side buffering per `research.md` Decision 7
- Verify each checkpoint before moving to the next phase

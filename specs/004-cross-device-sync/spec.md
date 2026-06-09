# Feature Specification: Cross-Device Sync

**Feature Branch**: `004-cross-device-sync`

**Created**: 2026-06-08

**Status**: Draft

**Input**: User description: "The server must push real-time change events to all connected clients whenever a contact or interaction is mutated via the REST API. The iOS client must queue mutations locally when offline and replay them on reconnect. When a conflict is detected between a queued offline mutation and the current server state, the user must be shown both versions and explicitly choose which to keep."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - WebSocket Authentication (Priority: P1)

A client establishes a real-time connection to the server to receive sync events. The server must verify the client's identity before admitting the connection; unidentified or unauthenticated clients must be turned away to prevent leaking other users' data.

**Why this priority**: Authentication is a gate that all other sync behavior depends on. A client that cannot connect receives no events and cannot replay queued mutations. Getting this right first means every subsequent scenario runs on a trusted foundation.

**Independent Test**: Can be fully tested by attempting WebSocket connections with a valid JWT, an expired JWT, and no JWT — verifying accepted and rejected outcomes independently of any data mutations.

**Acceptance Scenarios**:

1. **Given** a client with a valid JWT, **When** it initiates a WebSocket connection, **Then** the connection is accepted and the client begins receiving sync events.
2. **Given** a client with no JWT, **When** it initiates a WebSocket connection, **Then** the connection is rejected.
3. **Given** a client with an expired or invalid JWT, **When** it initiates a WebSocket connection, **Then** the connection is rejected.

---

### User Story 2 - Server Broadcasts Mutation Events (Priority: P2)

Whenever a user creates, updates, or deletes a contact or interaction through the REST API, the server immediately notifies all of that user's other connected clients. The receiving clients update their local view without requiring any user action.

**Why this priority**: Broadcasting is the core mechanism of cross-device sync. Without it, clients are permanently stale; the entire feature exists to solve this problem.

**Independent Test**: Can be fully tested in isolation: make a REST mutation, observe that connected clients receive the corresponding event with correct entity data — without requiring offline queue or conflict resolution logic.

**Acceptance Scenarios**:

1. **Given** a connected client and an authenticated session, **When** a contact is created via POST /api/contacts, **Then** the server broadcasts a `contact-created` event to all other connected clients for that user.
2. **Given** a connected client and an authenticated session, **When** a contact is updated via PATCH /api/contacts/:id, **Then** the server broadcasts a `contact-updated` event to all other connected clients for that user.
3. **Given** a connected client and an authenticated session, **When** a contact is deleted via DELETE /api/contacts/:id, **Then** the server broadcasts a `contact-deleted` event to all other connected clients for that user.
4. **Given** a connected client and an authenticated session, **When** an interaction is created, updated, or deleted via the interactions REST API, **Then** the server broadcasts the corresponding interaction event to all other connected clients for that user.
5. **Given** a mutation that fails (non-2xx response from the REST API), **When** the mutation is not persisted, **Then** the server does NOT broadcast an event for that mutation.

---

### User Story 3 - Real-Time Update Received by Client (Priority: P2)

A user working on the web client creates a contact. Within a few seconds, the same contact appears on their iOS client without them doing anything — no pull-to-refresh, no app restart.

**Why this priority**: This is the visible payoff of broadcasting. Story 2 defines the server side; this defines the client side. Both are P2 because they form one end-to-end flow.

**Independent Test**: Can be fully tested end-to-end: create a contact on one connected client and verify the other connected client reflects the change within the expected time window, with no user interaction.

**Acceptance Scenarios**:

1. **Given** two clients connected to the WebSocket server, **When** a contact is created on client A via the REST API, **Then** client B receives a `contact-created` event and reflects the new contact within a few seconds — without any user action on client B.
2. **Given** a client that receives a sync event with an unrecognized `X-Sync-Version` value, **When** the event arrives, **Then** the client warns the user rather than silently applying the event.

---

### User Story 4 - Offline Queue and Reconnect Replay (Priority: P3)

A user captures a new contact on their iOS device at a venue with no network coverage. The contact is saved locally and queued for sync. When the device regains connectivity, the queued mutations are automatically submitted to the server in the order they were made — no user action required.

**Why this priority**: Offline-first is a core product principle. Without it, any iOS mutation during poor connectivity is silently lost. P3 because it is a self-contained iOS concern that does not block the real-time broadcast stories.

**Independent Test**: Can be fully tested by placing the device into airplane mode, performing several mutations, re-enabling connectivity, and verifying each mutation appears on the server and triggers broadcast events to other clients.

**Acceptance Scenarios**:

1. **Given** a device that goes offline, **When** the user creates a contact, **Then** the mutation is stored in the local queue and persisted so it survives an app restart.
2. **Given** a device with queued mutations that restarts while offline, **When** the app is reopened, **Then** the queue is intact and no queued mutation has been lost.
3. **Given** a device with queued mutations that reconnects to the network, **When** connectivity is restored, **Then** all queued mutations are submitted to the server in enqueue order without requiring any user action.
4. **Given** a queued mutation that the server accepts, **When** the server returns a success response, **Then** the mutation is removed from the local queue.

---

### User Story 5 - Conflict Resolution (Priority: P3)

While a user's iOS device was offline, someone updated the same contact on the web client. When the device reconnects and attempts to submit its queued edit, the server detects that the contact has changed since the offline edit was made. The user is shown both versions side-by-side and chooses which one to keep.

**Why this priority**: P3 because conflicts only arise after offline queue and broadcasting are working. The app must never silently overwrite data, but this scenario is less frequent than the happy-path offline flow.

**Independent Test**: Can be fully tested by engineering a conflict (offline edit on device, server-side edit via REST), then replaying the queue and verifying the conflict prompt appears with both versions displayed and that the user's choice is applied correctly.

**Acceptance Scenarios**:

1. **Given** a queued mutation that targets an entity updated on the server since the mutation was enqueued, **When** the client submits the mutation during replay, **Then** the server returns a conflict response containing the current server state.
2. **Given** a conflict response has been received, **When** the user is presented with both the local version and the server version, **Then** the user MUST make an explicit choice — the app does not auto-resolve or time out to a default.
3. **Given** the user chooses the local version, **When** the choice is confirmed, **Then** the local version is submitted to the server as an overwrite and the server state is updated accordingly.
4. **Given** the user chooses the server version, **When** the choice is confirmed, **Then** the queued mutation is discarded and the local state is updated to match the server version.
5. **Given** a queue with multiple entries where one entry conflicts, **When** the conflict is resolved, **Then** replay continues with the remaining queued entries.

---

### User Story 6 - Retry and Error Surfacing (Priority: P4)

A user's device is in a weak-signal area. A queued mutation fails when submitted. The app retries automatically with increasing delays. If all four attempts fail, the user is told something went wrong and asked to try again.

**Why this priority**: Retry behavior is a robustness concern that sits on top of the replay flow. It does not need to be implemented before the offline queue, but it is required before the feature is considered production-ready.

**Independent Test**: Can be fully tested by simulating network failures and verifying the retry timing sequence (1s, 2s, 4s, 8s) and the error surface after the fourth failure.

**Acceptance Scenarios**:

1. **Given** a queued mutation whose REST submission fails, **When** the failure occurs, **Then** the client retries after 1 second.
2. **Given** a second consecutive failure, **When** it occurs, **Then** the client retries after 2 seconds.
3. **Given** a third consecutive failure, **When** it occurs, **Then** the client retries after 4 seconds.
4. **Given** a fourth consecutive failure, **When** it occurs, **Then** the client retries after 8 seconds, and if this attempt also fails, the error is surfaced to the user and retrying stops.

---

### Edge Cases

- What happens when a sync event arrives for an entity the client does not have locally (e.g., `contact-updated` for an unknown ID)? → The client fetches the full entity via REST and merges it into local state; it does not silently discard the event.
- What happens when the offline queue contains a delete mutation for a contact that was also deleted server-side? → The server returns 404 during replay; the client treats 404 as a successful resolution (entity is gone) and removes the entry from the queue.
- What happens when a conflict occurs for a delete mutation (local: delete, server: updated)? → The user is shown the local intent (delete) and the current server version; the user explicitly chooses to proceed with the delete or retain the server version.
- What happens when the device UUID cannot be read from local storage? → A new UUID is generated and persisted; no mutation is blocked.
- What happens when the WebSocket connection drops after a successful handshake? → The client attempts to reconnect; queued mutations are replayed once reconnected.
- What happens when the JWT expires while the WebSocket connection is active? → The server closes the connection; the client must re-authenticate and reconnect before receiving further events.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The server MUST broadcast a `contact-created` event to all authenticated WebSocket connections belonging to the mutating user immediately after a successful POST /api/contacts response is committed.
- **FR-002**: The server MUST broadcast a `contact-updated` event to all authenticated WebSocket connections belonging to the mutating user immediately after a successful PATCH /api/contacts/:id response is committed.
- **FR-003**: The server MUST broadcast a `contact-deleted` event to all authenticated WebSocket connections belonging to the mutating user immediately after a successful DELETE /api/contacts/:id response is committed.
- **FR-004**: The server MUST broadcast the corresponding interaction event (`interaction-created`, `interaction-updated`, `interaction-deleted`) to all authenticated WebSocket connections belonging to the mutating user immediately after each successful interaction mutation is committed.
- **FR-005**: The server MUST NOT broadcast an event when a REST mutation fails (non-2xx response).
- **FR-006**: The server MUST reject any WebSocket connection attempt that does not present a valid JWT.
- **FR-007**: Each sync event payload MUST include: entity type (`contact` | `interaction`), mutation type (`created` | `updated` | `deleted`), entity ID, full entity payload for creates and updates, and a tombstone marker (entity ID only) for deletes.
- **FR-008**: Each sync event MUST carry a version field corresponding to the `X-Sync-Version` in use at broadcast time.
- **FR-009**: When a client receives a sync event whose version field is not recognized by the client, the client MUST present a user-visible warning and MUST NOT apply the event to local state.
- **FR-010**: All REST API requests MUST include the `X-Sync-Version: 1` header.
- **FR-011**: Each device MUST generate and persist a unique UUID on first use, stored in platform-appropriate local storage, and include it in sync-related requests.
- **FR-012**: The iOS client MUST enqueue every contact and interaction mutation locally before attempting any network request; the local queue MUST persist across app restarts.
- **FR-013**: The iOS client MUST replay the local queue in enqueue order when network connectivity is restored, without requiring user action.
- **FR-014**: The server MUST return a conflict response when a replayed mutation targets an entity whose server-side state has changed since the mutation was enqueued.
- **FR-015**: The iOS client MUST present the user with the local (queued) version and the current server version when a conflict response is received; the user MUST explicitly choose one — the client MUST NOT auto-resolve.
- **FR-016**: After the user selects a version, the client MUST apply the chosen version: either overwrite the server state via REST (local wins) or discard the queued mutation and update local state (server wins).
- **FR-017**: A conflict on one queue entry MUST NOT prevent replay of subsequent entries once the conflict is resolved.
- **FR-018**: When a sync request fails, the client MUST retry with exponential backoff: 1 second, 2 seconds, 4 seconds, 8 seconds.
- **FR-019**: After four consecutive failed retry attempts, the client MUST surface the error to the user and cease further automatic retries for that entry.
- **FR-020**: Sync scope is limited to contacts and interactions; authentication tokens and other data MUST NOT be broadcast as sync events.

### Key Entities

- **Sync Event**: A server-to-client push notification describing a completed mutation. Attributes: entity type, mutation type, entity ID, full entity payload (creates/updates) or tombstone marker (deletes), sync version.
- **Offline Queue**: A durable, ordered list of pending mutations stored on the device. Each entry stores: mutation type, entity type, request payload, and enqueue timestamp. The queue persists across app restarts and is replayed in FIFO order on reconnect.
- **Device Identity**: A UUID assigned to each device installation on first use. Stored in platform-appropriate local storage and included in sync-related requests to attribute mutations.
- **Conflict**: A state detected during offline queue replay where the entity targeted by a queued mutation has been independently mutated on the server since the mutation was enqueued.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A mutation committed on one connected client appears on all other connected clients within 5 seconds under normal network conditions, with no user action required on the receiving clients.
- **SC-002**: 100% of WebSocket connection attempts without a valid JWT are rejected at the handshake stage.
- **SC-003**: Zero mutations made while offline are lost — every queued mutation either succeeds on replay, is explicitly resolved by the user, or surfaces an error to the user after four failed attempts.
- **SC-004**: 100% of conflicts detected during replay are shown to the user for explicit resolution; zero conflicts are auto-resolved silently.
- **SC-005**: After four consecutive failed retry attempts, 100% of those failures result in a user-visible error; automatic retrying stops.
- **SC-006**: 100% of received sync events carrying an unrecognized version value trigger a user-visible warning rather than silent state mutation.
- **SC-007**: The local queue survives an app restart with zero entry loss in 100% of tested cases.

## Assumptions

- **Auth, Contact Management, and Interaction Log** are fully specified, and their open questions resolved, before this feature proceeds to planning or implementation.
- **Broadcast scope**: sync events are delivered only to authenticated connections belonging to the same user who performed the mutation. Cross-user broadcasting is not in scope.
- **Conflict detection**: a conflict is identified by comparing the `updatedAt` timestamp of the server entity at the time the mutation is replayed against the `updatedAt` value recorded in the queued entry at enqueue time. The field name `updatedAt` must match the canonical field name defined in the Contact Management data model.
- **Queue entry granularity**: each discrete mutation (one create, one update, one delete) is a separate queue entry. Batch mutations are not in scope.
- **Partial queue replay**: if a queue contains multiple entries and one conflicts, conflict resolution is per-entry. Remaining entries continue to replay after the conflict is resolved.
- **WebSocket token lifecycle**: the JWT is validated at connection time. Mid-session token expiry closes the connection; the client must re-authenticate and reconnect. Token refresh behavior is defined by the Auth feature and is out of scope here.
- **Delete tombstones**: the format of delete sync events (tombstone) follows the deletion strategy (soft vs. hard delete) decided during the Contact Management specify/clarify cycle.
- **No server-side queue**: the server does not persist undelivered events for disconnected clients. A client that was offline during a broadcast must reconcile its state by fetching the current data via REST on reconnect, then processing any subsequent events normally.
- **Single-user product**: no multi-user or sharing scope; broadcast is always scoped to the single authenticated user.

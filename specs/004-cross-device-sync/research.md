# Research: Cross-Device Sync

**Feature**: `004-cross-device-sync` | **Date**: 2026-06-08

## Decision 1: WebSocket Authentication Method

**Decision**: The JWT is passed to the server via the Socket.io handshake auth object — a structured payload exchanged during the Socket.io connection upgrade, separate from query parameters and HTTP headers.

**Rationale**: Query-parameter tokens are logged by servers, proxies, and CDNs, which leaks credentials into log files. Custom HTTP headers are not reliably forwarded by all WebSocket upgrade intermediaries. The Socket.io handshake auth object is transmitted during the protocol handshake and is not appended to URLs or logged by default. The JWT is the same credential already used for REST API calls; no new credential type is needed.

**Alternatives considered**:
- *Query parameter (`?token=<jwt>`)* — Rejected: token appears in access logs on every connection attempt.
- *HTTP `Authorization` header during upgrade* — Rejected: not universally supported across Socket.io client implementations and reverse-proxy configurations.

---

## Decision 2: User-Scoped Event Isolation (Room Strategy)

**Decision**: The server places every authenticated Socket.io connection into a room keyed by the user's UUID (the JWT `sub` claim). All mutation broadcasts target that room. No connection in any other user's room receives the event.

**Rationale**: The application is single-user per account. All of a user's devices share the same `sub`; targeting by `sub` is both correct and minimal. No cross-user broadcast is ever needed.

**Alternatives considered**:
- *Separate namespace per user* — Rejected: dynamic namespace creation is non-trivial and adds operational complexity with no benefit over rooms for this scale.
- *Broadcast to all connections + filter client-side* — Rejected: leaks other users' data to all connections; unacceptable for correctness and security.

---

## Decision 3: Sync Event Naming Convention and Payload Shape

**Decision**: Events follow the pattern `<entity>:<mutation>` — e.g., `contact:created`, `contact:updated`, `contact:deleted`, `interaction:created`, `interaction:updated`, `interaction:deleted`. Each event carries a top-level `version` string, the event name, the originating `device_id`, and the entity payload in a `data` field. Delete events carry a tombstone (entity `id` only) instead of the full object.

**Rationale**: The `<entity>:<mutation>` pattern is readable and allows clients to register per-event handlers without string parsing. Including `device_id` lets receiving clients determine whether they originated the mutation and skip redundant local processing without the server needing to know. The `version` field enables the protocol mismatch check required by FR-009 and SC-006.

**Event payload structure**:
```
{
  "version": "1",
  "event": "<entity>:<mutation>",
  "device_id": "<originating device UUID>",
  "data": { /* full entity object, or { "id": "<uuid>" } for deletes */ }
}
```

**Alternatives considered**:
- *Single generic `mutation` event with entity type in payload* — Rejected: clients must inspect the payload to route handling; named events are cleaner.
- *Omitting `device_id`* — Rejected: without it, a client that originated the REST call has no reliable way to avoid double-processing without server-side filtering logic.

---

## Decision 4: Conflict Detection Mechanism

**Decision**: Conflict detection uses the standard HTTP `If-Match` header pattern. For PATCH and DELETE mutations during offline replay, the client sends `If-Match: "<entity_updated_at>"` where `entity_updated_at` is the ISO 8601 UTC timestamp of the entity recorded at enqueue time. The server compares the current entity `updated_at` value against the `If-Match` value. If they differ, the server returns 409 Conflict with the current server entity in the response body.

**Rationale**: `If-Match` is a well-understood HTTP standard (RFC 7232) that expresses optimistic concurrency control without inventing a custom header. Using `updated_at` as the ETag value is straightforward and requires no additional version column. POST (create) requests never send `If-Match` — new entities have no prior state to conflict with.

**Alternatives considered**:
- *Custom `X-Client-Version` header* — Rejected: `If-Match` is already defined in RFC 7232 for exactly this purpose; custom headers add cognitive overhead with no advantage.
- *Monotonic integer version counter column* — Rejected: `updated_at` already exists on contacts; adding a counter to interactions is additional schema complexity. `updated_at` as a precision-adequate proxy is sufficient for a single-user, low-write-frequency app.

---

## Decision 5: Conflict Resolution HTTP Flow

**Decision**:
- **Server wins**: Client discards the queued entry and updates its local state with the server entity from the 409 response body. No additional HTTP request is needed.
- **Local wins**: Client resubmits the original PATCH or DELETE request without the `If-Match` header. An absent `If-Match` on a PATCH or DELETE is treated by the server as an unconditional write, bypassing conflict detection.

**Rationale**: The unconditional-write-when-no-If-Match pattern is a standard interpretation of RFC 7232 semantics. It requires no new endpoint, no new header, and no state on the server. The 409 response body already contains the server entity, so the "server wins" path requires no extra round-trip.

**Alternatives considered**:
- *Dedicated resolution endpoint (e.g., `POST /api/conflicts/:id/resolve`)* — Rejected: adds API surface and a server-side conflict state machine with no benefit; the REST mutation endpoints already handle both paths cleanly.
- *Special `X-Force-Overwrite: true` header* — Rejected: semantically equivalent to omitting `If-Match`; RFC 7232 already defines this behavior.

---

## Decision 6: `updated_at` Added to Interaction Entity

**Decision**: The `interactions` table gains an `updated_at` timestamp column (managed by a database trigger or ORM lifecycle hook, identical to the pattern already used on `contacts`). This is required for `If-Match` conflict detection on interaction mutations.

**Rationale**: Without `updated_at` on interactions, offline PATCH/DELETE mutations against interactions have no server-side timestamp to compare against — conflict detection would be impossible for that entity type. Adding the column is the minimal schema change that unblocks sync without altering existing interaction semantics.

**Impact**: The Interaction API contract (003-interaction-log) will return `updated_at` in interaction objects once this column is added. Platform implementations must add this column in the migration for this feature.

---

## Decision 7: No Server-Side Event Buffering

**Decision**: The server does not persist or buffer undelivered sync events for offline clients. A client that was offline during a broadcast must reconcile by fetching current state via the REST API on reconnect (e.g., re-fetching the contact list and any open interaction lists). Subsequent Socket.io events after reconnect are then applied incrementally.

**Rationale**: Buffering events requires server-side storage and a delivery-confirmation protocol, which adds significant complexity. The REST API is the authoritative data source; a full re-fetch on reconnect is a reliable, simple alternative. At single-user personal CRM scale, the data volume of a re-fetch is negligible.

**Alternatives considered**:
- *Event store with sequence numbers and client-side cursor* — Rejected: high complexity, requires persistent event log, out of scope for this product scale (constitution Principle IV).
- *Socket.io "missed events" via volatile events + ack* — Rejected: volatile events are explicitly fire-and-forget; Socket.io acknowledgements only cover client-confirmed receipt, not recovery of missed events.

---

## Decision 8: Offline Queue Entry Structure

**Decision**: Each entry in the persistent offline queue stores the following fields:

| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID | Unique entry identifier; allows de-duplication on replay |
| `entity_type` | Enum | `contact` \| `interaction` |
| `mutation_type` | Enum | `created` \| `updated` \| `deleted` |
| `entity_id` | UUID | ID of the entity being mutated |
| `payload` | Object | Full request body to submit to the REST API |
| `entity_updated_at` | Timestamp (UTC) | `updated_at` of the entity at enqueue time; null for create entries |
| `enqueued_at` | Timestamp (UTC) | When the mutation was queued; used for ordering |

Entries are replayed in ascending `enqueued_at` order (FIFO).

**Rationale**: `entity_updated_at` is the value used in the `If-Match` header during replay. `id` allows the client to remove the entry from the queue after a successful submit or after user resolution. `enqueued_at` ensures ordering is preserved even if the device clock is adjusted between enqueue and replay.

---

## Resolved Unknowns

| Unknown | Resolution |
|---------|------------|
| WebSocket auth method | Socket.io handshake auth object; same JWT as REST calls |
| User event isolation | Per-user Socket.io room keyed by JWT `sub` |
| Event naming | `<entity>:<mutation>` convention; `version` + `device_id` in payload |
| Conflict detection | `If-Match: "<entity_updated_at>"` on PATCH/DELETE during replay; 409 on mismatch |
| Conflict resolution flow | Server wins: discard queue entry; Local wins: resubmit without `If-Match` |
| `updated_at` on Interaction | Added — required for conflict detection parity |
| Server-side event buffering | None — reconnecting clients re-fetch via REST |
| Queue entry format | Documented above with 7 fields including `entity_updated_at` and `enqueued_at` |
# Contract: WebSocket Sync Events

**Feature**: `004-cross-device-sync` | **Date**: 2026-06-08
**Transport**: Socket.io over WebSocket

## Connection

### Establishing a Connection

Clients connect to the server's Socket.io endpoint using the same host as the REST API. The JWT access token is passed via the Socket.io handshake **auth object** — not as a query parameter or HTTP header.

**Handshake auth payload** (sent by the client at connection time):

```json
{
  "token": "<jwt_access_token>"
}
```

The server validates the JWT immediately on connection. If the token is missing, expired, or invalid, the server rejects the connection.

### Connection Outcomes

| Outcome | Description |
|---------|-------------|
| **Accepted** | JWT is valid; server places the connection into the user's room (keyed by JWT `sub`); client begins receiving sync events |
| **Rejected — no token** | Handshake auth contains no `token` field; connection refused |
| **Rejected — invalid/expired token** | Token fails JWT verification; connection refused |

### Disconnection

If the JWT expires while the connection is active, the server closes the connection. The client must re-authenticate (obtain a fresh JWT via the Auth API) and reconnect before receiving further events.

---

## Server-to-Client Events

All events follow a shared envelope. The client MUST check the `version` field before processing.

### Shared Envelope

```json
{
  "version": "1",
  "event": "<entity>:<mutation>",
  "device_id": "<originating_device_uuid>",
  "data": { }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `version` | String | Protocol version. Currently `"1"`. If the client does not recognize this value, it MUST warn the user and MUST NOT apply the event to local state. |
| `event` | String | One of the six event names listed below. |
| `device_id` | UUID | UUID of the device that made the REST mutation. Clients MAY use this to skip processing if it matches their own device UUID. |
| `data` | Object | Entity payload (full object for creates/updates) or tombstone (see below). |

### Event Names

| Event Name | Trigger |
|------------|---------|
| `contact:created` | POST /api/contacts succeeded |
| `contact:updated` | PATCH /api/contacts/:id succeeded |
| `contact:deleted` | DELETE /api/contacts/:id succeeded |
| `interaction:created` | POST /api/contacts/:contact_id/interactions succeeded |
| `interaction:updated` | PATCH /api/contacts/:contact_id/interactions/:id succeeded |
| `interaction:deleted` | DELETE /api/contacts/:contact_id/interactions/:id succeeded |

---

### `contact:created`

Emitted after a contact is successfully created.

```json
{
  "version": "1",
  "event": "contact:created",
  "device_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "aaaabbbb-cccc-dddd-eeee-ffffaaaabbbb",
    "name": "Jordan Lee",
    "company": "Acme Corp",
    "role": "Engineer",
    "where_met": "TechConf 2026",
    "date_met": "2026-06-08",
    "follow_up_date": null,
    "notes": null,
    "priority": "warm",
    "created_at": "2026-06-08T10:00:00Z",
    "updated_at": "2026-06-08T10:00:00Z"
  }
}
```

---

### `contact:updated`

Emitted after a contact is successfully updated. `data` contains the full contact object after the update.

```json
{
  "version": "1",
  "event": "contact:updated",
  "device_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "aaaabbbb-cccc-dddd-eeee-ffffaaaabbbb",
    "name": "Jordan Lee",
    "company": "Acme Corp",
    "role": "Senior Engineer",
    "where_met": "TechConf 2026",
    "date_met": "2026-06-08",
    "follow_up_date": "2026-07-01",
    "notes": "Promoted last month.",
    "priority": "hot",
    "created_at": "2026-06-08T10:00:00Z",
    "updated_at": "2026-06-08T11:30:00Z"
  }
}
```

---

### `contact:deleted`

Emitted after a contact is successfully deleted. `data` is a tombstone containing only the `id`.

```json
{
  "version": "1",
  "event": "contact:deleted",
  "device_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

### `interaction:created`

Emitted after an interaction is successfully logged.

```json
{
  "version": "1",
  "event": "interaction:created",
  "device_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "data": {
    "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
    "contact_id": "550e8400-e29b-41d4-a716-446655440000",
    "date": "2026-06-08",
    "type": "coffee",
    "notes": "Discussed the new project.",
    "custom_type": null,
    "created_at": "2026-06-08T12:00:00Z",
    "updated_at": "2026-06-08T12:00:00Z"
  }
}
```

---

### `interaction:updated`

Emitted after an interaction is successfully edited. `data` contains the full interaction object after the update.

```json
{
  "version": "1",
  "event": "interaction:updated",
  "device_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "data": {
    "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900",
    "contact_id": "550e8400-e29b-41d4-a716-446655440000",
    "date": "2026-06-08",
    "type": "meeting",
    "notes": "Changed to an in-person meeting.",
    "custom_type": null,
    "created_at": "2026-06-08T12:00:00Z",
    "updated_at": "2026-06-08T13:45:00Z"
  }
}
```

---

### `interaction:deleted`

Emitted after an interaction is permanently deleted. `data` is a tombstone.

```json
{
  "version": "1",
  "event": "interaction:deleted",
  "device_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "data": {
    "id": "7b3e9f12-1a2b-4c3d-8e4f-556677889900"
  }
}
```

---

## Unrecognized Version Handling

If a client receives an event whose `version` field is not in its list of supported versions, the client MUST:
1. Display a user-visible warning (e.g., "Your app may be out of date. Please update to receive live changes.").
2. Discard the event — MUST NOT apply it to local state.

The client MUST NOT attempt to parse or infer the `data` payload of an unrecognized-version event.

---

## Event Delivery Guarantees

| Property | Guarantee |
|----------|-----------|
| **Ordering** | Events are emitted after the database commit; ordering within a single connection's event stream reflects commit order for that user |
| **At-most-once** | Events are emitted once per mutation; Socket.io does not retry unacknowledged events (fire-and-forget) |
| **No buffering** | Events emitted while a client is disconnected are not stored or replayed; the client must re-fetch current state via REST on reconnect |
| **Scope** | Events are delivered only to connections in the originating user's room; no cross-user leakage |

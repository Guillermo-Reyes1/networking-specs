# Data Model: Cross-Device Sync

**Feature**: `004-cross-device-sync` | **Date**: 2026-06-08

## New Entities

### Offline Queue Entry

A client-side record representing a mutation made while offline, pending submission to the REST API. The queue is persisted in platform-appropriate local storage so entries survive app restarts. Entries are replayed in ascending `enqueued_at` order (FIFO).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUID | NOT NULL, unique | Entry identifier; used to remove the entry after resolution |
| `entity_type` | Enum | NOT NULL | One of: `contact`, `interaction` |
| `mutation_type` | Enum | NOT NULL | One of: `created`, `updated`, `deleted` |
| `entity_id` | UUID | NOT NULL | ID of the contact or interaction being mutated |
| `payload` | Object | NOT NULL | Full request body to submit to the REST API on replay |
| `entity_updated_at` | Timestamp (UTC) | NULLABLE | `updated_at` of the entity recorded at enqueue time; null for `created` entries; used as the `If-Match` value on replay |
| `enqueued_at` | Timestamp (UTC) | NOT NULL | When the mutation was placed in the queue; determines replay order |

**Lifecycle**:
1. Entry is created locally when a mutation is made while offline.
2. On reconnect, entries are submitted to the REST API in `enqueued_at` order.
3. On success (2xx): entry is removed from the queue.
4. On conflict (409): entry is held pending user resolution; removed after resolution.
5. On failure after four retries: entry is held; error is surfaced to user. Entry is not automatically removed — user must take action.

---

### Sync Event

A server-to-client push notification delivered over Socket.io. Not persisted on the server. Describes a mutation that was successfully committed to the database.

| Field | Type | Notes |
|-------|------|-------|
| `version` | String | Protocol version; currently `"1"`. Clients MUST warn and discard if unrecognized. |
| `event` | String | One of the six event names: `contact:created`, `contact:updated`, `contact:deleted`, `interaction:created`, `interaction:updated`, `interaction:deleted` |
| `device_id` | UUID | UUID of the device that originated the REST mutation; allows receiving clients to skip redundant processing |
| `data` | Object | Full entity object for `created` and `updated` events; tombstone `{ "id": "<uuid>" }` for `deleted` events |

**Routing**: Events are emitted into a Socket.io room identified by the user's UUID (`sub` claim from the JWT). Only authenticated connections in that room receive the event.

---

### Device Identity

A stable identifier assigned to each device installation. Persisted in platform-appropriate local storage on first use and never rotated. Included in sync-related requests so the server can attribute mutations to their originating device.

| Field | Type | Notes |
|-------|------|-------|
| `device_id` | UUID | Generated on first app launch; persisted locally; sent with sync requests |

---

## Modified Entities

### Interaction (amendment to `003-interaction-log/data-model.md`)

The `interactions` table gains one new column required for conflict detection during offline queue replay.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `updated_at` | Timestamp (UTC) | NOT NULL, default now() | Updated on every PATCH; managed by ORM lifecycle hook or database trigger. Mirrors the pattern already established on the `contacts` table. |

**All other Interaction fields are unchanged.** See [003-interaction-log/data-model.md](../../003-interaction-log/data-model.md) for the full entity definition.

The Interaction object returned by the API (GET, POST, PATCH) now includes `updated_at` in the response body, positioned after `created_at`:

```
{
  "id": "...",
  "contact_id": "...",
  "date": "2026-06-07",
  "type": "coffee",
  "notes": "...",
  "custom_type": null,
  "created_at": "2026-06-07T14:22:00Z",
  "updated_at": "2026-06-08T09:00:00Z"
}
```

---

## Entity Relationships

```
User 1 ──── N Contact 1 ──── N Interaction
                │
                └── updated_at (existing)

Interaction
  └── updated_at (NEW — added by this feature)

Device (client-side, not server-stored)
  └── device_id (UUID, persisted locally)

Offline Queue (client-side, not server-stored)
  └── entries ordered by enqueued_at ASC
```

---

## Server-Side Storage Note

Sync Events and Offline Queue Entries are **not stored in the server database**. Sync Events are ephemeral Socket.io emissions. Offline Queue Entries live in the client's local storage. The only server-side schema change introduced by this feature is the `updated_at` column on `interactions`.

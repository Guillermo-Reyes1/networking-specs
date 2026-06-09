# Implementation Plan: Cross-Device Sync

**Branch**: `004-cross-device-sync` | **Date**: 2026-06-08 | **Spec**: [spec.md](spec.md)

## Summary

Define the real-time sync layer that ties together the REST API and all connected clients. The server broadcasts Socket.io events after every successful contact or interaction mutation. iOS clients queue mutations locally while offline and replay them on reconnect using HTTP `If-Match` for optimistic concurrency control. Conflicts surface a user-prompt; no silent overwrites. Exponential backoff (1s/2s/4s/8s) handles transient failures; errors are surfaced after four consecutive failures. The only server-side schema change is an `updated_at` column added to `interactions`.

## Shared Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| Spec | `specs/004-cross-device-sync/spec.md` | Acceptance scenarios and functional requirements |
| WebSocket Contract | `specs/004-cross-device-sync/contracts/websocket-events.md` | Socket.io connection and server-to-client event definitions |
| REST Sync Extensions | `specs/004-cross-device-sync/contracts/sync-rest-extensions.md` | `If-Match` header, 409 Conflict response, `updated_at` on interactions |
| Data Model | `specs/004-cross-device-sync/data-model.md` | Sync Event, Offline Queue Entry, Device Identity entities; Interaction amendment |
| Research | `specs/004-cross-device-sync/research.md` | Cross-platform decisions and rationale |
| Quickstart | `specs/004-cross-device-sync/quickstart.md` | 17 end-to-end validation scenarios |

## Resolved Open Questions

| Question | Resolution |
|----------|------------|
| WebSocket auth method | Socket.io handshake auth object; same JWT as REST calls; not a query parameter |
| User event isolation | Per-user Socket.io room keyed by JWT `sub` claim |
| Event naming convention | `<entity>:<mutation>` (e.g., `contact:created`); version + device_id in envelope |
| Conflict detection | `If-Match: "<entity_updated_at>"` on PATCH/DELETE replay; 409 on mismatch |
| Conflict resolution — server wins | Client discards queue entry; applies `server_version` from 409 body; no extra request |
| Conflict resolution — local wins | Client resubmits PATCH/DELETE without `If-Match` (unconditional write) |
| `updated_at` on Interaction | Added — required for conflict detection parity with Contact |
| Server-side event buffering | None — reconnecting clients re-fetch state via REST; events are fire-and-forget |
| Queue entry format | 7 fields: id, entity_type, mutation_type, entity_id, payload, entity_updated_at, enqueued_at |

## Platform Overlay Note

Implementation details — runtime, framework, ORM, WebSocket library, local storage mechanism, and source code structure — are platform-specific. They MUST be defined in each platform overlay (`networking-web`, `networking-ios`) and MUST NOT be specified here.

## Technical Context

**Language/Version**: Platform-specific — defined in platform overlays.

**Primary Dependencies**: Platform-specific — defined in platform overlays.

**Storage**: PostgreSQL. One schema change: `interactions` table gains `updated_at` timestamp column (NOT NULL, default now(), updated by ORM lifecycle hook or database trigger).

**Testing**: Platform-specific — defined in platform overlays.

**Target Platform**: Multi-platform. WebSocket server runs alongside the REST API. iOS client implements offline queue and conflict resolution.

**Project Type**: Shared, platform-agnostic spec. Covers server-side broadcast and client-side sync protocol.

**Performance Goals**: Server mutation events delivered to all connected clients within 5 seconds under normal conditions (SC-001).

**Constraints**: Socket.io WebSocket (delivery-only; all mutations via REST). JWT required at connection time. `X-Sync-Version: 1` header on all REST requests. Exponential backoff 1s/2s/4s/8s; error surfaced after four failures. No server-side event buffering. Hard-delete tombstones. Single-user scope.

**Scale/Scope**: Single user; realistic concurrent connections: 2–3 devices. Contact and interaction entities only.

## Constitution Check

*Evaluated against constitution v1.1.0. Re-checked after Phase 1 design.*

| Principle | Verdict | Notes |
|-----------|---------|-------|
| I. Speed of Capture First | ✅ PASS | Offline queue ensures tapping "save" persists locally within 100 ms regardless of network (constitution requirement met). Server sync happens in the background after the local write. No sync flow blocks the capture path. |
| II. Offline-First on Mobile | ✅ PASS | This is the spec that directly implements Principle II. Every contact and interaction mutation is enqueued locally first. Conflicts are surfaced to the user, never silently overwritten. Queue survives restarts. |
| III. Real-Time Cross-Device Sync | ✅ PASS | This spec is the direct implementation of Principle III. Socket.io broadcasts after every committed mutation. `X-Sync-Version` header enforced. Version mismatches warn the user rather than corrupting state. |
| IV. Simplicity Over Features | ✅ PASS | User-prompted conflict only — no automatic merge. No server-side event buffering. No cross-user scope. No sequence numbers or event log. Reconnect re-fetch instead of replay. All consistent with YAGNI and portfolio scale. |
| V. API-Driven Data Model | ✅ PASS | WebSocket is delivery-only. All mutations go through existing REST endpoints. Same shared contracts for all platforms. No client-specific endpoints. JWT authenticates both REST and WebSocket. |

**All gates pass.**

## Project Structure

### Documentation (this feature)

```text
specs/004-cross-device-sync/
├── plan.md                        # This file
├── spec.md                        # Feature spec
├── research.md                    # Decisions and rationale
├── data-model.md                  # Sync entities + Interaction amendment
├── quickstart.md                  # 17 end-to-end validation scenarios
├── contracts/
│   ├── websocket-events.md        # Socket.io connection and event contract
│   └── sync-rest-extensions.md   # If-Match, 409 Conflict, updated_at on interactions
├── checklists/
│   └── requirements.md            # Spec quality checklist
└── tasks.md                       # Generated by /speckit-tasks (not yet created)
```

### Source Code

Platform-specific. Defined in platform overlays. Not specified here.

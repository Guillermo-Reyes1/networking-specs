<!--
Sync Impact Report
==================
Version change: (unversioned template) → 1.0.0
New constitution — initial adoption.

Principles defined:
  I.   Speed of Capture First
  II.  Offline-First on Mobile
  III. Real-Time Cross-Device Sync
  IV.  Simplicity Over Features
  V.   API-Driven Data Model

Sections added:
  - Core Principles (5 principles)
  - Technical Constraints
  - Development Workflow
  - Governance

Templates status:
  ✅ .specify/templates/plan-template.md  — Constitution Check section present; gates are principle-agnostic placeholders, no update required
  ✅ .specify/templates/spec-template.md  — No constitution references; no update required
  ✅ .specify/templates/tasks-template.md — No constitution references; no update required

Deferred items:
  - None. All placeholders resolved.
-->

# Networking Contact Manager Constitution

## Core Principles

### I. Speed of Capture First

Mobile contact capture MUST be the fastest path to saving a person's details. Every design
decision that creates friction on the mobile capture flow MUST be rejected unless there is an
overriding safety or correctness requirement.

- The capture form MUST require only name; all other fields are optional.
- Tapping "save" MUST persist the contact locally within 100 ms, regardless of network state.
- No confirmation dialogs, multi-step wizards, or loading spinners MUST block the save path.

**Rationale**: A student at a live event has seconds, not minutes. Missed captures are
permanent information loss. Speed here directly fulfills the product promise.

### II. Offline-First on Mobile

The iOS client MUST operate fully without a network connection. All writes go to a local
queue first; sync to the server happens on reconnect.

- Every contact mutation (create, update, delete) MUST be enqueued locally before any
  network call is attempted.
- Conflicts detected on sync MUST be surfaced to the user for manual resolution — the app
  MUST NOT silently overwrite either version.
- The sync queue MUST survive app restarts.

**Rationale**: Networking events often have poor connectivity. Data captured on mobile MUST
never be lost because the server was unreachable.

### III. Real-Time Cross-Device Sync

Server-side mutations MUST propagate to all connected clients via WebSocket within a few
seconds. The web and mobile experiences MUST reflect the same data state.

- The server MUST broadcast change events over Socket.io after every successful mutation.
- Clients MUST apply incoming events optimistically without requiring a full page reload.
- API versioning via `X-Sync-Version` header MUST be enforced so clients can detect
  protocol mismatches and warn the user rather than silently corrupting state.

**Rationale**: The user switches between devices throughout the day. Stale data on one
device creates confusion and missed follow-ups.

### IV. Simplicity Over Features

This is a single-user portfolio project. Every proposed addition MUST pass the question:
"Does this directly serve the stated product goals?" If not, it MUST be deferred or rejected.

- No multi-user, team, or sharing features.
- No social login — email and password only.
- No AI-generated suggestions, smart sorting, or automatic categorization.
- No contact import/export (CSV, vCard, or otherwise).
- No email sending from within the app.

**Rationale**: YAGNI. Portfolio scope keeps implementation manageable and the product
focused. Adding scope now risks never shipping.

### V. API-Driven Data Model

All data access MUST go through the RESTful API. The data model and business rules live
server-side; clients are thin consumers.

- Web and iOS clients MUST share the same API contracts — no client-specific endpoints.
- All mutations MUST use the REST API; real-time updates are delivery-only via WebSocket.
- JWT authentication with refresh tokens MUST secure every API call.
- Contact fields are canonical: name, company, role, where met, date met, follow-up date
  (nullable), notes (nullable), priority (hot/warm/cold).
- The interaction log per contact is the authoritative history of follow-ups.

**Rationale**: A single source of truth prevents web/mobile divergence and makes the
backend independently testable.

## Technical Constraints

- **Auth**: Email + password only. JWT access tokens + refresh tokens. No social login.
- **Storage**: PostgreSQL. Schema migrations required for all structural changes.
- **Real-time**: Socket.io over WebSocket for push events.
- **iOS sync**: Offline queue persisted on device; user-prompted conflict resolution on
  reconnect.
- **API versioning**: `X-Sync-Version` header on all requests.
- **Pagination**: Strategy is an open question (cursor vs. offset) — MUST be decided before
  the first list endpoint is implemented and applied consistently thereafter.
- **Deletion**: Soft-delete vs. hard-delete is an open question — MUST be decided before
  any delete endpoint is implemented.

## Development Workflow

- Each feature MUST have a spec before implementation begins.
- The spec MUST define acceptance scenarios that are independently testable.
- Contact CRUD and interaction log MUST be developed and verified against the API contract
  before any client work begins.
- All open questions listed in PRODUCT.md MUST be resolved before the affected endpoint is
  built (not deferred to code review).
- Commits MUST be small and focused; one logical change per commit.
- Shared specs in networking-specs MUST be platform-agnostic. No references to localStorage, SwiftUI, React, Core Data, URLSession, or any platform-specific term are permitted in shared spec files.

## Governance

This constitution supersedes all other informal practices and ad-hoc decisions. When a
development decision conflicts with a principle, the principle wins unless an amendment is
ratified.

**Amendment procedure**:
1. Identify the principle or constraint being changed and explain the motivation.
2. Increment the version number per semantic versioning rules (see below).
3. Update this file and run the consistency propagation checklist (see Sync Impact Report
   header format) before committing.

**Versioning policy**:
- MAJOR — principle removed, renamed, or its non-negotiable rules materially weakened.
- MINOR — new principle or new mandatory constraint added.
- PATCH — wording clarification, typo fix, rationale expansion without semantic change.

**Compliance**: Every plan review and spec review MUST include a Constitution Check section
that verifies no principle is violated. Any violation MUST be justified in the Complexity
Tracking table of the plan.

**Version**: 1.0.0 | **Ratified**: 2026-06-03 | **Last Amended**: 2026-06-03

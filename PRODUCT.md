# Product Vision Document — Networking Contact Manager

## Product Vision
A lightweight personal CRM for tracking people met at hackathons, career fairs, and info sessions. Contacts captured on mobile appear instantly on the web, and vice versa.

## Target User
A single user — a computer science student actively building their professional network. They need to capture contacts quickly in the moment on mobile and review, organize, and follow up at a desktop.

## Problem Statement
Students at networking events have no lightweight tool to capture and follow up with contacts. Business cards get lost, LinkedIn connections have no context, and notes apps have no structure. The result is missed follow-ups and forgotten conversations.

## Product Goals
- Capture contacts quickly during live networking events on mobile
- Review, organize, and plan follow-ups from a desktop
- Never lose context on who someone is or where you met them
- Track follow-up interactions per contact over time
- Keep contacts in sync across devices instantly

## Non-Goals
- Not a multi-user or team product
- Not a social network or public profile system
- No social login (email and password only)
- No contact import/export (CSV, vCard)
- No email sending from within the app
- No AI suggestions or smart features

## Success Metrics
- Contacts captured during an event are visible on web within seconds
- Zero data loss when capturing offline and coming back online
- Follow-up date tracking prevents missed outreach

## Product Promise
"Capture who you meet, remember why they matter, and never miss a follow-up."

## Core Principles
1. Speed of capture on mobile above all else
2. Platform-agnostic data model — behavior is identical on web and iOS
3. User controls all conflict resolution — no silent data changes
4. Offline-first on mobile, sync on reconnect

## High-Level Requirements
- User authentication with email and password
- Contact CRUD with structured fields: name, company, role, where met, date met, follow-up date (nullable), notes (nullable), priority (hot/warm/cold)
- Interaction log per contact: date, type, notes
- Cross-device real-time sync with offline queue and user-prompted conflict resolution

## Constraints
- Technical: offline-first on iOS, real-time sync via WebSocket
- Scope: single user, portfolio project
- Time: self-directed, no hard deadline

## API Contracts
- RESTful API for all data mutations
- WebSocket (Socket.io) for real-time push
- JWT authentication with refresh tokens
- API versioning via X-Sync-Version header
- Retry: exponential backoff on failed sync requests — 1s, 2s, 4s, 8s; error surfaced to user after four attempts

## Open Questions
- [ ] Soft-delete or hard-delete for contacts?
- [ ] JWT access token expiry duration?
- [ ] Refresh token expiry duration?
- [ ] Interaction log — append-only or editable/deletable?
- [ ] Pagination strategy — cursor-based or offset?
> Pagination and deletion strategy MUST be resolved during the contacts feature specify/clarify cycle — not deferred beyond it.
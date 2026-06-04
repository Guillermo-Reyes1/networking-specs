# Implementation Plan: User Authentication

**Branch**: `001-user-auth` | **Date**: 2026-06-04 | **Spec**: [spec.md](spec.md)

## Summary

Define the REST API layer for user registration, login, token refresh, and logout. The
server issues short-lived JWT access tokens (1 hour) and long-lived opaque refresh tokens
(30 days) persisted server-side. All endpoints require the `X-Sync-Version: 1` header.
Refresh token rotation is applied on every refresh call. Client-side token storage is
platform-specific and out of scope for this shared specification.

## Shared Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| Spec | `specs/001-user-auth/spec.md` | Acceptance scenarios and functional requirements |
| API Contract | `specs/001-user-auth/contracts/auth-api.md` | Shared endpoint definitions consumed by all platforms |
| Data Model | `specs/001-user-auth/data-model.md` | Canonical entity definitions |
| Research | `specs/001-user-auth/research.md` | Cross-platform decisions and rationale |

## Resolved Open Questions

| Question | Resolution |
|----------|------------|
| Access token expiry | 1 hour (FR-011) |
| Refresh token expiry | 30 days (FR-012) |
| Refresh token strategy | Opaque random string, rotation on every use |
| X-Sync-Version initial value | `1` |
| Login failure response | Generic 401 for wrong password and unrecognized email |
| Password minimum length | 8 characters |

## Platform Overlay Note

Implementation details — runtime, framework, ORM, password hashing algorithm, request
validation library, and source code structure — are platform-specific. They MUST be
defined in each platform overlay (`networking-web`, `networking-ios`) and MUST NOT be
specified here.

## Constitution Check

| Principle | Verdict | Notes |
|-----------|---------|-------|
| I. Speed of Capture First | ✅ PASS | Auth is infrastructure. Does not touch the capture flow. |
| II. Offline-First on Mobile | ✅ PASS | Auth requires network by definition. Cached token handling is platform-specific. |
| III. Real-Time Cross-Device Sync | ✅ PASS | Auth uses REST only. No WebSocket involvement. |
| IV. Simplicity Over Features | ✅ PASS | Email + password only. No social login, OTP, magic links, or email verification. |
| V. API-Driven Data Model | ✅ PASS | All auth operations are REST endpoints shared by web and iOS. No client-specific endpoints. |

**All gates pass. No violations to justify.**
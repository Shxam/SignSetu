# Video Caption Pipeline — README

**Workspace:** SignSetu | **Collection:** Video Caption Pipeline

---

## Overview

This Postman collection tests the Video Caption Pipeline API end-to-end — covering authentication, video CRUD, caption processing, security vulnerabilities, race conditions, and cleanup.

---

## Collection Structure

| Folder | Purpose |
|--------|---------|
| `0 - Setup & Auth` | Authenticate and generate a unique session token |
| `1 - Video CRUD` | Create, list, and fetch videos; negative cases for missing fields and no auth |
| `2 - Caption Processing` | Trigger captions, poll status, fetch results; edge cases for invalid video IDs |
| `3 - Security & Validation` | SQL injection, XSS, IDOR, rate limiting, oversized payload, empty credentials |
| `4 - Race Conditions & Edge Cases` | Concurrency, duplicate triggers, wrong ownership, double-delete |
| `5 - Teardown` | Delete test videos, confirm deletion |

---

## How to Run

1. Import the collection JSON into Postman.
2. Create an environment and set `baseUrl` and `candidateId`.
3. Run folders in order: `0 → 1 → 2 → 3 → 4 → 5` using the Collection Runner.
4. Review test results — Bugs 1 and 2 will cause failures as the API incorrectly allows those operations.

---

## Environment Variables

| Variable | Scope | Purpose |
|----------|-------|---------|
| `baseUrl` | Environment | Base URL of the API server |
| `candidateId` | Environment / Collection | Auto-generated UUID per run |
| `authToken` | Collection | Bearer token from `POST /api/auth`, auto-set after login |
| `videoId` | Collection | Video ID from creation step, reused throughout |
| `xssVideoId` | Collection | Video ID created during XSS test, deleted in teardown |
| `X-Candidate-ID` | Collection | Injected automatically via pre-request script |

---

## API Flow

```
1. Pre-request script generates a unique candidateId → sets X-Candidate-ID header
2. POST /api/auth                                    → receive authToken
3. POST /api/videos                                 → receive videoId
4. POST /api/videos/{{videoId}}/process-captions    → trigger caption generation
5. GET  /api/captions?videoId={{videoId}}           → fetch generated captions
6. Edge case tests (wrong ID, duplicate, delete-while-processing)
7. DELETE /api/videos/{{videoId}}                   → cleanup
8. GET  /api/videos/{{videoId}}                     → confirm 404
9. DELETE /api/videos/{{videoId}}                   → double-delete check (must return 404, not 500)
```

---

## Pre-Request Script — Auto UUID Generation

**Location:** `POST Auth - Get Session Token` (Folder `0 - Setup & Auth`)

**Problem:** The API returned a `StateCollision` error when the same `candidateId` was reused across runs.

**Fix:** This script runs before every request and generates a fresh UUID, storing it in both environment and collection scope.

```js
const uniqueId = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    const r = Math.random() * 16 | 0;
    const v = c === 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
});
pm.environment.set("candidateId", uniqueId);
pm.collectionVariables.set("candidateId", uniqueId);
```

---

## Folder 4 — Race Conditions & Edge Cases

10 requests designed to test system stability under concurrent and adversarial conditions.

| # | Request | Method | Endpoint |
|---|---------|--------|----------|
| 1 | Authenticate | POST | `/api/auth` |
| 2 | Setup — Create Video | POST | `/api/videos` |
| 3 | Duplicate Caption Trigger (1st) | POST | `/api/videos/{{videoId}}/process-captions` |
| 4 | Duplicate Caption Trigger (2nd) | POST | `/api/videos/{{videoId}}/process-captions` |
| 5 | Delete Video While Processing | DELETE | `/api/videos/{{videoId}}` |
| 6 | Access Video — Wrong Candidate ID | GET | `/api/videos/{{videoId}}` |
| 7 | Fetch Captions — Wrong Candidate ID | GET | `/api/captions?videoId={{videoId}}` |
| 8 | Cleanup Delete | DELETE | `/api/videos/{{videoId}}` |
| 9 | Confirm Deletion | GET | `/api/videos/{{videoId}}` |
| 10 | Double Delete | DELETE | `/api/videos/{{videoId}}` |

### Test Cases in Folder 4

| Request | Expected Result | Bug Caught |
|---------|----------------|------------|
| Authenticate | `200/201`, token stored | — |
| Setup — Create Video | `200/201`, videoId stored | — |
| Duplicate Caption Trigger (1st) | `200` or `202` | — |
| Duplicate Caption Trigger (2nd) | `200 / 202 / 400 / 409` — must NOT be `500` | **Bug 3** |
| Delete Video While Processing | `200 / 204 / 409` — must NOT be `500` | — |
| Access Video — Wrong Candidate ID | `401 / 403 / 404` — must NOT be `200` | **Bonus Bug** |
| Fetch Captions — Wrong Candidate ID | `401 / 403 / 404` — must NOT be `200` | **Bonus Bug** |
| Cleanup Delete | `200 / 204 / 404` | — |
| Confirm Deletion | `404` or `401` | — |
| Double Delete | `404` — must NOT be `500` | **Bug 4** |

---

## Confirmed Bugs

### Bug 1 — No Authentication on Video Creation
**Severity:** Critical  
`POST /api/videos` with no `Authorization` header returns `201 Created`. Any anonymous caller can create content.  
**Expected:** `401 Unauthorized`

---

### Bug 2 — IDOR on Captions Endpoint
**Severity:** Critical  
Any user can read another user's captions using a known `videoId`. No ownership check is performed.  
**Expected:** `403 Forbidden` or `404 Not Found`  
**Endpoint:** `GET /api/captions?videoId=:id`

---

### Bug 3 — Wrong Status Code for Non-Existent Resource
**Severity:** High  
Requesting a video that doesn't exist returns `401 Unauthorized` instead of `404 Not Found`. This misleads clients into thinking their token is invalid when the resource simply doesn't exist.  
**Expected:** `404 Not Found`  
**Endpoints:** `GET /api/videos/:id`, `DELETE /api/videos/:id`

---

### Bug 4 — False Success on Delete of Non-Existent Resource
**Severity:** High  
Deleting a video ID that doesn't exist returns `204 No Content`, implying success. The API must verify the resource exists before claiming it was deleted.  
**Expected:** `404 Not Found`  
**Endpoint:** `DELETE /api/videos/:id`

---

### Bonus Bug — Ownership Not Enforced
**Severity:** High  
Accessing a video or its captions with a wrong `X-Candidate-ID` returns `200 OK`. The API does not verify that the requester owns the resource.  
**Expected:** `401 / 403 / 404`  
**Endpoints:** `GET /api/videos/:id`, `GET /api/captions?videoId=:id`

---

## Bug Summary

| # | Bug | Severity | Affected Endpoint |
|---|-----|----------|-------------------|
| 1 | No auth enforcement on video creation | Critical | `POST /api/videos` |
| 2 | IDOR — any user reads any captions | Critical | `GET /api/captions` |
| 3 | `401` returned for missing resource instead of `404` | High | `GET/DELETE /api/videos/:id` |
| 4 | `204` returned when deleting a non-existent video | High | `DELETE /api/videos/:id` |
| + | Wrong `X-Candidate-ID` returns `200` (ownership bypass) | High | `GET /api/videos/:id`, `GET /api/captions` |

---

## Glossary

| Term | Meaning |
|------|---------|
| **StateCollision** | Error when the same `candidateId` is reused, causing conflicting state. Fixed by UUID regeneration. |
| **UUID** | Universally Unique Identifier — 128-bit ID in `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` format. |
| **X-Candidate-ID** | Custom HTTP header that identifies the user. Used for ownership enforcement. |
| **IDOR** | Insecure Direct Object Reference — accessing another user's resource using a known ID. |
| **Race Condition** | Two operations running simultaneously causing unexpected behavior (e.g., triggering captions twice). |
| **Idempotency** | Calling an operation multiple times gives the same result. A second DELETE should return `404`, not `500`. |
| **Ownership Enforcement** | API verifies the requester owns the resource. Wrong `X-Candidate-ID` → `401/403/404`, never `200`. |
| **Pre-request Script** | JavaScript in Postman that runs before a request is sent. |
| **pm.environment.set()** | Postman method to store a value in the active environment scope. |
| **pm.collectionVariables.set()** | Postman method to store a value in the collection scope. |
| **HTTP 409 Conflict** | Status code for a request conflicting with the current resource state (e.g., duplicate caption trigger). |
| **HTTP 500 Internal Server Error** | Server-side error — must never appear for predictable edge cases. |

---

*SignSetu Workspace — Video Caption Pipeline Collection*

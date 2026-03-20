# Video Caption Pipeline — Bug Report

Postman collection test results for the Video Caption Pipeline API. Four confirmed bugs were found across authentication, authorization, and input validation.

---

## Confirmed Bugs

### Bug 1 — No Payload Size Validation

**Severity:** Medium

**Evidence:**
```
Oversized payload returns 201 instead of 400 or 413
```

**What happened:** The API accepted a description field containing ~100,000 characters and created the video successfully with a `201 Created` response. There is zero input length validation on any field.

**Expected behavior:** `400 Bad Request` or `413 Payload Too Large`

**Affected endpoint:** `POST /api/videos`

---

### Bug 2 — No Authentication Enforcement on DELETE

**Severity:** Critical

**Evidence:**
```
DELETE without auth token returns 204 instead of 401 or 403
```

**What happened:** Sending a `DELETE` request with no `Authorization` header succeeds. Any unauthenticated caller can delete videos by ID.

**Expected behavior:** `401 Unauthorized` or `403 Forbidden`

**Affected endpoint:** `DELETE /api/videos/:id`

---

### Bug 3 — IDOR on DELETE Endpoint

**Severity:** Critical

**Evidence:**
```
Deleting another user's video returns 204 instead of 403 or 404
```

**What happened:** A candidate can delete any other candidate's video by providing a known video ID. The API performs no ownership check before deletion — only the video ID is validated, not whether the requesting user owns it.

**Expected behavior:** `403 Forbidden` or `404 Not Found`

**Affected endpoint:** `DELETE /api/videos/:id`

---

### Bug 4 — 401 Returned for Missing Resource Instead of 404

**Severity:** Minor

**Evidence:**
```
Deleted video returns 401 instead of 404
```

**What happened:** After the IDOR test above deleted a video, the teardown step that attempts to confirm deletion via `GET /api/videos/:id` received `401` instead of `404`. The API returns `401 Unauthorized` for resources that don't exist, leaking that the resource is gone (or never existed) in a way that conflates auth errors with not-found errors.

**Expected behavior:** `404 Not Found`

**Affected endpoint:** `GET /api/videos/:id`

---

## Test Script Fix (Not an API Bug)

Several test scripts crash with the following error when the API returns `204 No Content`:

```
JSONError: No data, empty input at 1:1
```

This happens because the test script calls `pm.response.json()` on an empty body. Fix any assertion block that may follow a `DELETE` call:

```javascript
if (pm.response.code !== 204 && pm.response.text().length > 0) {
    const body = pm.response.json();
    // your assertions here
}
```

---

## Summary Table

| # | Bug | Evidence | Severity |
|---|-----|----------|----------|
| 1 | No payload size validation | `201` returned for ~100k character description | Medium |
| 2 | DELETE requires no auth token | `204` returned with no `Authorization` header | Critical |
| 3 | IDOR on DELETE — any user can delete any video | `204` returned with a different Candidate ID | Critical |
| 4 | `401` returned for missing resource instead of `404` | Confirm-deletion step returns `401` | Minor |

---

## Collection Structure

| Folder | Purpose |
|--------|---------|
| `0 - Setup & Auth` | Obtain session token, validate token uniqueness |
| `1 - Video CRUD` | Create, list, fetch videos; negative cases for missing fields and no auth |
| `2 - Caption Processing` | Trigger caption processing, poll status, fetch captions; edge cases for missing/invalid video IDs |
| `3 - Vulnerability Tests` | SQL injection, XSS, IDOR (read + delete), rate limiting, oversized payload, empty credentials |
| `4 - Teardown` | Delete test videos created during the run, verify deletion |

---

## Variables

| Variable | Description |
|----------|-------------|
| `baseUrl` | API base URL |
| `candidateId` | Your candidate identifier, sent as `X-Candidate-ID` on every request |
| `authToken` | Session token obtained from `POST /api/auth`, auto-set by the auth test script |
| `videoId` | ID of the video created in the CRUD section, used throughout the collection |
| `xssVideoId` | ID of the XSS test video if created, deleted during teardown |

---

## How to Run

1. Import the collection JSON into Postman.
2. Create an environment and set `baseUrl` and `candidateId`.
3. Run the collection using the Collection Runner in folder order: `0 → 1 → 2 → 3 → 4`.
4. Review test results. Bugs 2 and 3 will cause the IDOR and no-auth DELETE tests to fail as the API incorrectly allows those operations.

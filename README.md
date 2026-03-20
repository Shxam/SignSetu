# Video Caption Pipeline — Bug Report

Postman collection test results for the Video Caption Pipeline API. The following are the most critical and frequently encountered bugs found across authentication, authorization, and API contract correctness.

---

## Confirmed Bugs

### Bug 1 — No Authentication Enforcement on CREATE

**Severity:** Critical

**Evidence:**
```
Create video without auth token returns 201
```

**What happened:** Sending a `POST /api/videos` request with no `Authorization` header succeeds and creates a video. The endpoint does not validate the presence or validity of a session token. Any anonymous caller can create content.

**Expected behavior:** `401 Unauthorized`

**Affected endpoint:** `POST /api/videos`

---

### Bug 2 — IDOR on Captions Endpoint

**Severity:** Critical

**Evidence:**
```
Captions with wrong Candidate ID returns 200
```

**What happened:** Any candidate can read another candidate's captions by supplying a known `videoId`. The API performs no ownership check — it returns caption data regardless of whether the requesting user owns the video.

**Expected behavior:** `403 Forbidden` or `404 Not Found`

**Affected endpoint:** `GET /api/captions?videoId=:id`

---

### Bug 3 — Wrong Status Code for Missing Resource

**Severity:** High

**Evidence:**
```
Non-existent video returns 401 instead of 404
```

**What happened:** Requesting a video ID that does not exist returns `401 Unauthorized` instead of `404 Not Found`. This conflates an authentication failure with a missing resource, breaks standard REST semantics, and misleads API consumers into thinking their token is invalid when the resource simply doesn't exist.

**Expected behavior:** `404 Not Found`

**Affected endpoints:** `GET /api/videos/:id`, `DELETE /api/videos/:id`

---

### Bug 4 — False Success on Delete of Non-Existent Resource

**Severity:** High

**Evidence:**
```
Deleting non-existent video returns 204 instead of 404
```

**What happened:** Sending a `DELETE` request for a video ID that does not exist returns `204 No Content`, implying a successful deletion. The API should verify the resource exists before responding with success.

**Expected behavior:** `404 Not Found`

**Affected endpoint:** `DELETE /api/videos/:id`

---

## Summary Table

| # | Bug | Evidence | Severity |
|---|-----|----------|----------|
| 1 | No auth enforcement on video creation | `201` returned with no `Authorization` header | Critical |
| 2 | IDOR on captions — any user reads any captions | `200` returned with a different Candidate ID | Critical |
| 3 | `401` returned for missing resource instead of `404` | Non-existent video returns `401` | High |
| 4 | `204` returned when deleting non-existent video | Delete of unknown ID returns success | High |

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
4. Review test results. Bugs 1 and 2 will cause the no-auth create and IDOR captions tests to fail as the API incorrectly allows those operations.

# QA Test Plan — Memory Storage Layer (Mem0 Integration)

---

**Plan ID:** T-003-001
**Story:** S-003-001
**Epic:** E-003
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates that the `MemoryService` abstraction layer correctly wraps Mem0 for all four operations: `addMemory`, `fetchMemories`, `deleteMemory`, and `listMemories`. Testing covers: add and retrieve with correct schema fields, relevance ranking, delete and non-returnability, idempotency on duplicate content, Mem0 unavailability fallback, category-filtered listing, API key security (never in client), malformed response error handling, capacity eviction, and concurrency safety.

## Out of Scope

- Memory extraction from conversations (S-003-003)
- Memory review UI (S-003-007)
- Contradiction resolution logic (S-003-005)
- Memory importance weighting and versioning (future scope)
- iOS client calling Mem0 directly — this story explicitly prohibits it

## Prerequisites

- S-002-003 (module routing) is complete — `MemoryCategory` enum must match module routing categories
- Mem0 SDK is installed and configured with `MEM0_API_KEY` in the backend environment
- Backend API routes (`/api/v1/memory/*`) are implemented and reachable
- A test user account exists with a known `user_id`
- Backend environment is accessible for API key inspection

---

## Core Test Flow

### TC-003-001-001: Add, retrieve by query, delete, list with filter — full CRUD with schema validation
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006, AC-007
**Dependencies:** S-002-003

**Preconditions:**
- Test user has zero existing memories in Mem0
- Backend is running with a valid `MEM0_API_KEY` in environment variables
- iOS client network traffic is inspectable (e.g., via a proxy during test)

**Steps:**
1. Call `POST /api/v1/memory` with `{ content: "User hates slim-fit jeans", category: "wardrobe", confidence: 0.88, source: "chat", userId: "<test_user_id>" }`
2. Assert the response body contains a `Memory` object with: `id` (UUID), `user_id`, `content`, `category: "wardrobe"`, `confidence: 0.88`, `source: "chat"`, `created_at`, `updated_at`
3. Call `GET /api/v1/memory?query=jeans+preference&userId=<test_user_id>` — assert the memory from step 1 is returned in the results and appears first (highest relevance)
4. Add a second memory: `{ content: "User prefers tailored trousers", category: "wardrobe", confidence: 0.75, source: "chat" }` — then call `GET /api/v1/memory?query=outfit+fit` and assert both memories are returned ranked by relevance
5. Call `DELETE /api/v1/memory/<id_from_step_1>` — assert 200 response
6. Call `GET /api/v1/memory?query=jeans+preference` — assert the deleted memory is not present in results
7. Call `GET /api/v1/memory/all?category=wardrobe` — assert only the remaining wardrobe memory is returned; the deleted one is absent
8. Call `GET /api/v1/memory/all?category=gym` — assert zero results (no gym memories exist for this user)
9. Inspect iOS client network traffic during all steps — assert no call to any Mem0 endpoint originates from the device; all calls go to `/api/v1/memory/*`
10. Check backend environment — assert `MEM0_API_KEY` is an environment variable; assert it does not appear in any response body, log line accessible to the client, or iOS bundle

**Expected Result:**
- Step 2: Full Memory schema present with all required fields
- Step 3: Memory is returned; most relevant result is first
- Step 4: Both memories returned; ranked by relevance to query
- Step 5: Delete returns 200
- Step 6: Deleted memory absent from query results
- Step 7: Only surviving wardrobe memory listed; no deleted memory
- Step 8: Empty list (no gym memories)
- Step 9: Zero Mem0 API calls from the iOS device in traffic log
- Step 10: API key is only in backend env; not in any client-accessible surface

**Failure Indicators:**
- Any Memory schema field missing from the add response
- Relevance ranking not ordered (less relevant result appears first)
- Deleted memory still returned in fetch or list
- Deleted memory appears in category-filtered list
- Any Mem0 API call observed from the iOS client
- `MEM0_API_KEY` visible in any client-side network response

---

## Sub Flows

### TC-003-001-002: Idempotency — duplicate content updates confidence and timestamp, no duplicate created
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-002-003

**Preconditions:**
- Test user has zero existing memories

**Steps:**
1. Call `addMemory` with `{ content: "User trains fasted", category: "gym", confidence: 0.70, source: "chat" }`
2. Note the returned memory `id` and `updated_at`
3. Call `addMemory` again with identical `content` and `category` but `confidence: 0.85`
4. Call `listMemories` for the user's gym category

**Expected Result:**
- Step 4 returns exactly one memory with `content: "User trains fasted"` — no duplicate
- The returned memory has `confidence: 0.85` (updated to the higher value)
- `updated_at` is more recent than the value noted in step 2
- `id` is unchanged (same record)

**Failure Indicators:**
- Two memories returned with identical content
- Confidence still shows the original 0.70 after the second add
- `id` changed (a new record was created)

---

### TC-003-001-003: Mem0 unavailable — `MemoryServiceError.unavailable` thrown, pipeline falls back
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-005
**Dependencies:** S-002-003

**Preconditions:**
- Mem0 API is unreachable (simulate by revoking the API key temporarily or blocking outbound calls in test environment)

**Steps:**
1. Call `fetchMemories` for the test user
2. Observe the error thrown and the orchestration pipeline behavior

**Expected Result:**
- `MemoryService` throws `MemoryServiceError.unavailable`
- The orchestration pipeline catches the error and falls back to USER.md context only (no crash)
- The user receives a valid AI response (using USER.md context); no error is surfaced to the user
- The failure is logged server-side

**Failure Indicators:**
- App crashes or returns a 500 to the user
- Error message about Mem0 is visible in the UI
- Pipeline does not fall back — response generation is blocked entirely

---

### TC-003-001-004: Malformed Mem0 response — `MemoryServiceError.decodingFailure`, graceful recovery
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-002-003

**Preconditions:**
- A test shim or mock intercepts the Mem0 API response and returns malformed JSON

**Steps:**
1. Trigger an `addMemory` call with the malformed response shim active
2. Observe the error and server log

**Expected Result:**
- `MemoryService` throws `MemoryServiceError.decodingFailure`
- The calling pipeline step catches the error and logs the raw response for debugging
- No crash; operation fails gracefully
- User is not aware of the failure

**Failure Indicators:**
- App crashes on malformed response
- Error propagates to the user
- Raw Mem0 response not logged for debugging

---

### TC-003-001-005: Capacity limit reached — oldest lowest-confidence memory evicted, new memory stored
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-002-003

**Preconditions:**
- Test user is at Mem0's per-user record limit (populate with synthetic memories to reach the cap)
- The oldest memory in the same category has the lowest confidence score

**Steps:**
1. Call `addMemory` for a new item in the same category as the oldest low-confidence memory
2. Observe the resulting memory list

**Expected Result:**
- The new memory is stored successfully
- The oldest, lowest-confidence memory in the same category is evicted
- Total count does not exceed the Mem0 per-user limit

**Failure Indicators:**
- `addMemory` returns a capacity error without eviction
- A memory with higher confidence is evicted instead of the lowest-confidence one
- New memory is not stored even after eviction attempt

---

### TC-003-001-006: Empty content — `MemoryServiceError.invalidContent`, no API call made
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-002-003

**Preconditions:**
- Test user exists

**Steps:**
1. Call `addMemory` with `content: ""`
2. Observe the error and check whether an outbound Mem0 API call was made

**Expected Result:**
- `MemoryService` throws `MemoryServiceError.invalidContent` before any Mem0 API call
- Zero outbound Mem0 API calls are made for this request
- Error is caught by the caller; user is not affected

**Failure Indicators:**
- Empty content passes validation and reaches the Mem0 API
- Mem0 API call is made and then fails (validation should have caught it earlier)
- App crashes or returns unhandled error

---

### TC-003-001-007: Concurrent duplicate add — idempotency via upsert, no duplicate created
**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-002-003

**Preconditions:**
- Test user has zero existing memories

**Steps:**
1. Simultaneously trigger two `addMemory` calls with identical `content` and `category` (fire both concurrently from two separate test threads)
2. After both complete, call `listMemories` for the user

**Expected Result:**
- Exactly one memory exists with the given content — no duplicate
- Both calls complete without error
- The upsert operation is deterministic: one write wins; the other is treated as an update

**Failure Indicators:**
- Two memories with identical content appear in the list
- Either call throws an unhandled concurrency error
- Memory count is 0 (both writes failed)

---

## Automation Notes
- **API-level testing:** all test cases target the backend REST API (`/api/v1/memory/*`) directly — do not test through the iOS UI for this story; this keeps tests fast and deterministic
- **Schema validation:** use a JSON schema validator to assert the full `Memory` object structure on every `addMemory` response; this catches future regressions when fields are added or renamed
- **Mem0 unavailability simulation:** use a backend environment variable flag (`MEM0_OVERRIDE_FAIL=true`) to force `MemoryService` into a failure mode without actually revoking the API key
- **Capacity test setup:** write a test fixture script that populates the user to the Mem0 record limit; run this once as test setup, not per-run
- **Concurrency test (TC-003-001-007):** use `asyncio.gather()` (Python) or `Task {}` concurrency (Swift) to fire both calls simultaneously; run 20 iterations to catch intermittent duplicate creation
- **Security assertion:** use a proxy (e.g., mitmproxy) in CI to assert zero direct Mem0 calls from any client-facing IP during the test run

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

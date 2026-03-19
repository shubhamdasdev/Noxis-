# QA Test Plan — Memory Extraction from Chat Conversations

---

**Plan ID:** T-003-003
**Story:** S-003-003
**Epic:** E-003
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates that the post-response memory extraction pass correctly identifies user-stated facts from chat exchanges, writes them to Mem0 via `MemoryService`, and does so asynchronously without blocking the response stream. Testing covers: extraction of clear preferences and routine facts, empty extraction on fact-free exchanges, the 5-memory-per-exchange cap, non-blocking async dispatch, idempotency on re-extraction of existing facts with higher confidence, extraction failure graceful handling, short/emoji-only message edge case, and non-English normalization to English.

## Out of Scope

- Contradiction resolution (S-003-005)
- Displaying extracted memories to the user during or after extraction
- Blocking the chat response on extraction completion
- Confidence recalibration across sessions (future scope)
- User-triggered memory correction mid-conversation (future scope)

## Prerequisites

- S-004-005 (orchestration pipeline) is complete — extraction is dispatched as step 7 after the stream completes
- S-003-001 (memory storage) is complete — extracted candidates are written via `MemoryService`
- Extraction function `extractMemories(userId, userMessage, noxisResponse)` is implemented
- Extraction AI call is configured (GPT-4o-mini or equivalent; max tokens: 300; JSON output)
- A test user account exists with chat access
- Server error log is inspectable

---

## Core Test Flow

### TC-003-003-001: Extract preference and routine fact, enforce 5-item cap, verify async dispatch, confirm idempotency
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006, AC-007
**Dependencies:** S-004-005, S-003-001

**Preconditions:**
- Test user has zero existing memories
- Pipeline step 7 dispatch is instrumentable (can observe when `extractMemories` is called relative to stream completion)

**Steps:**
1. Send: "I always wear oversized fits — I hate anything that fits tight" — wait for the response to finish streaming
2. Assert that the input bar re-enables and the response is fully visible before the extraction task completes (confirm async: extraction must not re-disable the input bar or hold the stream)
3. Allow the extraction task to complete; call `listMemories` for the test user
4. Assert a memory with content reflecting "User prefers oversized fits" (or equivalent) exists under `category: wardrobe` with `confidence >= 0.65`
5. Send: "I train at 6am every weekday without fail"
6. Allow extraction to complete; call `listMemories`
7. Assert a memory with content reflecting "User trains at 6am every weekday" exists under `category: routines` with `confidence >= 0.65`
8. Craft a message containing 8 distinct, extractable user facts (e.g., outfit preferences, training time, diet, goal, city, sleep habit, social preference, spending habit) — send it
9. Allow extraction to complete; call `listMemories`
10. Assert exactly 5 new memories were written — no more; the 5 with highest confidence are the ones stored
11. Re-send the message from step 1 (same content) with the wardrobe memory already existing
12. Allow extraction to complete; call `listMemories` for the wardrobe category
13. Assert still exactly one wardrobe memory for that content — no duplicate; confidence and `updated_at` updated if new confidence is higher

**Expected Result:**
- Step 2: Input bar re-enables immediately after stream completes; extraction runs in background
- Step 4: Wardrobe preference memory stored with `confidence >= 0.65`
- Step 7: Routines memory stored with `confidence >= 0.65`
- Step 10: Exactly 5 memories added; the 3 lowest-confidence candidates not stored
- Step 13: One memory with updated confidence and `updated_at`; no duplicate

**Failure Indicators:**
- Input bar re-enable is delayed until extraction completes (blocking)
- No memory extracted for a clearly stated preference
- Memory stored under wrong category
- More than 5 memories written from a single exchange
- Duplicate memory created on re-extraction of the same content
- Confidence not updated on re-extraction

---

## Sub Flows

### TC-003-003-002: No extractable facts — empty array returned, no memories written, no error
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-003
**Dependencies:** S-004-005, S-003-001

**Preconditions:**
- Test user has no existing memories (for clean observation)

**Steps:**
1. Send a message with no user-stated facts: "That makes sense, I see what you mean"
2. Allow extraction to complete; call `listMemories`

**Expected Result:**
- Extraction model returns an empty array
- Zero new memories written
- No error logged; pipeline continues normally

**Failure Indicators:**
- Memories written from a content-free acknowledgement
- Extraction throws an error on empty array
- Server error log shows a failure for this exchange

---

### TC-003-003-003: Confidence below 0.65 — candidate discarded silently
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — confidence gate)
**Dependencies:** S-003-001

**Preconditions:**
- A test shim forces the extraction AI call to return a candidate with `confidence: 0.55`

**Steps:**
1. Send a message; allow the extraction task to run with the low-confidence shim active
2. Call `listMemories`

**Expected Result:**
- Zero memories written for the low-confidence candidate
- No error surfaced to the user
- Failure is silent (the discard is expected behavior, not an error)

**Failure Indicators:**
- Memory written despite confidence below 0.65 threshold
- User sees an error message

---

### TC-003-003-004: Extraction AI call fails or times out — user unaffected, failure logged
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-007
**Dependencies:** S-004-005, S-003-001

**Preconditions:**
- Extraction AI endpoint is force-failed or the timeout is set to 0ms for this test (use a test environment variable)

**Steps:**
1. Send a message that would normally produce extractable memories
2. Allow the extraction task to fail
3. Observe the user experience and the server error log

**Expected Result:**
- User sees no error; the response is delivered normally; the input bar re-enables as usual
- Zero memories written for this exchange
- Server error log contains an entry for the extraction failure (including the exchange identifier and error reason)

**Failure Indicators:**
- User sees an error or the app becomes unresponsive
- App crashes
- Failure is not logged
- Extraction failure blocks the chat response

---

### TC-003-003-005: Short or emoji-only message — empty extraction, no error
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-004-005, S-003-001

**Preconditions:**
- Test user is authenticated

**Steps:**
1. Send "k" — allow extraction to run; call `listMemories`
2. Send "👍" — allow extraction to run; call `listMemories`
3. Send "yeah" — allow extraction to run; call `listMemories`

**Expected Result:**
- For all three messages: extraction model returns an empty array; no memories written; no error
- Pipeline continues normally for each

**Failure Indicators:**
- Any memory written from a single short-form response or emoji
- Extraction throws an error on minimal input

---

### TC-003-003-006: Non-English message — fact extracted and stored in English
**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-004-005, S-003-001

**Preconditions:**
- Test user is authenticated

**Steps:**
1. Send a message in Spanish: "Siempre uso ropa holgada, odio la ropa ajustada" (translation: "I always wear loose clothes, I hate tight clothes")
2. Allow extraction to complete; call `listMemories`

**Expected Result:**
- A memory is stored in English (e.g., "User prefers loose/oversized clothing; dislikes tight fits") under `category: wardrobe`
- Memory content is not in Spanish
- `confidence >= 0.65`

**Failure Indicators:**
- Memory stored in Spanish
- No memory extracted (extraction fails on non-English input)
- Memory stored with the wrong category

---

### TC-003-003-007: AI-stated fact not extracted — only user-originated facts stored
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — extraction rule: only user-stated facts)
**Dependencies:** S-004-005, S-003-001

**Preconditions:**
- Craft an exchange where Noxis states a specific recommendation (e.g., "You should try training with compounds") that does not appear in the user's message

**Steps:**
1. User message: "Not sure what to do at the gym today"
2. Noxis response (mocked): "You should do compound movements — squats, deadlifts, bench. Keep it simple."
3. Allow extraction to complete on the above exchange; call `listMemories`

**Expected Result:**
- No memory is written for Noxis's recommendation (compound movements) as a user-stated fact
- If any memory is written, it must be grounded in what the user stated, not what Noxis recommended

**Failure Indicators:**
- Memory stored saying "User should do compound movements" (Noxis's statement extracted as user fact)
- Extraction writes any Noxis-originated information as a user memory

---

## Automation Notes
- **Async dispatch verification:** instrument the pipeline's stream completion event and the extraction task dispatch; assert the extraction task is fired after the stream completion event and does not hold the pipeline's main await
- **Cap enforcement test (TC-003-003-001 step 8):** prepare a fixed test message with exactly 8 distinct, unambiguous facts; use a test shim that returns all 8 as candidates with known confidence scores; assert only the top 5 by confidence are stored
- **Confidence gate test:** use a backend test environment variable `EXTRACTION_MOCK_CONFIDENCE=0.55` to force low-confidence output deterministically
- **Log assertion:** capture server error logs during extraction failure tests; assert the log entry contains the exchange ID and error reason
- **Non-English test:** maintain a small fixture of non-English test messages with known expected English memory outputs
- **Flakiness risk:** extraction is a live AI call with non-deterministic output; for assertion purposes, check that the category is correct and confidence meets the threshold, not that the exact content string matches; use `content contains keyword` style assertions

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# QA Test Plan — Orchestration Pipeline: Personality + Memory + Context

---

**Plan ID:** T-004-005
**Story:** S-004-005
**Epic:** E-004
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

Validates the full 8-step server-side orchestration pipeline: message classification, memory retrieval and filtering, system prompt assembly in the correct layer order, AI provider call with telemetry logging, SSE streaming initiation, and asynchronous dispatch of memory extraction (S-003-003) and dashboard sync (S-005-007). Covers SOUL.md presence, memory injection, module context routing for images, telemetry record completeness, async step isolation, decision policy capacity cap, and prompt layer ordering. Does not cover streaming display UI (S-004-006), memory extraction internals (S-003-003), or dashboard sync internals (S-005-007).

## Out of Scope

- Streaming display rendering (S-004-006)
- Memory extraction logic implementation (S-003-003) — only verifies the dispatch call
- Dashboard sync implementation (S-005-007) — only verifies the dispatch call
- Pipeline versioning and A/B testing — Future Scope
- Latency breakdown admin dashboard — Future Scope
- On-device LLM pipeline — Future Scope
- Exposing pipeline internals, system prompt content, or memory items in any UI — must never occur

## Prerequisites

- S-002-001 deployed: SOUL.md is loadable and has a version hash
- S-002-002 deployed: tone mode is retrievable per user
- S-002-003 deployed: module routing classification logic is available
- S-003-001 deployed: Mem0 integration is operational and can return ranked memory items
- S-003-004 deployed: decision policy filter is implemented
- Backend pipeline logging (`PipelineLogger`) is instrumented
- Test user account has pre-seeded memory items in Mem0 relevant to at least one life module
- AI provider (OpenAI or Anthropic) is reachable or a deterministic mock is available
- Server-Sent Events endpoint is accessible from the test client

---

## Core Test Flow

### TC-004-005-001: Full pipeline execution from user message to streamed response with all 8 steps verified

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-002-001, S-002-002, S-002-003, S-003-001, S-003-004

**Preconditions:**
- User has at least 5 memory items stored in Mem0 relevant to the wardrobe module
- User's tone mode is set to "direct" (not brother)
- A prepared outfit image is available to send (so module context layer is exercised)
- Backend pipeline log endpoint or debug output is accessible to verify log fields
- Network is stable; AI provider is responding

**Steps:**
1. Send a text message: "What do you think about how I dress in general?" to exercise text-only classification and memory retrieval.
2. Wait for the response to complete streaming.
3. Inspect the pipeline log record for this request (via backend log endpoint or debug export).
4. Verify SOUL.md is present as the first content block in the assembled system prompt (via log or test assertion).
5. Verify that the response reflects stored memory about the user's wardrobe without the user having re-stated it in this message.
6. Send the outfit image in chat.
7. Wait for the response to complete.
8. Inspect the pipeline log for the image message.
9. Verify that the system prompt log entry shows module context (wardrobe judgment lens) in layer 3.
10. Verify the prompt layer order in the log: SOUL.md → tone → module context → memory → conversation history → current message.
11. Verify the log record contains all required fields: `pipeline_run_id`, `user_id`, `soul_version`, `model`, `tokens_in`, `tokens_out`, `latency_ms`, `image_category`.
12. Confirm that steps 7 (memory extraction dispatch) and 8 (dashboard sync dispatch) completed asynchronously without delaying the visible response stream.

**Expected Result:**
- Step 5: Noxis's response to the wardrobe question references specific details from stored memory (e.g. a known style preference or past item) without the user restating those facts.
- Step 9: Pipeline log shows the wardrobe judgment lens text in layer 3 of the assembled prompt for the image message.
- Step 10: Prompt layers are in strict order: SOUL.md content first, tone instruction second, module context third, memory block fourth, conversation history fifth, current message last.
- Step 11: All listed telemetry fields are populated and non-null in the log record; `image_category` is populated for the image message.
- Step 12: No measurable streaming delay attributable to steps 7 or 8; SSE stream begins within the normal latency window and is not interrupted.

**Failure Indicators:**
- SOUL.md content is absent from the assembled prompt (layer 1 missing)
- Response does not reflect any stored memory despite relevant items existing
- Module context (judgment lens) is absent from layer 3 for an image message
- Prompt layers are out of order (e.g. memory appears before SOUL.md, or current message is not last)
- Any telemetry field is null or missing from the log record
- SSE stream is noticeably delayed or interrupted after response completion (indicating async steps are blocking)
- Pipeline internals, system prompt content, or raw memory items appear anywhere in the app UI

---

## Sub Flows

### TC-004-005-002: Memory filter caps at 20 items even when 30 candidates are retrieved

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-6
**Dependencies:** S-003-001, S-003-004

**Preconditions:**
- Test user has at least 30 memory items in Mem0 that are all relevant to the queried module (to ensure 30 candidates are retrieved)
- Decision policy filter is not configured to drop all items (items have confidence ≥ 0.5)

**Steps:**
1. Send a message that is module-relevant and would trigger retrieval of the full 30-item candidate set.
2. Inspect the pipeline log for `memory_count`.
3. Inspect the assembled prompt (via log) for the number of memory items injected.

**Expected Result:**
- `memory_count` in the log record is ≤ 20; the assembled prompt does not contain more than 20 memory items in the "What I know about this user:" block regardless of how many candidates were retrieved.

**Failure Indicators:**
- Log shows more than 20 memory items injected into the prompt
- Decision policy filter is not being applied and all 30 candidates pass through
- `memory_count` is 0 when relevant items exist (over-filtering)

---

### TC-004-005-003: Mem0 retrieval timeout falls through gracefully with USER.md profile only

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-003-001

**Preconditions:**
- Mem0 retrieval is stubbed to delay beyond 5 seconds for this test
- USER.md core profile fields for the test user are populated

**Steps:**
1. Send a message while Mem0 retrieval is artificially delayed past 5 seconds.
2. Observe that the pipeline completes and a response is returned.
3. Inspect the pipeline log for a timeout entry.
4. Observe the response for evidence of USER.md context (if identifiable from the response).
5. Confirm no error message or indicator is shown to the user.

**Expected Result:**
- The pipeline continues after 5 seconds; a Noxis response is generated using the USER.md core profile as context; the response is coherent and on-brand (SOUL.md still applied); a timeout log entry is present for monitoring; the user sees no error indicator or degraded UI state.

**Failure Indicators:**
- Pipeline halts and no response is returned when Mem0 times out
- An error message is shown to the user ("memory unavailable" or similar)
- SOUL.md is also absent from the prompt during the timeout fallback (unrelated failure compounding)
- No timeout log entry is written

---

### TC-004-005-004: AI provider failure after 3 retries returns structured error to client

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** None (AI provider abstraction layer)

**Preconditions:**
- AI provider is stubbed to return errors for all requests during this test

**Steps:**
1. Send a message while the AI provider is returning errors.
2. Observe the client UI after the pipeline exhausts its 3 retries.
3. Inspect the network response returned to the client.
4. Tap the "Try again" tap target shown in the chat.
5. Restore the AI provider to normal operation before or during the retry.
6. Observe the outcome.

**Expected Result:**
- Step 2: The chat UI shows a brief inline error message below the user's sent message; a "Try again" tap target is visible; no raw error text, status codes, or exception details are exposed to the user.
- Step 3: The server response to the client contains `{ "error": "response_unavailable" }` — the exact specified error structure.
- Step 6: After restoring the provider, tapping "Try again" triggers a new pipeline run and the response streams successfully.

**Failure Indicators:**
- Raw AI provider error (e.g. "Rate limit exceeded", HTTP 429) is displayed in the chat UI
- No "Try again" option is shown
- Client shows a generic app-level crash or alert dialog
- Pipeline does not retry before failing (fewer than 3 attempts logged)

---

### TC-004-005-005: Very long user message trims conversation history while preserving SOUL.md and current message

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** None (PromptTruncator logic)

**Preconditions:**
- A long conversation history exists (10+ prior message pairs) to provide trimming candidates
- A very long user message (>2000 tokens) is prepared
- Prompt token count is inspectable via pipeline logs

**Steps:**
1. Send the very long message (>2000 tokens) in chat.
2. Inspect the assembled prompt token count in the pipeline log.
3. Verify that the total prompt is within the model's context limit minus 1000 tokens (response buffer).
4. Verify the content of the trimmed prompt: check that SOUL.md is present in full and the current user message is present in full.
5. Verify that the trimmed sections are from the oldest end of the conversation history only.

**Expected Result:**
- Total prompt tokens ≤ (model context limit − 1000); SOUL.md content is fully present; the current user message is fully present; the oldest conversation history entries are the ones trimmed (not SOUL.md, not memory items, not the current message).

**Failure Indicators:**
- Prompt exceeds the model's context limit (causes an API error)
- SOUL.md is truncated or partially absent after trimming
- The current user message is truncated
- History is trimmed from the newest end rather than the oldest end

---

### TC-004-005-006: Memory extraction failure is silent — user-facing response is unaffected

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** S-003-003

**Preconditions:**
- Memory extraction step (step 7) is stubbed to throw an error during this test
- AI provider returns a normal response

**Steps:**
1. Send a message that would normally produce extractable memory candidates.
2. Observe the chat response.
3. Inspect the pipeline log for a memory extraction failure entry.
4. Confirm no error message or visual change appears to the user.

**Expected Result:**
- The Noxis response streams and completes normally; the user sees no error, spinner change, or delayed response; the pipeline log contains a memory extraction failure entry; no memory items are written to Mem0 for this exchange.

**Failure Indicators:**
- The visible response is delayed or interrupted due to the memory extraction failure
- An error indicator appears in the UI
- The pipeline crashes rather than logging the failure silently

---

### TC-004-005-007: Steps 7 and 8 are dispatched asynchronously and do not block the response stream

**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-5
**Dependencies:** S-003-003, S-005-007

**Preconditions:**
- Memory extraction (step 7) and dashboard sync (step 8) are instrumented with timestamps in the pipeline log
- A baseline streaming latency is established for a message with async steps disabled

**Steps:**
1. Send a message under normal conditions (steps 7 and 8 enabled).
2. Record the time from message send to first SSE token (time-to-first-token).
3. Record the time from first SSE token to final `event: done`.
4. Compare with baseline latency.
5. Inspect the pipeline log to confirm steps 7 and 8 start after the stream completes (not before or during).

**Expected Result:**
- Time-to-first-token and total streaming duration are within an acceptable range of the baseline (no statistically significant increase attributable to async steps); pipeline log shows step 7 and step 8 dispatch timestamps occurring after the `event: done` SSE event.

**Failure Indicators:**
- Steps 7 or 8 add measurable latency to the visible stream
- Pipeline log shows steps 7 or 8 blocking the SSE connection before `event: done`
- Any user-visible delay or pause in the token stream attributable to these steps

---

## Automation Notes

- TC-004-005-001 (full pipeline E2E) should use a deterministic mock AI provider that returns a fixed response; use log assertions to verify prompt layer order and telemetry fields rather than inferring from UI alone.
- TC-004-005-002 (memory cap) is best tested as a server-side unit test on `DecisionPolicyFilter` with 30 synthetic memory items; assert output count ≤ 20.
- TC-004-005-003 (Mem0 timeout) requires a `URLProtocol` stub on the Mem0 endpoint that delays response by 6 seconds; assert pipeline completion time is ≥ 5s and ≤ 10s.
- TC-004-005-004 (AI provider failure) should be automated with a stub that returns errors for 3 consecutive calls, then succeeds on the 4th; assert the client receives `{ "error": "response_unavailable" }` after the first 3.
- TC-004-005-005 (context trimming) is a unit test for `PromptTruncator`; supply a synthetic prompt over the limit and assert the trimmed output token count and content preservation.
- TC-004-005-006 and TC-004-005-007 require integration test setup with injectable async steps; assert step completion timestamps from the log.
- Never assert on system prompt content in UI tests — pipeline internals must not appear in any UI component.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

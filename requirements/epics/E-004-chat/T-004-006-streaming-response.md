# QA Test Plan — Streaming Response Display

---

**Plan ID:** T-004-006
**Story:** S-004-006
**Epic:** E-004
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

Validates the client-side streaming response display: typing indicator appearance and minimum 300ms duration, seamless transition from indicator to first token, in-place token appending without list disruption, input bar disabled state during streaming, partial response preservation and "Continue" button on stream interruption, resumable generation via backend resume endpoint, final delivered state commit to local database, and input bar re-enable within 100ms of stream completion. Does not cover the backend SSE endpoint implementation or the "Regenerate" button (which is explicitly excluded from v1).

## Out of Scope

- Backend SSE endpoint implementation (only client-side rendering is in scope)
- "Regenerate" button — deliberate product exclusion in v1; must not be implemented or tested
- "Stop generating" button — Future Scope
- Animated fade-in per word text appearance — Future Scope
- Response speed control — Future Scope

## Prerequisites

- S-004-001 deployed: `MessageRowView` and `ChatInputBar` exist so streaming can update an existing row
- S-004-005 deployed: SSE streaming endpoint is operational and sends `data: { "token": String }` chunks and a final `event: done`
- Backend supports `POST /api/v1/chat/resume` with `{ thread_id, message_id, last_token_index }` for resumable generation tests
- A mechanism to interrupt the SSE stream mid-response is available for interruption tests (network proxy, airplane mode, or backend stub)
- Local database (ThreadRepository) is instrumented so `updateMessage` calls can be verified in tests
- Authenticated user session active

---

## Core Test Flow

### TC-004-006-001: Full streaming happy path covering all acceptance criteria

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-004-001, S-004-005

**Preconditions:**
- Network is stable
- AI provider returns a response of at least 20 tokens so streaming is clearly observable
- Test is able to measure timing (use instrumented build or system timestamp logging)

**Steps:**
1. Send a message in the chat.
2. Immediately observe the Noxis message position in the chat list.
3. Time how long the typing indicator is visible before the first token appears.
4. Observe the transition from typing indicator to first streaming token.
5. Watch tokens appear in the message row as streaming progresses.
6. While streaming is in progress, observe the send button and image button in the input bar.
7. While streaming is in progress, type text into the `GlassInput` field.
8. Attempt to tap the send button while streaming.
9. Wait for streaming to complete (final `event: done`).
10. Observe the input bar state immediately after completion.
11. Verify the message status and database record after streaming ends.

**Expected Result:**
- Step 2: `NoxisAvatar` mark appears immediately in the Noxis message position, followed by three pulsing animated dots (typing indicator).
- Step 3: Typing indicator is visible for at least 300ms before the first token replaces it — even if the first token arrives in under 300ms.
- Step 4: Typing indicator is replaced directly by the first token text; no blank white flash, no empty bubble frame, no visual gap between indicator and text.
- Step 5: Tokens append to the same message row in real time; no new rows are created; no visible scroll position jump; text uses the same Body typography as a completed Noxis message.
- Steps 6-8: Send button and image button are visually muted (opacity 0.3) and non-interactive; the `GlassInput` field accepts typed text but the send button does not activate or submit.
- Step 10: Input bar re-enables (send button returns to full opacity if field is non-empty) within 100ms of the `event: done` SSE event.
- Step 11: Message `status` is `.delivered`; `ThreadRepository.updateMessage` has been called with the full response content; the `updatedAt` timestamp on the thread record is refreshed.

**Failure Indicators:**
- Typing indicator appears for less than 300ms even when the first token arrives early
- A blank flash or empty frame is visible between the indicator disappearing and text appearing
- Each token creates a new message row rather than updating the existing one
- The message list scroll position jumps or resets when tokens are appended
- Send button is active and submittable while streaming is in progress
- Input bar re-enables more than 100ms after `event: done`
- Database record is missing or has `status: .sending` after streaming completes
- A "Regenerate" button appears anywhere in the UI

---

## Sub Flows

### TC-004-006-002: Stream disconnects mid-response with ≥ 10 tokens — partial text preserved and "Continue" shown

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-5, AC-6
**Dependencies:** S-004-005

**Preconditions:**
- SSE stream can be interrupted after at least 10 tokens have been received (use network proxy or airplane mode on a real device, or a backend stub that drops the connection after N tokens)
- Backend supports resumable generation (`/api/v1/chat/resume`)

**Steps:**
1. Send a message that triggers a long Noxis response.
2. After at least 10 tokens have streamed in, interrupt the network connection.
3. Observe the Noxis message row.
4. Observe any UI control that appears below the partial response.
5. Restore network connectivity.
6. Tap the "Continue" button.
7. Observe the streaming behavior after tapping Continue.

**Expected Result:**
- Step 3: The partial text received so far is preserved and visible in the message row; no text disappears or is replaced with an error state.
- Step 4: A "Continue" button appears directly below the partial text in the Noxis message position; no inline error message is shown (the partial text itself is the signal that the response was cut short).
- Step 7: Streaming resumes from the last received token; text continues appending to the same message row (no duplicate content, no restart from the beginning); the response completes normally and transitions to `status: .delivered`.

**Failure Indicators:**
- Partial text disappears on stream disconnection
- No "Continue" button appears
- "Continue" restarts the response from the beginning (not resuming)
- Duplicate content appears when resuming (tokens already received are re-appended)
- Backend resume endpoint returns an error that is exposed to the user

---

### TC-004-006-003: SSE connection drops before any token arrives — typing indicator replaced by inline error

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-004-005

**Preconditions:**
- SSE connection is stubbed or the network drops before any `data:` line is received by the client

**Steps:**
1. Send a message.
2. Observe the typing indicator appearing.
3. Drop the network before any token arrives.
4. Observe the Noxis message position.
5. Observe the input bar.

**Expected Result:**
- Step 4: The typing indicator (pulsing dots) is replaced by an inline error message: "Noxis couldn't respond — try again"; the `NoxisAvatar` mark remains visible beside the error text.
- Step 5: The send button re-enables immediately (within the normal input re-enable window); the user can compose and send a new message without restarting the app.

**Failure Indicators:**
- Typing indicator spins indefinitely with no timeout or error state
- Error message does not appear
- Send button remains disabled after the connection failure
- The exact specified copy is not used (different error wording)
- App crashes or enters a broken state

---

### TC-004-006-004: Backend returns an error event in the SSE stream — streaming stops and inline error is shown

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-004-005

**Preconditions:**
- Backend SSE stub is configured to send some tokens followed by `event: error\ndata: rate_limited` mid-stream

**Steps:**
1. Send a message to trigger the stubbed error stream.
2. Observe tokens appearing in the message row.
3. After the `event: error` is received, observe the message row state.
4. Observe the input bar.

**Expected Result:**
- Step 3: Streaming stops at the error event; any partial text received before the error is preserved and displayed in the message row; an inline error message is appended or shown below the partial text; the error message does not contain the raw SSE event data (e.g. "rate_limited" is not shown verbatim to the user).
- Step 4: Input bar re-enables immediately after the error event is processed.

**Failure Indicators:**
- Raw error event data ("rate_limited" or similar) is displayed directly to the user
- Streaming continues after an error event
- Partial text is erased on error
- Input bar remains disabled after the error event

---

### TC-004-006-005: Very short response (1-3 tokens) — typing indicator still shows for exactly 300ms

**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-1 (300ms minimum)
**Dependencies:** S-004-005

**Preconditions:**
- AI provider or SSE stub is configured to return a 1-3 token response that arrives in under 100ms

**Steps:**
1. Send a message with the fast/short response stub active.
2. Measure the time the typing indicator is visible before text appears.
3. Observe the final message row after streaming completes.

**Expected Result:**
- Typing indicator remains visible for at least 300ms even though all tokens have already arrived; after 300ms the message transitions from indicator to final text in a single render; no jarring instant appearance occurs.

**Failure Indicators:**
- Typing indicator flashes and disappears in under 300ms
- Text appears before the 300ms minimum has elapsed
- No typing indicator is shown at all for short responses

---

### TC-004-006-006: User scrolls up during streaming — auto-scroll stops, chevron appears, stream continues

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** S-004-001

**Preconditions:**
- A long enough response is streaming that the user has time to scroll before it completes
- At least 4 prior messages exist to allow scrolling up past the 3-message threshold

**Steps:**
1. Send a message that produces a long response (10+ seconds of streaming).
2. While streaming is in progress, scroll up more than 3 messages from the bottom.
3. Observe whether auto-scroll continues firing during streaming.
4. Observe the "scroll to bottom" chevron button.
5. Observe whether streaming continues while the user is scrolled away.
6. Tap the chevron button.
7. Observe the scroll position and the message row.

**Expected Result:**
- Step 3: Auto-scroll stops once the user has scrolled up; the streaming message row continues to update in place but the view does not jump the user back to the bottom.
- Step 4: The "scroll to bottom" chevron button appears at the bottom right.
- Step 5: Streaming continues; new tokens append to the Noxis message row; the user simply cannot see it because they have scrolled away.
- Steps 6-7: Tapping the chevron smoothly scrolls to the bottom; the fully or partially streamed message is visible; streaming (if still in progress) continues appending tokens.

**Failure Indicators:**
- Auto-scroll keeps firing and forces the user's view back to the bottom while they are scrolling away
- Streaming halts or errors when the user scrolls away from the streaming row
- Chevron button does not appear when scrolled away during streaming
- The streaming message row loses its accumulated text when the view scrolls back to it

---

## Automation Notes

- TC-004-006-001 should be automated with XCUITest; use a mock SSE server via `URLProtocol` that sends tokens at a controlled rate; assert typing indicator duration ≥ 300ms with `XCTClockMetric` or a timestamp comparison.
- TC-004-006-002 (stream interruption + Continue) requires a proxy or `URLProtocol` that closes the connection after N chunks; automate the resume request assertion by intercepting the `/api/v1/chat/resume` call and verifying `last_token_index` matches the received count.
- TC-004-006-003 and TC-004-006-004 (connection drop / error event) should be unit-tested at the `StreamingMessageViewModel` layer — inject error events and assert `@Published` state transitions: `isStreaming = false`, inline error text set correctly.
- TC-004-006-005 (300ms minimum for short response) is best unit-tested by injecting a single-token SSE stream and asserting that `isTypingIndicator` remains true for 300ms before transitioning.
- TC-004-006-006 (scroll during stream) is best verified with a combined XCUITest + scroll simulation; assert the auto-scroll does not fire while `scrollOffset > 3 message heights` and that the chevron button appears.
- Never add a "Regenerate" button test — it is explicitly not in scope and must not exist in the UI.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

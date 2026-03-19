# QA Test Plan — Chat Interface: Send & Receive Messages

---

**Plan ID:** T-004-001
**Story:** S-004-001
**Epic:** E-004
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

Validates the core chat interface: empty state rendering, message send/receive display, input bar keyboard behavior, auto-scroll logic (including the "scroll to bottom" chevron), send button guard on empty input, and swipe-to-dismiss keyboard. This plan does not cover streaming animation (S-004-006), image input (S-004-002), AI call logic (S-004-005), or conversation history switching (S-004-004).

## Out of Scope

- Streaming text animation (S-004-006)
- Conversation threading or history switching (S-004-004)
- Image send flow beyond camera button placeholder (S-004-002)
- AI call logic and orchestration (S-004-005)
- Reactions on messages (long-press haptic menu) — Future Scope
- Message search within a conversation — Future Scope
- Swipe-to-reply for threading — Future Scope

## Prerequisites

- S-007-003 design system components deployed: `MessageBubble`, `GlassInput`, `NoxisAvatar`
- S-002-001 SOUL.md loaded so that Noxis can generate a real response
- S-002-002 tone mode is configured for the test user
- Test device running the app with a valid authenticated session
- A clean conversation state (no prior messages) for empty-state tests

---

## Core Test Flow

### TC-004-001-001: Full send-and-receive happy path covering all acceptance criteria

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7, AC-8
**Dependencies:** S-007-003, S-002-001, S-002-002

**Preconditions:**
- User is authenticated and on the Chat tab
- No prior messages exist in the conversation (fresh session)
- Network is reachable
- Backend can return a Noxis response

**Steps:**
1. Open the Chat tab with no prior conversation history.
2. Observe the screen before typing anything.
3. Tap the input field to open the keyboard.
4. Observe the input bar position relative to the keyboard.
5. Type "Hey Noxis, how's my progress looking?" in the `GlassInput` field.
6. Observe the send button state.
7. Tap the Send button.
8. Observe the message list immediately after tapping.
9. Wait for Noxis to respond.
10. Observe the Noxis response layout.
11. Scroll up at least 4 messages from the bottom (requires 4+ messages; send additional messages if needed to reach this threshold).
12. Trigger a new Noxis message (send another message).
13. Observe whether the view auto-scrolls or shows a chevron button.
14. Swipe down on the message list while the keyboard is open.
15. Observe keyboard dismissal behavior.

**Expected Result:**
- Step 2: Empty state is displayed — `NoxisAvatar` is centered on screen, muted placeholder text reads "What's on your mind?", no conversation history UI elements are visible.
- Step 4: The `ChatInputBar` sits flush above the keyboard top edge; no content overlap, no layout jump, no gap.
- Step 6: Send button is active (not muted) because the field contains non-whitespace text.
- Step 8: The user's message appears immediately as a right-aligned glass bubble; the `GlassInput` field clears to empty.
- Step 10: Noxis response appears left-aligned as plain text with no bubble background; `NoxisAvatar` mark is visible to the left of the first line only; subsequent lines of the same response are not prefixed with the avatar.
- Step 13: Because the user scrolled up more than 3 messages, auto-scroll does not occur; a "scroll to bottom" chevron button is visible at the bottom right of the message list.
- Step 15: Keyboard dismisses interactively, following the swipe velocity; it does not snap closed instantly.

**Failure Indicators:**
- Empty state shows chat history UI (headers, message rows) when there are no messages
- Input bar overlaps message content or jumps when keyboard appears
- User message does not appear until server response returns (missing optimistic display)
- `GlassInput` field is not cleared after send
- Noxis message has a bubble background applied
- `NoxisAvatar` mark appears on every line of a multi-line Noxis response
- Auto-scroll fires even though user had scrolled past the 3-message threshold
- Chevron button does not appear when user is scrolled up more than 3 messages
- Keyboard dismisses with an abrupt snap rather than following the swipe gesture

---

## Sub Flows

### TC-004-001-002: Send button is disabled and muted on empty or whitespace-only input

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-7
**Dependencies:** S-007-003

**Preconditions:**
- User is on the Chat tab
- `GlassInput` field is empty (default state)

**Steps:**
1. Observe the send button with the input field empty.
2. Tap the send button while it is muted.
3. Type a message consisting entirely of spaces (e.g. "     ").
4. Observe the send button state.
5. Attempt to tap the send button.
6. Delete all whitespace from the field.
7. Type a single character "a".
8. Observe the send button state.

**Expected Result:**
- Steps 1-2: Send button is visually muted and non-interactive; tapping it has no effect; no message appears in the list.
- Steps 3-5: Whitespace-only input does not activate the send button; it remains muted; tapping produces no message.
- Steps 7-8: Send button becomes active (fully opaque, tappable) after any non-whitespace character is entered.

**Failure Indicators:**
- Send button is active with an empty field
- A whitespace-only message is submitted and appears in the chat list
- Send button does not activate after a real character is typed

---

### TC-004-001-003: Network failure after optimistic message display shows inline error with Retry

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-007-003, S-002-001

**Preconditions:**
- User is on the Chat tab with at least one prior message
- Network can be interrupted (use airplane mode or a proxy rule to simulate failure after the message is dispatched)

**Steps:**
1. Type a message "Test network failure" in the input field.
2. Immediately after tapping Send, enable airplane mode (or block the outbound request via proxy).
3. Observe the message list.
4. Wait 5 seconds for the request to time out or fail.
5. Observe the state of the user's message bubble and the input bar.
6. Tap the "Retry" tap target that appears beneath the failed message.
7. Restore network connectivity before or during the retry.
8. Observe the outcome after retry.

**Expected Result:**
- Step 3: The user's message appears immediately as a right-aligned glass bubble (optimistic display) with a `status: .sending` indicator (e.g. a subtle spinner or pending icon).
- Step 5: After failure, the message bubble remains visible in the list; an inline error indicator appears beneath it; a "Retry" tap target is visible; the `GlassInput` input bar re-enables so the user can type.
- Step 8: After successful retry, the error indicator disappears and the message transitions to the delivered state; Noxis responds normally.

**Failure Indicators:**
- The user's message disappears from the list on failure
- No error indicator or Retry option appears beneath the failed message
- Input bar remains disabled after the failure, blocking further input
- The raw network error string is surfaced to the user

---

### TC-004-001-004: New message while previous response is still streaming is queued

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-004-005, S-004-006

**Preconditions:**
- User is in an active chat where a Noxis response is currently streaming
- The streaming response takes at least 3 seconds to complete

**Steps:**
1. Send a message that triggers a slow Noxis response (e.g. a complex question).
2. While the response is streaming (before it completes), type a second message in the input field.
3. Tap Send on the second message.
4. Observe the input bar and the second message behavior.
5. Wait for the first response to complete.
6. Observe the second message and Noxis's subsequent response.

**Expected Result:**
- Step 3-4: The second message is accepted; a brief "sending..." indicator appears in the input bar confirming it is queued; the second message does not interrupt or replace the ongoing stream.
- Steps 5-6: After the first response completes, the second message is sent to the pipeline and a new Noxis response begins streaming below the first response.

**Failure Indicators:**
- The second message interrupts the ongoing stream
- No "sending..." indicator is shown for the queued message
- The second message is silently dropped
- Two simultaneous streaming responses appear in the chat list

---

### TC-004-001-005: Very long pasted message (over 2000 characters) is accepted and displayed in full

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** S-007-003

**Preconditions:**
- A text string of at least 2100 characters is available on the clipboard
- User is on the Chat tab

**Steps:**
1. Tap the `GlassInput` field.
2. Paste the 2100-character string.
3. Observe the input field rendering.
4. Tap Send.
5. Observe the resulting message bubble in the chat list.

**Expected Result:**
- Step 3: The `GlassInput` field expands up to 5 lines and then scrolls internally; no truncation or rejection of the pasted text occurs in the input field.
- Step 5: The full 2100-character message appears in the right-aligned glass bubble; text is not truncated; the bubble expands vertically to contain the full content.

**Failure Indicators:**
- Input field silently truncates the pasted string
- Message bubble shows a "Read more" expand control or cuts off text
- App crashes or freezes when handling a large paste

---

### TC-004-001-006: Landscape rotation adapts input bar and message list; bubble max-width is capped

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** S-007-003

**Preconditions:**
- At least 3 messages are visible in the chat list (mix of user and Noxis messages)
- Device supports rotation

**Steps:**
1. Observe the chat in portrait orientation; note the user bubble widths relative to screen width.
2. Rotate the device to landscape orientation.
3. Observe the message list layout, input bar, and bubble widths.
4. Rotate back to portrait.
5. Observe that layout returns to normal.

**Expected Result:**
- Step 3: Message list reflows to the new width without wrapping issues; the input bar remains pinned to the bottom above the safe area; user message bubbles do not exceed 75% of the landscape screen width; Noxis text reflows naturally within the new width.
- Step 5: Layout returns to portrait dimensions correctly; no visual artifacts or misaligned elements.

**Failure Indicators:**
- Message bubbles exceed 75% screen width in either orientation
- Input bar is not visible or detaches from the bottom in landscape
- Message list does not reflowing — content is clipped or overflows

---

### TC-004-001-007: "Scroll to bottom" chevron dismisses and auto-scroll resumes when user scrolls back down

**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-6
**Dependencies:** S-007-003

**Preconditions:**
- At least 10 messages exist in the chat so that scrolling up more than 3 messages is possible
- A response is expected (network available)

**Steps:**
1. Scroll up more than 3 messages from the bottom.
2. Send a message that will produce a Noxis response.
3. Confirm the chevron button appears at bottom right.
4. Tap the chevron button.
5. Observe scroll behavior and chevron visibility.
6. Scroll back up more than 3 messages again.
7. Wait for another Noxis response to arrive.
8. Manually scroll back down to the bottom without using the chevron.
9. Observe whether a subsequent new message auto-scrolls.

**Expected Result:**
- Step 3: Chevron button is visible; the view does not auto-scroll to the new message.
- Steps 4-5: Tapping the chevron smoothly scrolls the list to the latest message; the chevron button disappears once the bottom is reached.
- Steps 8-9: After the user manually returns to the bottom, the next arriving message triggers auto-scroll again; chevron does not reappear unless the user scrolls up again.

**Failure Indicators:**
- Chevron button does not appear when scrolled past the threshold
- Tapping the chevron does not scroll to the latest message
- Chevron remains visible after the user has returned to the bottom
- Auto-scroll permanently disabled after the first manual scroll-up

---

## Automation Notes

- TC-004-001-001 is suitable for XCUITest E2E automation covering the main happy path; mock the network response to ensure deterministic Noxis reply content.
- TC-004-001-002 (empty/whitespace guard) should be unit-tested at the `ChatInputBar` view model layer — verify `isSendEnabled` returns false for empty and whitespace strings.
- TC-004-001-003 (network failure) requires a network simulation layer or `URLProtocol` stub; automate as an integration test.
- TC-004-001-004 (queued message) requires timing control; use a mock `OrchestrationPipeline` that enforces an artificial delay.
- TC-004-001-005 and TC-004-001-006 are best verified manually on device due to paste/clipboard and rotation constraints; screenshot snapshots are useful for regression.
- TC-004-001-007 (chevron behavior) can be automated with `ScrollViewReader` scroll-position assertions in XCUITest.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

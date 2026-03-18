# Story: Streaming Response Display

---

**ID:** S-004-006
**Epic:** E-004
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis's responses to appear word by word as they are generated**, so that **the conversation feels alive and immediate — not like waiting for a document to load**.

## Goal

Noxis responses stream token by token from the moment the first word is available. A subtle typing indicator precedes the first token. Text flows in naturally without a loading spinner. If the stream is interrupted mid-response, the partial text is preserved and a "Continue" tap option is shown.

## Why / Rationale

The latency of generating a full AI response (2-8 seconds) feels like a dead wait if nothing appears. Streaming makes the same latency feel like responsiveness. It is the difference between a conversation and a query-response system. This is a perceived-performance requirement as much as a functional one.

## Functional Requirements

- When the orchestration pipeline begins generating a response, a typing indicator appears immediately in Noxis's position in the chat list: three animated dots at the same size and position as a Noxis message — the `NoxisAvatar` mark appears, then the dots pulse
- Typing indicator is shown for a minimum of 300ms regardless of how fast the first token arrives — prevents a jarring flash
- As tokens arrive via SSE, each token is appended to the Noxis message `@State` string and re-renders in place — no new message rows are added; the existing row updates in real time
- Text renders with the same typography and layout as a completed Noxis message — no visual difference between streaming and final state
- The send button and image button in the input bar are disabled and visually muted while a response is streaming; the input field accepts typing but the send button does not activate until streaming completes
- If the SSE stream is interrupted (network drop, server timeout, or user force-quits and returns), the partial response text is preserved and displayed with a "Continue" button below it — tapping resumes from where the response left off (backend must support resumable generation via the `last_token_index` parameter)
- When streaming completes, the message transitions to `status: .delivered` and is committed to the local database; the input bar re-enables
- No "Regenerate" button in v1 — do not add it

## Prerequisites

- S-004-001 (chat interface — the message row must exist before streaming can update it)
- S-004-005 (orchestration pipeline — streaming tokens originate from the SSE endpoint)

## Acceptance Criteria

- [ ] Given the user sends a message and the pipeline begins, when the first token has not yet arrived, then the typing indicator (NoxisAvatar + three pulsing dots) is visible in the Noxis message position for at least 300ms
- [ ] Given the first SSE token arrives, when the typing indicator is still visible, then the indicator is replaced by the streaming text starting from that first token — no blank flash between indicator and text
- [ ] Given tokens are arriving via SSE, when each token is appended, then the message text updates in place without creating new list rows or causing scroll position jumps
- [ ] Given a response is streaming, when the send button is checked, then it is visually muted and non-interactive; the input field accepts typing but does not submit
- [ ] Given the SSE stream disconnects mid-response, when the partial text is at least 10 tokens long, then the partial text is preserved and a "Continue" button appears below it
- [ ] Given the user taps "Continue" on an interrupted response, when the request is sent, then the backend resumes generation from the last received token index and the text continues appending
- [ ] Given streaming completes, when the final token is received, then the message status transitions to delivered, the local database record is written, and the input bar re-enables within 100ms

## Negative Scenarios

- SSE connection drops before any token is received → the typing indicator is replaced by an inline error message: "Noxis couldn't respond — try again"; the send button re-enables immediately
- Backend returns an error event in the SSE stream (e.g. `event: error\ndata: rate_limited`) → streaming stops, partial text (if any) is preserved, an inline error message is shown, the input re-enables

## Edge Cases

- Response is extremely short (1-3 tokens) → the typing indicator shows for exactly 300ms even if all tokens arrive before that; prevents a visually jarring instant appearance
- User scrolls up while streaming is in progress → auto-scroll stops; the "scroll to bottom" chevron appears; streaming continues appending to the message invisibly; user can tap the chevron to jump back down

## Dependencies

- S-004-001 (chat interface)
- S-004-005 (orchestration pipeline)

## Technical Requirements

- **SSE client:** `URLSession` streaming response with `delegate: URLSessionDataDelegate`; `urlSession(_:dataTask:didReceive:)` processes each `data:` line; parses JSON `{ "token": String }` chunks
- **`StreamingMessageViewModel.swift`** — `@Published var streamingText: String = ""`; accumulates tokens; `@Published var isStreaming: Bool`; `@Published var isTypingIndicator: Bool`
- **Typing indicator timing:** `DispatchQueue.main.asyncAfter(deadline: .now() + 0.3)` before hiding the indicator and showing first token; `isTypingIndicatorMinimumMet: Bool` flag
- **In-place update:** each `MessageRowView` observes its `ChatMessage` object via `@ObservedObject`; when `content` changes, the view re-renders without the list needing to reload
- **Continue / resumable generation:** `POST /api/v1/chat/resume` with `{ thread_id, message_id, last_token_index }`; the client tracks `tokenIndex` as a counter on each received chunk
- **Disable input during stream:** `ChatInputBar` receives `@Binding var isStreaming: Bool`; send button: `.disabled(isStreaming)`, `.opacity(isStreaming ? 0.3 : 1.0)`
- **Database commit:** on `event: done` SSE event, `ThreadRepository.updateMessage(id:, content:, status: .delivered)` is called; triggers `updatedAt` refresh on the thread record

## Future Scope

- Animated text appearance (subtle fade-in per word rather than instant append) for a more premium feel
- "Stop generating" button that cancels the stream and commits what has arrived
- Response speed control (faster/slower streaming rate for users who find the default too slow or too fast)

## Do Not Do

- In this story, do not add a "Regenerate" button — deliberate product decision; Noxis does not invite second-guessing its responses
- Do not implement the backend SSE endpoint here — only the client-side rendering and the state model
- Do not add any loading spinner — the typing indicator is the only waiting affordance

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

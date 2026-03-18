# Story: Chat Interface — Send & Receive Messages

---

**ID:** S-004-001
**Epic:** E-004
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **to send messages to Noxis and receive responses in a premium chat interface**, so that **every interaction feels intentional, private, and like talking to something built specifically for me**.

## Goal

A fully functional chat screen is implemented: user messages appear as right-aligned glass bubbles, Noxis responses appear as left-aligned plain text (no bubble), the input bar handles keyboard interactions gracefully, and the conversation auto-scrolls to the latest message. New conversations begin with a purposeful empty state.

## Why / Rationale

This is the primary product surface. Before memory, modules, or daily briefs have value, the chat interface must feel premium and respond correctly. A broken or generic-feeling chat UI undermines the entire product promise. This ships first.

## Functional Requirements

- User messages are displayed as right-aligned glass bubble components (`MessageBubble` from S-007-003) with body-sized text and a timestamp beneath on long-press
- Noxis messages are displayed left-aligned with no background — text only, using the Body typography style, with the `NoxisAvatar` mark to the left of the first line only
- Input bar is pinned to the bottom of the screen, above the safe area, and rises with the keyboard without content shifting
- Keyboard dismisses on swipe down on the message list (interactive dismissal)
- Message list auto-scrolls to the bottom when a new message arrives; if the user has scrolled up more than 3 messages from the bottom, auto-scroll does not interrupt — a "scroll to bottom" chevron button appears instead
- Empty state shown when no messages exist: centered `NoxisAvatar`, a single line of muted text ("What's on your mind?"), no other UI clutter
- Input field is a `GlassInput` component accepting multi-line text, expanding up to 5 lines before scrolling internally
- Send button is disabled and visually muted when input field is empty; activates with any non-whitespace text
- Image attachment button (camera icon) sits left of the text field in the input bar — tapping is handled by S-004-002
- Sending a message appends it immediately to the list (optimistic display) before the server response begins

## Prerequisites

- S-007-003 (design system — MessageBubble, GlassInput, NoxisAvatar components must exist)
- S-002-001 (SOUL.md load — required before any real response can be generated)
- S-002-002 (tone mode — determines response character)

## Acceptance Criteria

- [ ] Given the user opens the Chat tab with no prior conversation, when the screen renders, then the empty state is shown with NoxisAvatar centered and muted placeholder text — no chat history UI
- [ ] Given the user types a message and taps Send, when the message is sent, then it appears immediately as a right-aligned glass bubble and the input field clears
- [ ] Given Noxis is generating a response, when the response arrives, then it appears left-aligned as plain text with the NoxisAvatar mark on the first line and no bubble background
- [ ] Given the keyboard opens, when the user is composing a message, then the input bar rises exactly to the top of the keyboard with no content overlap or jump
- [ ] Given the user is scrolled to the bottom and a new message arrives, when the list updates, then the view automatically scrolls to show the new message
- [ ] Given the user has scrolled up more than 3 messages from the bottom and a new message arrives, when the list updates, then auto-scroll does not occur and a "scroll to bottom" chevron button appears at the bottom right
- [ ] Given the input field is empty or contains only whitespace, when displayed, then the send button is visually muted and non-interactive
- [ ] Given the user swipes down on the message list while the keyboard is open, when the gesture is recognized, then the keyboard dismisses interactively following the swipe

## Negative Scenarios

- Network request fails after optimistic message display → the user message remains visible; an inline error indicator appears beneath it with a "Retry" tap target; the input bar re-enables
- User sends a message while a previous response is still streaming → the new message is queued and sent after the current response completes; a brief "sending..." indicator shows in the input bar

## Edge Cases

- User pastes a very long message (over 2000 characters) → the message is accepted and displayed in full; the bubble expands naturally; no truncation in v1
- Screen is rotated to landscape → input bar and message list adapt to the new width; bubble max-width is capped at 75% of screen width in both orientations

## Dependencies

- S-007-003 (design system components)
- S-002-001 (SOUL.md system prompt)
- S-002-002 (tone mode selection)

## Technical Requirements

- **`ChatView.swift`** — main SwiftUI view; `ScrollViewReader` with `scrollTo` on new message IDs; `@FocusState` for keyboard management
- **`MessageListView.swift`** — `LazyVStack` inside `ScrollView` for performance; each message is a `MessageRowView` keyed by message ID
- **`ChatInputBar.swift`** — `HStack` with `GlassInput`, camera button (SF Symbol `camera`), and send button; uses `.toolbar(.hidden, for: .navigationBar)` to maintain full bleed
- **Keyboard avoidance:** `.ignoresSafeArea(.keyboard)` on the scroll view + `safeAreaInset(edge: .bottom)` for the input bar — avoids the broken `padding` approach
- **Optimistic message model:** `ChatMessage` struct with `id: UUID`, `role: MessageRole`, `content: String`, `status: MessageStatus (.sending | .delivered | .failed)`, `timestamp: Date`
- **Scroll-to-bottom detection:** track `scrollOffset` via `PreferenceKey`; show chevron button when offset > 3 message heights from bottom
- **Empty state:** conditional render on `messages.isEmpty`; no separate view file needed

## Future Scope

- Reactions on messages (long-press → haptic menu)
- Message search within a conversation
- Swipe-to-reply for threading

## Do Not Do

- In this story, do not implement streaming text animation — that is S-004-006
- Do not implement conversation threading or history switching — that is S-004-004
- Do not implement the image send flow beyond displaying the camera button placeholder — that is S-004-002
- Do not implement any AI call logic — that is S-004-005

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

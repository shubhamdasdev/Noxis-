# Story: Side Panel — Conversation History & Quick-Jump

---

**ID:** S-007-002
**Epic:** E-007
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a swipeable side panel that shows my conversation history and quick-jumps to any module**, so that **I can find past advice and navigate the app without leaving the chat flow**.

## Goal

A ChatGPT-style side panel slides in from the left edge of the Chat screen on swipe-right or hamburger tap. It shows a scrollable list of past conversations grouped by date, plus quick-jump buttons to each life module. The panel is glass + black, feels native and premium, and dismisses cleanly on swipe-left or background tap.

## Why / Rationale

As the user's chat history grows, being able to find past advice ("what did Noxis say about my blue suit?") becomes a retention driver. The side panel keeps this accessible without adding UI complexity to the main chat screen. Module quick-jumps make it a control center, not just a history list.

## Functional Requirements

**Opening:**
- Swipe right from the left edge of the Chat screen (gesture zone: 20pt from left edge)
- Tap the hamburger icon (top-left of Chat screen header)

**Panel content:**
- **Header:** "Noxis" wordmark + close (×) button
- **New Chat** button at top — starts a fresh conversation thread
- **Conversation list:** grouped by date (Today, Yesterday, This Week, Older); each entry shows the first user message truncated to ~40 chars and the timestamp; tapping navigates to that conversation in Chat
- **Module quick-jump section:** labelled "Modules" with icon + label buttons for each of the 5 life modules (Wardrobe, Gym, Food, Spending, Routines) — tapping navigates to that module in the Dashboard tab

**Closing:**
- Swipe left on the panel
- Tap anywhere on the dimmed overlay behind the panel
- Tap the × button

**Appearance:**
- Panel width: 80% of screen width
- Background: glass blur (dark) over a very dark backdrop
- Dimmed overlay (40% black) on the right side while panel is open
- Panel slides in with a spring animation (natural iOS feel)

## Prerequisites

- S-007-003 (Glass + Black Design System)

## Acceptance Criteria

- [ ] Given the user is on the Chat screen, when they swipe right from the left edge, then the side panel slides in with a spring animation
- [ ] Given the panel is open and the user taps a past conversation, when navigated, then the Chat screen loads that conversation thread
- [ ] Given the panel is open and the user taps a module quick-jump, when navigated, then the Modules tab opens to that specific module
- [ ] Given the panel is open and the user taps the dimmed overlay, when tapped, then the panel closes with a slide-out animation
- [ ] Given the user taps New Chat, when tapped, then a fresh conversation starts and the panel closes

## Negative Scenarios

- No conversation history yet (new user) → show an empty state: "Your conversations will appear here"
- Panel opened from a screen other than Chat → gesture only works on Chat screen; hamburger icon only appears on Chat screen

## Edge Cases

- Very long conversation list (100+ items) → panel list is virtualized; no scroll jank

## Dependencies

- S-007-003

## Technical Requirements

- **Custom drawer component** — SwiftUI `HStack` with a `.offset()` modifier driven by drag gesture; spring animation on release
- **Conversation storage** — conversation threads stored locally (Core Data or SQLite); each thread has `id`, `firstMessage`, `timestamp`, `messages[]`
- **Blur background** — `UIVisualEffectView` with `.systemThinMaterialDark` as panel background

## Future Scope

- Search within conversation history (v2)
- Pin/star important conversations (v2)

## Do Not Do

- In this story, do not implement conversation search
- In this story, do not show memory or profile information in the panel — that is S-007-005

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

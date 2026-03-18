# Story: Conversation History & Thread Management

---

**ID:** S-004-004
**Epic:** E-004
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **my conversations with Noxis to be organized into persistent threads that survive app restarts**, so that **I can return to an ongoing exchange, review past advice, and start fresh when I'm moving to a new topic**.

## Goal

Conversations are stored as named threads, each auto-titled from the first message. The user can browse past threads from the side panel, start a new thread from a clear CTA, and scroll up within an active thread to load earlier messages via pagination. Threads persist locally and sync to the backend.

## Why / Rationale

Without thread persistence, every app open starts a blank slate. Users lose context, Noxis loses continuity, and the product feels ephemeral rather than like a trusted advisor that remembers. Thread management is the infrastructure layer that makes repeated use valuable.

## Functional Requirements

- Every conversation is a `Thread` with: `id: UUID`, `title: String`, `createdAt: Date`, `updatedAt: Date`, `messageCount: Int`, `lastMessagePreview: String`
- Thread title is auto-generated: the first user message is truncated to 40 characters; if the user's first message is a photo, the title defaults to "Photo conversation — {date}"
- The side panel (S-007-002) shows a list of past threads sorted by `updatedAt` descending; each row shows the title, date, and last message preview
- "New Chat" button is accessible from the side panel and from a `+` icon in the navigation bar of the Chat screen; tapping creates a new thread and dismisses the panel
- The active (most recent) thread is loaded automatically when the user opens the Chat tab
- Within a thread, messages are paginated: the initial load shows the last 40 messages; scrolling to the top of the list triggers a load of the previous 40 messages — a loading indicator shows at the top while fetching
- Threads are stored in a local SQLite database (via GRDB or CoreData) and synced to the backend; offline reads always work; sends queue when offline
- Maximum thread age shown in history: 90 days; threads older than 90 days are archived and not shown by default
- Swiping left on a thread row in the side panel reveals a "Delete" action; delete is permanent after a confirmation alert
- Deleting a thread deletes all its messages from both local storage and the backend

## Prerequisites

- S-004-001 (chat interface — thread must have a screen to render into)
- S-007-002 (side panel — thread list lives here)

## Acceptance Criteria

- [ ] Given the user sends their first message in a new chat, when the thread record is created, then the thread title is set to the first 40 characters of that message (or "Photo conversation — {date}" for a photo-only first message)
- [ ] Given the user opens the app after previously closing it mid-conversation, when the Chat tab loads, then the most recent thread is restored with all messages visible
- [ ] Given the user opens the side panel, when the thread list is displayed, then threads are sorted newest-first with title, relative date, and last message preview on each row
- [ ] Given the user taps "New Chat" in the side panel, when the action triggers, then a new empty thread is created, the side panel closes, and the Chat screen shows the empty state
- [ ] Given the user scrolls to the top of a thread with more than 40 messages, when the top is reached, then a loading indicator appears and the next 40 older messages load and prepend to the list without scrolling the user's view position
- [ ] Given the user swipes left on a thread in the side panel and taps Delete, when the confirmation alert is accepted, then the thread and all its messages are removed from the local database and a backend delete request is dispatched
- [ ] Given the device is offline, when the user opens a previously loaded thread, then all cached messages are visible and a subtle "No connection" banner appears at the top

## Negative Scenarios

- Backend sync fails for a new thread → thread is created and stored locally; sync retries with exponential backoff on next network availability; user sees no error unless offline for >24 hours
- User tries to load a thread that was deleted on another device → local copy is removed silently on the next sync; user is redirected to the most recent remaining thread or the empty state

## Edge Cases

- User creates dozens of threads in rapid succession (stress test) → the list renders with `LazyVStack` pagination; no loading delay for the first 20 rows; remaining rows load on scroll
- First message in a new thread is exactly 40 characters → title is set to the full 40 characters with no ellipsis
- User renames a thread title manually (not in v1 but data model must support it) → `title` field is writable; `isCustomTitle: Bool` flag distinguishes auto vs. user-set

## Dependencies

- S-004-001 (chat interface)
- S-007-002 (side panel)

## Technical Requirements

- **Local database:** GRDB (Swift) with two tables: `threads (id, title, created_at, updated_at, message_count, last_preview, is_archived)` and `messages (id, thread_id, role, content, media_url, image_category, status, timestamp)`
- **`ThreadRepository.swift`** — data access layer; exposes `fetchThreads()`, `fetchMessages(threadId:, offset:, limit:)`, `createThread()`, `deleteThread(id:)`, `updateThread(id:, title:)`
- **Pagination:** `fetchMessages` uses SQL `LIMIT 40 OFFSET n`; UI tracks current offset per thread
- **Backend sync:** `POST /api/v1/threads`, `POST /api/v1/threads/{id}/messages`, `DELETE /api/v1/threads/{id}`; sync runs in background after each local write
- **Offline queue:** `SyncQueue` struct wrapping `[SyncOperation]` persisted to UserDefaults; processes on `NWPathMonitor` `isSatisfied` event
- **Auto-title generation:** runs in `ThreadRepository.createThread(firstMessage:)` — truncate to 40 chars, strip leading/trailing whitespace
- **90-day archive:** a background task (BGProcessingTask) runs nightly; threads where `updatedAt < Date() - 90 days` are flagged `is_archived = true`

## Future Scope

- User-editable thread titles
- Thread pinning (pin important conversations to the top of the list)
- Cross-device thread sync via iCloud in addition to the backend
- Thread tagging by module category (Wardrobe, Gym, etc.)

## Do Not Do

- In this story, do not implement memory extraction from conversations — that is S-003-003
- Do not implement search across threads in v1
- Do not implement thread sharing or export in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

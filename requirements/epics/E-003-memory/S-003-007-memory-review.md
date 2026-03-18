# Story: Memory Review in Profile

---

**ID:** S-003-007
**Epic:** E-003
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **to see what Noxis remembers about me and remove anything that's wrong or outdated**, so that **I stay in control of my own data and can correct the model when it has learned something incorrectly**.

## Goal

A Memory screen in the Profile tab shows all stored memory items grouped by category, with the content and date visible for each. Users can delete individual entries. Total memory count is shown. The screen is view-only except for deletion — no editing.

## Why / Rationale

Trust requires transparency. A user who knows Noxis is building a model of them needs the ability to inspect and correct that model. Without a review screen, the memory layer is a black box — and users who distrust black boxes will share less, which degrades the product. The Memory Review screen signals that Noxis is honest about what it knows and respects user control.

## Functional Requirements

- The Memory Review screen is accessible from the Profile tab (S-007-005) via a "What Noxis remembers" row or similar entry point
- The screen shows all memory entries for the authenticated user fetched via `MemoryService.listMemories`
- Memories are grouped into sections by category: Wardrobe, Gym, Food, Spending, Routines, Social, General — in that order
- Empty categories are not shown; if a category has zero items, the section is omitted
- Each memory row shows: the memory content (full text, wrapping to multiple lines), and the relative date (e.g., "3 days ago", "2 months ago")
- A header above the grouped list shows: "Noxis remembers [N] things about you" — total count across all categories
- Swiping left on a memory row reveals a "Forget" action button; tapping it triggers a confirmation alert: "Forget this? Noxis won't remember '[truncated content]' any more." with "Forget" (destructive) and "Cancel" options
- Confirmed deletion calls `MemoryService.deleteMemory(id)` and removes the row from the list with a smooth animation; if the deletion request fails, the row is restored and an inline error message appears
- Categories with only one item remaining show the section header without a count; categories collapse from the list if their last item is deleted
- The screen has a loading state (shimmer on the glass cards while the fetch completes) and an empty state (shown if the user has zero memories across all categories): "Noxis hasn't learned anything specific yet — it will as we talk"
- No edit functionality — content is read-only; only deletion is permitted
- The screen does not show memory confidence scores or source metadata — only human-readable content and date

## Prerequisites

- S-003-001 (memory storage — MemoryService.listMemories must be implemented)
- S-007-005 (Profile tab — entry point to this screen)

## Acceptance Criteria

- [ ] Given the user navigates to the Memory Review screen, when memories exist, then they are displayed grouped by category in the defined order with content and relative date visible for each entry
- [ ] Given the user navigates to the Memory Review screen, when zero memories exist across all categories, then the empty state is shown: "Noxis hasn't learned anything specific yet — it will as we talk"
- [ ] Given memories exist across 3 categories, when the screen renders, then only those 3 section headers appear — empty categories are not shown
- [ ] Given the user swipes left on a memory row and taps "Forget", when the confirmation alert is accepted, then the memory is deleted via MemoryService and the row is removed from the list with an animation
- [ ] Given the user taps "Forget" and the API call fails, when the failure occurs, then the row is restored to the list and an inline error message appears below the row
- [ ] Given the user deletes the last item in a category, when the deletion completes, then the section header for that category disappears from the list
- [ ] Given the total memory count is shown in the header, when a deletion completes, then the count decrements immediately to reflect the new total

## Negative Scenarios

- `MemoryService.listMemories` returns an error → the screen shows a full-screen error state: "Couldn't load memories — pull to refresh" with a `GlassButton` to retry
- User rapidly deletes many memories in quick succession → each deletion is queued and processed in order; the list updates optimistically and reconciles with server responses; overlapping deletion requests do not cause duplicated deletes

## Edge Cases

- A memory entry has very long content (250+ characters) → the row expands to show the full text; no truncation; the row height grows naturally
- User opens the Memory Review screen while a background memory extraction is in progress → the new memory may not be visible yet; the screen does not auto-refresh in real time; user can pull to refresh to see the latest state

## Dependencies

- S-003-001 (memory storage)
- S-007-005 (Profile tab)

## Technical Requirements

- **`MemoryReviewView.swift`** — SwiftUI `List` with `Section` per category; sections use `MemoryCategory.allCases` for ordering; populated via `@StateObject var viewModel: MemoryReviewViewModel`
- **`MemoryReviewViewModel.swift`** — `@Published var memoriesByCategory: [MemoryCategory: [Memory]]`; `@Published var totalCount: Int`; `func load()` calls `GET /api/v1/memory/all`; `func delete(memory: Memory)` calls `DELETE /api/v1/memory/{id}`
- **List animation:** `.animation(.spring(), value: memoriesByCategory)` on the outer `List`; individual row removal uses `withAnimation { memoriesByCategory[category]?.removeAll { $0.id == memory.id } }`
- **Relative date formatting:** `RelativeDateTimeFormatter` with `DateComponents` precision — ".day" for recent, ".month" for older
- **Confirmation alert:** `.alert(isPresented: $showDeleteAlert)` with `.destructive` button role
- **Optimistic deletion:** row is removed from local state immediately; if the API call fails, the memory is re-inserted at the same position via `insert(_:at:)` on the array
- **Loading state:** `MemoryRowShimmer` view using `GlassCard` with animated opacity pulse; shown while `isLoading: Bool` is true
- **Navigation entry:** `Profile` screen has a `NavigationLink` row labelled "What Noxis remembers" with the total memory count as a trailing label

## Future Scope

- Bulk delete by category ("Forget everything in Wardrobe")
- Memory correction: user can tap a memory to edit its content in-place rather than deleting and re-entering
- "Why does Noxis remember this?" — tapping an info button shows the source conversation snippet that generated the memory

## Do Not Do

- In this story, do not show confidence scores, source metadata, or memory IDs to the user
- Do not implement memory editing — view and delete only in v1
- Do not build the bulk delete feature in v1
- Do not show session log summaries on this screen — this screen is for extracted memory items only

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

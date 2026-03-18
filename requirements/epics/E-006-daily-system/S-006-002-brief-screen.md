# Story: Daily Brief Screen — Display & History

---

**ID:** S-006-002
**Epic:** E-006
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a dedicated Daily tab that shows today's brief and a history of past briefs**, so that **I can read what Noxis prepared for me each morning and look back at past insights to see how my patterns have evolved**.

## Goal

The Daily tab renders today's brief as a prominent glass card with the full insight and action items, plus a scrollable history of the last 30 days of briefs below. Each past brief shows date and first line. Unread briefs show a visual indicator. Tapping a past brief expands to full content.

## Why / Rationale

The brief needs its own screen — not a chat message, not a push notification with no destination. Users who engage with their brief need to be able to re-read it, refer back to it during the day, and see their recent history to notice their own patterns. The screen is what makes the brief a daily ritual rather than a one-time read.

## Functional Requirements

- The Daily tab is one of the bottom tab bar items (S-007-001)
- **Today's brief card** — shown at the top of the screen; a large `GlassCard` with:
  - Date header: "Today — [Day, Month Date]" in Caption typography, muted
  - Insight paragraph in Body typography
  - Action items as a bulleted list (3 bullets max) with a subtle left-accent bar in front of each item
  - Optional curated content card at the bottom of the brief (if present): shows title, type badge, and description; tapping opens the content in `SFSafariViewController` (for articles/URLs)
  - If the brief has not been viewed before (first open today), the `viewed_at` timestamp is set on the `daily_briefs` record when this card is first fully displayed (tracked via `onAppear`)
  - If no brief exists for today: shows a state card: "No brief today — keep using Noxis and tomorrow's brief will reflect your activity"
  - If brief generation is in progress (status: pending): shows a shimmer loading card of the same dimensions

- **Brief history** — a vertical list below today's card showing briefs for the last 30 days (excluding today); each row:
  - Date in Caption typography
  - First sentence of the insight, truncated with ellipsis
  - Unread indicator: a small accent-color dot on the left edge if `viewed_at` is null
  - Tapping a row expands it in-place (accordion style) to show the full insight, action items, and curated content if present; tapping again collapses

- The "Today" card and the history list are in the same `ScrollView` — the whole screen scrolls; the today card is not sticky or pinned
- Pull-to-refresh reloads the current brief status
- The screen title "Daily" is shown in Display typography in the navigation bar (large title style)

## Prerequisites

- S-006-001 (daily brief generation engine — briefs must be generated before this screen can display them)
- S-007-001 (bottom tab bar — Daily tab must exist)
- S-007-003 (design system — GlassCard, typography components)

## Acceptance Criteria

- [ ] Given a brief for today exists with `status: generated`, when the user opens the Daily tab, then today's full brief (insight + action items + optional curated content) is displayed in the top glass card
- [ ] Given the user opens the Daily tab for the first time today and the brief has not been viewed, when the today card is fully rendered on screen, then `viewed_at` is set on the `daily_briefs` record
- [ ] Given the history list renders, when it is displayed, then up to 30 past briefs are shown below the today card, sorted newest-first, each with truncated first sentence and date
- [ ] Given a past brief has `viewed_at: null`, when its row renders in the history list, then an accent-color dot indicator is visible on the left edge of the row
- [ ] Given the user taps a past brief row, when the tap is registered, then the row expands in-place to show the full insight, action items, and curated content
- [ ] Given no brief exists for today, when the Daily tab renders, then the state card "No brief today — keep using Noxis and tomorrow's brief will reflect your activity" is shown in place of the today card
- [ ] Given the curated content card is present, when the user taps it, then `SFSafariViewController` opens the linked URL in-app without leaving the app

## Negative Scenarios

- `GET /api/v1/briefs` fails → an inline error state is shown in place of both the today card and history list: "Couldn't load your brief — pull to refresh"
- Today's brief has `status: failed` (generation failed) → the today card shows: "Brief unavailable today — Noxis will try again tomorrow"; no broken states or error codes

## Edge Cases

- User has been inactive for 10+ days (no briefs generated) → the history list is empty below the today card; the today card shows the "no brief today" state; no empty list error state is needed
- Brief insight is unusually long (500+ characters) → the full text is shown without truncation; the today card height expands naturally; no character limit on display

## Dependencies

- S-006-001 (brief generation engine)
- S-007-001 (bottom tab bar)
- S-007-003 (design system)

## Technical Requirements

- **`DailyTabView.swift`** — `NavigationStack` with `.navigationTitle("Daily")` large title; contains a `ScrollView` with `TodayBriefCard` + `BriefHistoryList`
- **`TodayBriefCard.swift`** — `GlassCard` component; renders `BriefContent` struct (insight, action_items, curated_content); accepts `brief: DailyBrief?`, `isLoading: Bool`
- **`BriefHistoryList.swift`** — `LazyVStack` of `BriefHistoryRow` components; each row is expandable using `@State var expandedId: UUID?` in the parent view
- **`DailyBriefViewModel.swift`** — `@Published var todayBrief: DailyBrief?`; `@Published var historyBriefs: [DailyBrief]`; `func load()` → `GET /api/v1/briefs/today` + `GET /api/v1/briefs/history?days=30`
- **Viewed tracking:** `DailyBriefViewModel.markViewed(id:)` calls `PATCH /api/v1/briefs/{id}/viewed`; triggered from `TodayBriefCard.onAppear` only if `brief.viewed_at == nil`
- **Accordion expand:** `withAnimation(.spring(response: 0.3)) { expandedId = expandedId == id ? nil : id }` on row tap
- **Curated content:** `SFSafariViewController` presented via `.sheet(isPresented:)` wrapper when the content card is tapped
- **Pull-to-refresh:** `.refreshable { await viewModel.load() }`
- **Backend routes:** `GET /api/v1/briefs/today`, `GET /api/v1/briefs/history?days=30`, `PATCH /api/v1/briefs/{id}/viewed`

## Future Scope

- Brief sharing: long-press today's card to share a redacted version (insight only, no personal data) as a screenshot
- Brief bookmarking: pin specific past briefs to a "saved" section at the top of the history list
- Week-in-review card: every Monday, a slightly longer brief summarizing the full previous week appears as the today card

## Do Not Do

- In this story, do not pin or sticky the today card when scrolling — the whole screen scrolls as one
- Do not implement inline reply-to-brief in v1 (e.g., tapping to chat about the brief); navigation to Chat tab is sufficient
- Do not paginate beyond 30 days of history in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

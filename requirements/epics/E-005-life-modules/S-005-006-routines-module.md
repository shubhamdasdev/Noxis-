# Story: Routines Module — Habit Stacks & Streak Tracking

---

**ID:** S-005-006
**Epic:** E-005
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a routines module that tracks my daily habits with streaks**, so that **I can see which habits are holding and which are slipping — and Noxis can give me grounded consistency advice**.

## Goal

The Routines module shows defined habit stacks with individual streak counters, a quick-check button to mark habits as done for today, and a weekly consistency view. Habits are added manually or suggested by Noxis from chat conversations.

## Why / Rationale

Consistency is where most self-improvement efforts die. A man who wants to build a morning routine needs to see it in numbers, not just feel it. The routines module is the accountability layer — it transforms intentions into visible streaks that both the user and Noxis can reference. When Noxis knows the user skipped their morning cold shower three days running, it has something concrete to address.

## Functional Requirements

- The Routines module screen shows:
  1. **Today's habit stack** — a vertical checklist of all defined habits with a checkmark button on each; habits not yet checked today show as active; habits already checked today show with a filled checkmark and a muted style; the section header shows "Today — N/M complete"
  2. **Streak counters** — beneath each habit name, a small streak count (e.g., "12-day streak") using `StreakBadge` component
  3. **Weekly consistency view** — a 7-column grid (Mon–Sun) at the bottom; each cell is a small pill per habit checked on that day; fully green row = all habits done; partial = mixed; empty = no habits done

- **Add habit** — a `+` button in the navigation bar opens a sheet: habit name (required, max 60 chars), optional emoji icon, target frequency (Daily / Weekdays / Custom — custom not implemented in v1, just stored as "daily")
- Habits can also be suggested by Noxis from chat: when Noxis identifies a commitment in conversation ("I'll do cold showers every morning"), it surfaces a suggestion card: "Want to track this?" with Add / Dismiss; tapping Add pre-fills the add habit sheet
- Checking a habit for today calls `POST /api/v1/routines/checkins` with `{ habit_id, date }`; UI updates optimistically
- Unchecking is allowed before midnight — calls `DELETE /api/v1/routines/checkins/{id}`; after midnight the day is locked
- Streak logic: consecutive calendar days where the habit was checked; missing one day resets to 0; today does not count until the habit is checked
- Maximum 10 habits in v1; the add button is disabled with a "Maximum habits reached" tooltip if at 10
- Habits can be deleted (swipe left on the habit row → Delete); deleting removes all history for that habit after confirmation
- Empty state: "No routines set up yet — add your first habit or tell Noxis what you're working on in chat"

## Prerequisites

- S-005-001 (dashboard grid — navigation entry point)
- S-005-007 (chat extraction pipeline — Noxis-suggested habits surfaced from chat commitments)
- S-003-001 (memory storage — routine consistency stored as memories)

## Acceptance Criteria

- [ ] Given the user has 3 defined habits and has checked 2 of them today, when the Routines screen renders, then the today section shows "2/3 complete" and 2 habits show checked state while 1 shows unchecked
- [ ] Given the user taps the checkmark on an unchecked habit, when the check is recorded, then the habit immediately transitions to checked state and the streak counter increments if this extends the streak
- [ ] Given the user has a habit with 8 consecutive days checked, when the module renders, then the streak badge shows "8-day streak"
- [ ] Given Noxis detects a commitment in chat ("I'll cold shower every morning"), when the chat extraction pipeline runs, then a suggestion card appears on the Routines screen (or via notification) with "Want to track this?" and pre-filled habit name
- [ ] Given the user already has 10 habits, when they tap the add button, then the button is disabled or the sheet opens with an inline "Maximum habits reached" message and no save option
- [ ] Given the user swipes left on a habit and confirms Delete, when the deletion completes, then the habit and all its checkin history are removed and the consistency view updates
- [ ] Given the weekly consistency view is rendered, when a day has all habits checked, then all cells in that day's column are filled in accent color

## Negative Scenarios

- `POST /api/v1/routines/checkins` fails → optimistic check is rolled back; habit reverts to unchecked state; inline error appears: "Couldn't save — tap to retry"
- User tries to uncheck a habit after midnight on the same calendar day → the UI shows the habit as locked (non-interactive checkmark) for past days; today's checkins remain editable until midnight

## Edge Cases

- User checks a habit at 23:59 and the day rolls over while the streak calculation is running → the streak is evaluated using the timestamp at the time of check; no edge case strip of earned streaks
- User has a habit with frequency "Weekdays" but checks it on Saturday → the Saturday check is stored and counts for the streak; frequency is currently advisory, not enforced in v1

## Dependencies

- S-005-001 (dashboard grid)
- S-005-007 (chat extraction pipeline)
- S-003-001 (memory storage)

## Technical Requirements

- **`RoutinesModuleView.swift`** — `ScrollView` with `TodayHabitStackView`, `WeeklyConsistencyGrid`; `HabitRow` components
- **`Habit` model:** `(id UUID, user_id, name: String, emoji: String?, frequency: HabitFrequency (.daily | .weekdays), created_at, source: HabitSource (.manual | .chat_suggestion))`
- **`HabitCheckin` model:** `(id UUID, habit_id, user_id, date: Date, created_at)`
- **Streak calculation:** `GET /api/v1/routines/summary` returns `{ habits: [{ id, name, emoji, streak, checked_today, weekly_checkins: [Date] }] }`; server computes streaks
- **Optimistic check:** `RoutinesViewModel.checkHabit(habitId)` appends a local `HabitCheckin` and updates `checkedToday = true` before API call resolves; rolls back on failure
- **Weekly consistency grid:** `WeeklyConsistencyGrid` — a `Grid` with 7 columns; each column is a `VStack` of small pill views per habit; color: accent if checked, muted if not checked, surface if habit didn't exist yet that day
- **Noxis suggestion flow:** `S-005-007` writes a `RoutineSuggestion` record; `RoutinesViewModel` includes a `pendingSuggestions: [RoutineSuggestion]` array; shown as a banner card above the habit stack on the Routines screen
- **10-habit limit:** enforced server-side in `POST /api/v1/routines/habits` with a 409 Conflict response; client enforces by hiding/disabling the add button when `habits.count >= 10`
- **Backend routes:** `GET /api/v1/routines/summary`, `POST /api/v1/routines/habits`, `DELETE /api/v1/routines/habits/{id}`, `POST /api/v1/routines/checkins`, `DELETE /api/v1/routines/checkins/{id}`

## Future Scope

- Habit scheduling: specify a target time for each habit (e.g., cold shower at 06:30); push reminder at that time
- Streak protection: one "forgive" per 30-day period that doesn't break a streak
- Noxis-written habit reviews: weekly summary of habit consistency in the daily brief
- Shared accountability stacks (opt-in, v3): two users can hold each other accountable on shared habits

## Do Not Do

- In this story, do not implement custom frequency (e.g., 3x per week) in v1 — daily and weekdays only
- Do not implement push reminders per habit in v1 — that is a future feature
- Do not show other users' habits or any social feature in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

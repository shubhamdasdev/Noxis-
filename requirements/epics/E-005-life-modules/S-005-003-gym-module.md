# Story: Gym Module — Workout Logging & Streak Tracking

---

**ID:** S-005-003
**Epic:** E-005
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a gym module that tracks my workout consistency with a streak counter and a simple log**, so that **I can see my training frequency at a glance and Noxis can give me fitness advice grounded in my actual history**.

## Goal

The Gym module detail screen shows the user's current workout streak, a weekly frequency bar chart, a chronological workout log, and a quick-log button for logging today's workout. Data from the module feeds the orchestration pipeline with real training context.

## Why / Rationale

Gym advice without training data is generic. When Noxis knows the user trained yesterday (Push), it doesn't need to be told — it can suggest the right follow-up. The streak counter is also a behavioral mechanism: seeing a 12-day streak is a concrete signal that compounds motivation. The module exists to make consistent training feel tangible.

## Functional Requirements

- The Gym module screen has three visual sections:
  1. **Streak header** — current consecutive training day streak displayed as a large Mono-font number + the word "day streak" beneath; a `StreakBadge` component shows this prominently
  2. **Weekly frequency chart** — a 7-day bar chart (Mon–Sun, current week); bars are filled for days with a logged workout, empty otherwise; shown in accent color for logged days, muted for empty
  3. **Workout log** — chronological list (most recent first) of all logged workouts; each row: date (relative), workout type, optional notes; no pagination in v1 (show last 30 entries)

- **Quick-log button** — a `GlassButton` at the bottom of the screen: "Log today's workout"; tapping opens a sheet with:
  - Workout type selector: Push / Pull / Legs / Upper / Lower / Cardio / Other (segmented picker or scrollable pill selector)
  - Optional notes field (max 200 chars)
  - "Save" button

- On save, the workout is written to `POST /api/v1/gym/workouts` and the streak and chart update immediately (optimistic update)
- Streak logic: consecutive calendar days with at least one logged workout; streak resets if a calendar day has no logged workout; today's log extends the streak; the streak includes today if a workout has already been logged today
- If a workout has already been logged for today, the quick-log button changes to "Add another workout" (same sheet, allows multiple per day)
- Workouts can also be auto-synced from chat via S-005-007 (e.g., "I just finished a leg session")
- Empty state (no workouts ever logged): "Start your streak — log your first workout" with the quick-log button prominent

## Prerequisites

- S-005-001 (dashboard grid — navigation entry point)
- S-005-007 (chat extraction pipeline — auto-sync workouts from chat)
- S-003-001 (memory storage — gym preferences and frequency stored as memories)

## Acceptance Criteria

- [ ] Given the user has logged workouts on 5 consecutive calendar days including today, when the Gym screen renders, then the streak header shows "5 day streak"
- [ ] Given the user taps "Log today's workout", when the sheet opens, then all 7 workout type options are selectable and the optional notes field is visible
- [ ] Given the user selects "Pull" and taps Save, when the save completes, then the workout appears in the log list as the first row and the streak counter increments if today was not already logged
- [ ] Given a workout is auto-synced from chat via S-005-007, when the Gym screen is next loaded, then the synced workout appears in the log list with `source: chat_sync` distinction (shown as a subtle tag or icon)
- [ ] Given the user has no logged workouts, when the screen renders, then the empty state is shown with the quick-log button as the primary CTA
- [ ] Given the weekly frequency chart is rendered, when a day has a logged workout, then that bar is filled in accent color; days without workouts show a muted empty bar
- [ ] Given the user logged a workout yesterday but not today, when the streak is calculated, then the streak count reflects only consecutive days up to and including yesterday (not broken yet until end of today)

## Negative Scenarios

- `POST /api/v1/gym/workouts` fails → the optimistic update is rolled back; the workout row is removed from the log and the streak reverts; an inline error appears: "Couldn't save — try again"
- User attempts to log a workout with an empty workout type (somehow bypasses the UI) → the backend rejects with a 400 error; client shows "Please select a workout type"

## Edge Cases

- User logs 3 workouts in one day → all three are stored; the streak counts the day once regardless of how many workouts are logged
- Day boundary edge case (user logs at 23:59 and then again at 00:01) → workout timestamps are stored in UTC but streak calculation uses the user's local timezone calendar day
- User starts using the app mid-week with a pre-existing gym habit → they manually log back-dated workouts (not supported in v1; only today's date is permitted in the quick-log flow)

## Dependencies

- S-005-001 (dashboard grid)
- S-005-007 (chat extraction pipeline)
- S-003-001 (memory storage)

## Technical Requirements

- **`GymModuleView.swift`** — `ScrollView` with three sections: `StreakHeaderView`, `WeeklyChartView`, `WorkoutLogView`; plus `GymQuickLogSheet`
- **`Workout` model:** `(id UUID, user_id, type: WorkoutType, notes: String?, logged_at: Date, source: LogSource (.manual | .chat_sync))`
- **`WorkoutType` enum:** `push | pull | legs | upper | lower | cardio | other`
- **Streak calculation:** backend computes — `GET /api/v1/gym/summary` returns `{ current_streak: Int, workouts_this_week: [WorkoutDay], recent_workouts: [Workout] }`; client does not compute streaks locally
- **`WeeklyChartView.swift`** — custom `Canvas`-based bar chart; 7 bars mapped to Mon–Sun; bar fill color: `DesignSystem.accent` for logged days, `DesignSystem.muted` for empty; no third-party charting library
- **Optimistic update:** `GymViewModel.logWorkout()` appends to `recentWorkouts` and increments `currentStreak` before the API call resolves; rolls back in `.catch`
- **Backend routes:** `GET /api/v1/gym/summary`, `POST /api/v1/gym/workouts`, `GET /api/v1/gym/workouts?limit=30`
- **Streak boundary:** server evaluates streaks using `users.timezone` for calendar day calculation

## Future Scope

- Workout split tracking: Noxis detects the user's split pattern from the log and surfaces it ("You're running Push/Pull/Legs/Push/Pull/Legs — you're on track for 6 days")
- Personal records: track max weight or volume per workout type
- Calendar view: monthly calendar heatmap showing training days
- Integration with Apple Health for automatic workout import

## Do Not Do

- In this story, do not track sets, reps, or weights — qualitative logging only in v1
- Do not integrate with Apple Health in v1
- Do not implement back-dated workout logging in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

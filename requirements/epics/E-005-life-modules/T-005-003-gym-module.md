# QA Test Plan — Gym Module — Workout Logging & Streak Tracking

---

**Plan ID:** T-005-003
**Story:** S-005-003
**Epic:** E-005
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the Gym module detail screen: streak header display with accurate consecutive-day calculation, weekly frequency bar chart rendering (filled vs. empty bars), chronological workout log (last 30 entries), quick-log sheet (all 7 workout types + notes), optimistic update on save and rollback on failure, the "Add another workout" button state when today is already logged, auto-synced workout entries from chat pipeline, and the empty state. Error scenarios cover API save failure and empty workout type validation.

## Out of Scope

- Sets, reps, weight tracking (not in v1)
- Apple Health integration (not in v1)
- Back-dated workout logging (not in v1)
- Workout split tracking, personal records, calendar heatmap (future scope)
- Dashboard grid navigation entry point (tested in T-005-001)

## Prerequisites

- S-005-001 (dashboard grid) is implemented and the Gym card navigates to this screen
- S-005-007 (chat extraction pipeline) is implemented; workout records with `source: chat_sync` can be pre-seeded
- S-003-001 (memory storage) is implemented
- Backend routes are deployed: `GET /api/v1/gym/summary`, `POST /api/v1/gym/workouts`, `GET /api/v1/gym/workouts?limit=30`
- Test user A: has workouts logged on 5 consecutive calendar days including today (streak = 5); has at least 1 chat_sync workout
- Test user B: has zero gym data
- Test user C: has a workout logged yesterday but not today
- Server timezone for test user A is set and `users.timezone` is configured

---

## Core Test Flow

### TC-005-003-001: Full happy path — streak renders, chart renders, log renders, quick-log creates a new entry, streak increments

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-005-001, S-005-007, S-003-001

**Preconditions:**
- Test user A has workouts on 5 consecutive days including today (today's workout is already logged so streak = 5)
- A second session is prepared where today's workout is NOT yet logged (streak = 4 before logging)
- Weekly chart for the current week has workouts on Mon, Tue, Wed, Thu (today = Thursday assumed)
- One chat_sync workout is pre-seeded with `source: chat_sync` for a prior day
- `GET /api/v1/gym/summary` returns `{ current_streak: 4, workouts_this_week: [...], recent_workouts: [...last 30...] }`
- Today has no workout logged yet for this precondition run

**Steps:**
1. Navigate to the Gym module from the Modules tab
2. Observe the streak header at the top of the screen
3. Observe the weekly frequency bar chart (Mon–Sun)
4. Scroll down to inspect the workout log
5. Locate the chat_sync workout entry in the log
6. Tap the "Log today's workout" button at the bottom of the screen
7. In the quick-log sheet, verify all 7 workout types are selectable
8. Select "Pull" as the workout type
9. Enter "Heavy rows and pull-ups" in the notes field
10. Tap "Save"
11. Observe the workout log
12. Observe the streak header after save
13. Observe the weekly bar chart after save

**Expected Result:**
- Step 2: streak header shows "4" in large Mono-font with "day streak" below it
- Step 3: bars for Mon, Tue, Wed, Thu are filled in accent color; Fri, Sat, Sun bars are muted/empty
- Step 4: workout log shows most recent entry first; entries include date (relative), workout type, and optional notes
- Step 5: the chat_sync workout row shows a subtle tag or icon distinguishing it as `source: chat_sync`
- Step 7: all 7 options are visible and selectable: Push, Pull, Legs, Upper, Lower, Cardio, Other
- Step 10: sheet dismisses immediately
- Step 11: the new "Pull" workout appears as the first row in the log with notes "Heavy rows and pull-ups" (optimistic update — visible before API confirmation)
- Step 12: streak header shows "5" (incremented by 1 via optimistic update)
- Step 13: today's bar (Thursday) is now filled in accent color

**Failure Indicators:**
- Streak header shows incorrect number or wrong typography
- Bar chart shows no bars or wrong filled/empty pattern
- Log is empty or unsorted
- Quick-log sheet is missing any of the 7 workout types
- New workout does not appear in the log after save
- Streak does not increment after first workout of the day

---

## Sub Flows

### TC-005-003-002: Today already logged — button label changes to "Add another workout"

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-3
**Dependencies:** S-005-001

**Preconditions:**
- Test user A has already logged a workout today (pre-seeded)
- `GET /api/v1/gym/summary` returns a streak that includes today

**Steps:**
1. Navigate to the Gym module
2. Inspect the label of the quick-log button at the bottom of the screen
3. Tap the button
4. Select "Cardio" and tap Save
5. Inspect the log

**Expected Result:**
- Step 2: button label reads "Add another workout" (not "Log today's workout")
- Step 3: the same log sheet opens
- Step 5: a second workout entry for today appears in the log (multiple workouts on the same day are allowed)

**Failure Indicators:**
- Button still says "Log today's workout" when a workout has already been logged today
- Second workout is rejected or not shown in the log

---

### TC-005-003-003: Streak counts only consecutive days — yesterday logged, today not logged yet

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-7
**Dependencies:** S-005-001

**Preconditions:**
- Test user C has a workout logged yesterday and no workout logged today
- `GET /api/v1/gym/summary` returns the streak computed for yesterday's date as the most recent logged day

**Steps:**
1. Navigate to the Gym module as test user C
2. Read the streak header

**Expected Result:**
- The streak header shows the count reflecting consecutive days up to and including yesterday
- Today's bar is empty/muted in the weekly chart
- Button label is "Log today's workout" (not "Add another workout")
- Streak is not yet broken (it is still within today's calendar day)

**Failure Indicators:**
- Streak shows 0 because today has not been logged
- Streak shows a number that includes today when no workout was logged today

---

### TC-005-003-004: Empty state — no workouts ever, quick-log button is primary CTA

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-5
**Dependencies:** S-005-001

**Preconditions:**
- Test user B has zero gym workouts

**Steps:**
1. Navigate to the Gym module as test user B
2. Read the screen content

**Expected Result:**
- Empty state message reads: "Start your streak — log your first workout"
- The "Log today's workout" quick-log button is prominently displayed as the primary CTA
- The streak header shows 0 or is not displayed
- The weekly chart shows all empty/muted bars
- The workout log section shows no entries (no crash, no placeholder rows)

**Failure Indicators:**
- Empty state message does not appear
- Quick-log button is hidden or deprioritized in the empty state

---

### TC-005-003-005: POST /api/v1/gym/workouts fails — optimistic update is rolled back

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-005-001

**Preconditions:**
- `POST /api/v1/gym/workouts` is configured to return a 500 error (network stub)
- Test user A has a streak of 3 and today has no workout yet

**Steps:**
1. Navigate to the Gym module
2. Note the streak count (3) and workout log state
3. Tap "Log today's workout"
4. Select "Legs" and tap Save
5. Observe the log immediately after the sheet dismisses (optimistic state)
6. Wait for the API error to be received (~1–3 seconds depending on timeout)
7. Observe the log and streak after rollback

**Expected Result:**
- Step 5: the "Legs" workout row briefly appears at the top of the log and streak shows 4 (optimistic update visible)
- Step 7: the "Legs" workout row is removed from the log; streak reverts to 3
- An inline error appears: "Couldn't save — try again"
- The quick-log button is available again and the user can retry

**Failure Indicators:**
- The workout row remains in the log despite the API failure
- The streak remains incremented after rollback
- No error message is shown to the user
- The app crashes on rollback

---

### TC-005-003-006: Empty workout type validation — backend rejects 400, client shows error message

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-005-001

**Preconditions:**
- A way exists to submit the quick-log sheet without a selected workout type (e.g., deep-link or accessibility bypass simulating a UI glitch)
- Backend is configured to return 400 for a workout POST with no type

**Steps:**
1. Attempt to submit the quick-log sheet with no workout type selected (via test bypass)
2. Observe the error response handling

**Expected Result:**
- The client shows the error message: "Please select a workout type"
- No workout is written to the log
- The sheet remains open for the user to correct the selection

**Failure Indicators:**
- A blank workout type entry is saved to the log
- No user-facing error is shown
- The app crashes on a 400 response

---

### TC-005-003-007: Multiple workouts in one day — streak counts the day once

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-005-001

**Preconditions:**
- Test user A has 2 workouts logged today (pre-seeded at different times)
- `GET /api/v1/gym/summary` is healthy

**Steps:**
1. Navigate to the Gym module
2. Read the streak value
3. Inspect today's bar in the weekly chart
4. Tap "Add another workout", select "Cardio", save (3rd workout of the day)
5. Re-read the streak value

**Expected Result:**
- Steps 2 and 5: the streak increments at most once for today (not by the number of workouts logged today)
- Step 3: today's bar in the weekly chart is filled — it does not appear "more full" for having multiple workouts
- The log shows all 3 workout entries for today

**Failure Indicators:**
- Streak increments once per workout rather than once per calendar day
- Today's bar is not filled after the first workout is logged

---

### TC-005-003-008: Chat-synced workout appears in log with source distinction

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-4
**Dependencies:** S-005-001, S-005-007

**Preconditions:**
- Test user A has a workout with `source: chat_sync` pre-seeded (type: "legs", notes: "heavy squats today")

**Steps:**
1. Navigate to the Gym module
2. Scroll through the workout log
3. Locate the chat_sync entry
4. Inspect its visual presentation

**Expected Result:**
- The entry is present in the log with workout type "Legs" and notes "heavy squats today"
- A subtle visual indicator (tag, icon, or label) distinguishes it as a chat-synced entry (e.g., a small chat bubble icon or "via chat" tag)
- The entry is sorted chronologically with other workouts

**Failure Indicators:**
- The chat_sync workout is absent from the log
- No visual distinction from manually logged workouts

---

## Automation Notes

- Selector strategy: accessibility identifiers on `StreakHeaderView` (accessibilityLabel = "streak-count"), `WeeklyChartView` bars (keyed by day index), `WorkoutLogView` rows (keyed by workout ID), and the quick-log button
- Optimistic update rollback test: requires URLProtocol stub with a configurable delay (e.g., 300ms) before returning the error, to allow assertion of the optimistic state before rollback
- Streak calculation is server-side — unit test the `StreakBadge` rendering separately (given a value, does it display correctly) rather than testing the calculation logic client-side
- Weekly chart: use snapshot testing (swift-snapshot-testing) for the custom Canvas-based chart to detect regressions in bar fill state
- Timezone edge case (23:59 / 00:01): integration test requires manipulating server-side `users.timezone` and injecting workouts with boundary timestamps; run as a backend-level integration test, not XCUITest
- Framework: XCTest (unit for ViewModel / GymViewModel optimistic logic) + XCUITest (E2E flows)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# QA Test Plan — Daily Brief Screen — Display & History

---

**Plan ID:** T-006-002
**Story:** S-006-002
**Epic:** E-006
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the Daily tab screen: rendering of today's full brief (insight, action items, curated content card) in the top glass card, `viewed_at` timestamp setting on first view, the 30-day brief history list below the today card with correct sorting and unread dot indicators, accordion expand/collapse on history rows, the "no brief today" state card, the "brief unavailable" state for failed generation, in-app URL opening via `SFSafariViewController` for curated content, and pull-to-refresh. Error handling covers API failure for brief fetch. Edge cases cover long-inactive users and very long insight text.

## Out of Scope

- Brief generation engine (S-006-001) — tested in T-006-001
- Push notification delivery (S-006-004) — tested in T-006-004
- Brief sharing, bookmarking, week-in-review card (future scope)
- Inline reply-to-brief (not in v1)
- History pagination beyond 30 days (not in v1)
- Sticky/pinned today card (explicitly not in scope per Do Not Do)

## Prerequisites

- S-006-001 (brief generation engine) is implemented and has produced briefs for test users
- S-007-001 (bottom tab bar) is implemented and the Daily tab is accessible
- S-007-003 (design system) is implemented; `GlassCard` and typography components exist
- Backend routes are deployed: `GET /api/v1/briefs/today`, `GET /api/v1/briefs/history?days=30`, `PATCH /api/v1/briefs/{id}/viewed`
- Test user A: has today's brief with `status: generated`, `viewed_at: null`, and 10 past briefs; at least 2 past briefs have `viewed_at: null` (unread)
- Test user B: has today's brief with curated content (non-null `curated_content` with a valid URL)
- Test user C: no brief for today, has 5 past briefs
- Test user D: today's brief has `status: failed`
- Test user E: no briefs at all (inactive for 10+ days)

---

## Core Test Flow

### TC-006-002-001: Full happy path — today's brief renders with all content, viewed_at is set, history renders with unread dots, accordion expand works, curated content opens in-app

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-006-001, S-007-001, S-007-003

**Preconditions:**
- Test user A: today's brief has `viewed_at: null`, insight text is 2 sentences, 3 action items, no curated content
- Test user B: today's brief has curated content with a valid HTTPS URL
- History for test user A: 10 past briefs; entries at index 0 and 2 (most recent) have `viewed_at: null`; all others have `viewed_at` set

**Steps (Test user A — core flow):**
1. Open the Daily tab for test user A
2. Confirm the screen title
3. Inspect the today card at the top of the screen
4. Read the date header on the today card
5. Read the insight text
6. Count and read the action items
7. Scroll below the today card to the history list
8. Count the history entries
9. Inspect entries at index 0 and 2 for the unread dot indicator
10. Inspect an entry with `viewed_at` set (no unread indicator expected)
11. Tap a past brief row at index 3 (has `viewed_at` set)
12. Inspect the expanded row content
13. Tap the same row again
14. Immediately after step 1, query `PATCH /api/v1/briefs/{id}/viewed` call logs

**Steps (Test user B — curated content):**
15. Switch to test user B and open the Daily tab
16. Inspect the today card for the curated content section
17. Tap the curated content card

**Expected Result:**
- Step 2: screen title "Daily" displayed in Display typography (large title style)
- Step 3: a prominent glass card is shown at the top; no shimmer
- Step 4: date header reads "Today — [Day, Month Date]" in Caption typography with muted color (e.g., "Today — Wednesday, March 18")
- Step 5: insight paragraph displayed in Body typography; full text visible, not truncated
- Step 6: exactly 3 action items in a bulleted list; each item has a subtle left-accent bar; text is specific and actionable
- Step 8: 10 past brief rows visible below the today card, sorted newest-first
- Step 9: rows at index 0 and 2 show an accent-color dot on the left edge of the row
- Step 10: other rows have no dot indicator
- Step 11: the row expands in-place (accordion animation) to show: full insight text, action items list, curated content if present
- Step 13: the row collapses back to its truncated state
- Step 14: `PATCH /api/v1/briefs/{id}/viewed` was called exactly once for today's brief (not called again on subsequent views)
- Step 16: a curated content card is visible at the bottom of the today card; it shows a title, type badge, and description
- Step 17: `SFSafariViewController` opens the linked URL within the app; the app is not exited to Safari

**Failure Indicators:**
- Today card is missing insight text or action items
- Date header format is wrong or shows a placeholder
- Less than or more than 3 action items rendered
- History list is not sorted newest-first
- Unread dot missing on rows with `viewed_at: null`
- Accordion expand navigates to a new screen instead of expanding in-place
- `viewed_at` is set on every open rather than only the first time
- Curated content link opens the system Safari browser instead of in-app

---

## Sub Flows

### TC-006-002-002: No brief for today — state card with correct copy is shown

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-6
**Dependencies:** S-006-001, S-007-001

**Preconditions:**
- Test user C: no brief for today; has 5 past briefs

**Steps:**
1. Open the Daily tab as test user C
2. Read the state card content in place of the today card

**Expected Result:**
- The state card reads: "No brief today — keep using Noxis and tomorrow's brief will reflect your activity"
- The history list still renders below with the 5 past briefs
- No error UI, no broken layout

**Failure Indicators:**
- State card text differs from the specified copy
- The history list is also hidden when there is no today brief
- A crash or blank screen is shown

---

### TC-006-002-003: Brief has status: failed — "unavailable" message shown in today card

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-006-001

**Preconditions:**
- Test user D: today's brief has `status: failed`

**Steps:**
1. Open the Daily tab as test user D
2. Read the today card content

**Expected Result:**
- The today card shows: "Brief unavailable today — Noxis will try again tomorrow"
- No error codes, no stack trace, no empty card
- The history list renders normally below

**Failure Indicators:**
- The card is blank or shows a spinner indefinitely
- An error code or technical message is displayed
- The card shows partial brief content from the failed generation

---

### TC-006-002-004: GET /api/v1/briefs fails — inline error with pull-to-refresh instruction

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-007-001

**Preconditions:**
- Both `GET /api/v1/briefs/today` and `GET /api/v1/briefs/history?days=30` return 500 errors (network stub)

**Steps:**
1. Open the Daily tab
2. Wait for the fetch to fail
3. Inspect both the today card area and the history list area
4. Pull down to trigger pull-to-refresh

**Expected Result:**
- Step 3: an inline error state appears in place of both the today card and history list; message reads: "Couldn't load your brief — pull to refresh"
- No blocking modal or full-screen error overlay
- Step 4: pull-to-refresh triggers a new fetch attempt; if the stub is removed, content loads successfully

**Failure Indicators:**
- Today card and history list show blank/empty states without an error message
- Pull-to-refresh is unavailable in the error state
- A modal error dialog blocks interaction

---

### TC-006-002-005: Inactive user — no history, no today brief, correct states shown

**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-007-001

**Preconditions:**
- Test user E: no briefs at all in `daily_briefs` table

**Steps:**
1. Open the Daily tab as test user E
2. Read the today card area
3. Observe the area below the today card

**Expected Result:**
- Step 2: "No brief today — keep using Noxis and tomorrow's brief will reflect your activity" state card
- Step 3: no history list rows; no empty list error state shown; the area below the today card is simply empty (or has a graceful empty state)
- No crash, no "null" text, no broken layout

**Failure Indicators:**
- App crashes when both today brief and history are empty
- A "no items" error state appears with alarming visual treatment

---

### TC-006-002-006: Long insight text — full text displayed, card height expands naturally

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-006-001, S-007-001

**Preconditions:**
- Today's brief for the test user has an `insight` field with 600+ characters (seeded in the test database)

**Steps:**
1. Open the Daily tab
2. Read the today card
3. Scroll through the card

**Expected Result:**
- The full insight text is displayed without truncation
- The today card height expands to fit the content naturally
- No text clips at the card boundary
- Character count does not affect layout of the action items below the insight

**Failure Indicators:**
- Insight text is truncated with an ellipsis
- Card height is fixed and text overflows outside the card boundary

---

### TC-006-002-007: Accordion expand — only one history row expanded at a time

**Type:** Happy Path
**Priority:** P2
**AC Covered:** AC-5
**Dependencies:** S-006-001

**Preconditions:**
- Test user A has at least 5 past briefs in the history list

**Steps:**
1. Open the Daily tab
2. Tap past brief row at index 0 — it expands
3. Tap past brief row at index 2 — observe both rows

**Expected Result:**
- After step 2: row 0 is expanded showing full content
- After step 3: row 2 expands; row 0 collapses (only one row expanded at a time — `expandedId` is singular)

**Failure Indicators:**
- Both rows are expanded simultaneously
- Tapping a second row does not collapse the first

---

## Automation Notes

- Selector strategy: accessibility identifiers on `TodayBriefCard` (id = "today-brief-card"), `BriefHistoryRow` keyed by brief ID, unread dot indicator (accessibilityIdentifier = "unread-indicator-{briefId}"), curated content card (accessibilityIdentifier = "curated-content-card")
- `viewed_at` assertion: inspect network call logs or API call count via XCUITest + URLProtocol observer; assert `PATCH /viewed` called exactly once per session
- Accordion animation: assert `isExpanded` state via accessibility traits; visual snapshot testing for the expanded/collapsed state
- `SFSafariViewController` detection: assert that the presented view controller is `SFSafariViewController` class rather than checking browser navigation
- Pull-to-refresh: XCUITest `.swipeDown()` on the scroll view; assert network request is re-issued
- Flakiness risk: `viewed_at` timing assertion — add a short explicit wait (XCTNSPredicateExpectation) to allow the PATCH call to complete before asserting
- Framework: XCTest (unit for DailyBriefViewModel state logic) + XCUITest (E2E flows)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

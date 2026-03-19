# QA Test Plan — Dashboard Grid — Module Cards Overview

---

**Plan ID:** T-005-001
**Story:** S-005-001
**Epic:** E-005
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the end-to-end behavior of the Modules tab dashboard grid: the rendering of all five module cards in the correct 2-column order, accurate live status summaries fetched from `GET /api/v1/modules/summary`, shimmer loading states, navigation to module detail screens via card tap, pull-to-refresh, fallback copy for empty modules, and the screen title. Tests validate both the happy path and degraded states including API failure and small device rendering.

## Out of Scope

- Module detail screens (S-005-002 through S-005-006) — navigation destination content is not tested here
- `ModuleCard` component internals — defined in S-007-003
- Bottom tab bar implementation (S-007-001)
- Module reordering (future scope)
- Module-level notification dots (future scope)
- A sixth custom module card (future scope)
- Charts, graphs, or trend indicators on cards
- Full module data inline on cards

## Prerequisites

- S-007-001 (bottom tab bar) is implemented and the Modules tab is accessible
- S-007-003 (design system) is implemented and `ModuleCard` component exists
- `GET /api/v1/modules/summary` endpoint is deployed and returns the documented JSON shape
- A test user account exists with at least one module containing data (e.g., 3 wardrobe items, 4-day gym streak)
- A second test user account with zero data across all modules

---

## Core Test Flow

### TC-005-001-001: Full happy path — all five cards render with live data, navigation works, pull-to-refresh updates data

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- Test user has: 7 wardrobe items, 4-day gym streak, 3 meals logged this week, 2 purchases this week, 5 active habits
- `GET /api/v1/modules/summary` is healthy and returns accurate counts
- User is on the app home screen (not currently on Modules tab)

**Steps:**
1. Tap the Modules tab in the bottom tab bar
2. Observe the screen during initial load before data arrives
3. Wait for data fetch to complete (up to 5 seconds)
4. Inspect the screen title above the grid
5. Count and verify the order of all five module cards
6. Read the status summary on each card
7. Verify the Wardrobe card status line
8. Verify the Gym card status line
9. Verify the Food card status line
10. Verify the Spending card status line
11. Verify the Routines card status line
12. Tap the Gym card
13. Confirm the Gym module detail screen is pushed onto the navigation stack
14. Navigate back to the Modules tab
15. Pull down on the grid to trigger pull-to-refresh
16. Observe the refresh and confirm status lines update

**Expected Result:**
- During step 2: each card displays a shimmer animation in place of the status summary line; no static text is visible in the status area
- Step 4: "Your Life" is displayed in Display typography, left-aligned, above the grid
- Step 5: exactly 5 cards rendered in a 2-column grid, in order top-left to bottom-right: Wardrobe (tshirt icon), Gym (figure.strengthtraining.traditional icon), Food (fork.knife icon), Spending (creditcard icon), Routines (checkmark.seal icon)
- Step 7: Wardrobe card shows "7 items in your wardrobe"
- Step 8: Gym card shows "4-day streak"
- Step 9: Food card shows "3 meals logged this week"
- Step 10: Spending card shows "2 purchases this week"
- Step 11: Routines card shows "5 habits active"
- Step 13: the Gym module detail screen is pushed (back button visible in navigation bar); Modules grid is not visible
- Step 16: summary data is re-fetched; status lines reflect the latest values from the API

**Failure Indicators:**
- Fewer or more than 5 cards rendered
- Cards appear in a different order than Wardrobe → Gym → Food → Spending → Routines
- Any status line shows placeholder, blank, or lorem ipsum text after data load
- No shimmer animation during loading
- Tapping Gym card does not navigate
- Pull-to-refresh does not trigger a new API call

---

## Sub Flows

### TC-005-001-002: Empty module — zero data fallback copy is shown, never a blank line

**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-6
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- Test user has zero wardrobe items, zero gym workouts, zero food logs, zero spending entries, zero active habits
- `GET /api/v1/modules/summary` returns zero counts for all modules

**Steps:**
1. Open the Modules tab as this zero-data test user
2. Wait for data load to complete
3. Read the status line on each card

**Expected Result:**
- Wardrobe: "0 items in your wardrobe" (or the defined fallback if count 0 has a specific phrase — if the copy spec treats 0 the same as "N items", "0 items in your wardrobe" is acceptable)
- Gym: "No workouts logged yet"
- Food: "Nothing logged yet"
- Spending: "Nothing tracked yet"
- Routines: "No routines set up yet"
- No card shows a blank or empty status line

**Failure Indicators:**
- Any card shows an empty string or whitespace-only status line
- Any card shows the shimmer animation after data has finished loading

---

### TC-005-001-003: API failure — status lines show dash placeholder, cards remain interactive

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- `GET /api/v1/modules/summary` is configured to return a 500 error (network intercept or test environment flag)
- Test user is authenticated

**Steps:**
1. Open the Modules tab
2. Wait for the fetch attempt to complete and fail
3. Inspect all five card status lines
4. Tap the Wardrobe card
5. Tap the back button and tap the Food card

**Expected Result:**
- All five card status lines display a muted "—" dash placeholder
- No blocking error overlay or modal covers the grid
- Tapping Wardrobe navigates to the Wardrobe detail screen
- Tapping Food navigates to the Food detail screen
- Navigation functions normally despite the API failure

**Failure Indicators:**
- A blocking error state prevents card interaction
- Any card shows an empty string instead of "—"
- Navigation is disabled while in the error state

---

### TC-005-001-004: Card tap while summary is still loading — navigation proceeds immediately

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- Network latency is artificially increased (e.g., 3-second delay on `GET /api/v1/modules/summary`) so shimmer is visible for an extended time
- Test user is authenticated

**Steps:**
1. Open the Modules tab
2. While the shimmer animation is still showing (before data loads), tap the Spending card

**Expected Result:**
- The app navigates immediately to the Spending module detail screen
- The shimmer loading state is dismissed
- No crash or hang occurs

**Failure Indicators:**
- Tap on card while loading is unresponsive
- Navigation is blocked until data fetch completes

---

### TC-005-001-005: Small screen rendering — iPhone SE, 2-column grid compresses gracefully

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- Simulator or device set to iPhone SE (375pt wide, or the smallest supported width)
- Test user has standard module data

**Steps:**
1. Open the Modules tab on an iPhone SE-sized device/simulator
2. Visually inspect all five cards
3. Scroll to ensure the fifth card (Routines) is reachable

**Expected Result:**
- All five cards are visible in a 2-column layout
- No card content (icon, name, status text) is clipped or overflows its card boundary
- Cards maintain their aspect ratio with reduced horizontal padding compared to larger devices
- The "Your Life" title and grid are fully legible

**Failure Indicators:**
- Any card text clips or is truncated with no graceful handling
- Cards overlap each other
- The grid layout breaks to 1 column unintentionally

---

### TC-005-001-006: Large data count — 999+ wardrobe items renders overflow copy

**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-2
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- Test user's `wardrobe.item_count` is set to 1000 in the test database
- `GET /api/v1/modules/summary` returns `{ wardrobe: { item_count: 1000 } }`

**Steps:**
1. Open the Modules tab
2. Wait for data load
3. Read the Wardrobe card status line

**Expected Result:**
- Wardrobe card status line shows "999+ items in your wardrobe"
- No number overflow, no clipping, no raw "1000" or higher integer displayed

**Failure Indicators:**
- Status line shows the raw count "1000 items in your wardrobe"
- Status text overflows its container or clips

---

## Automation Notes

- Selector strategy: use accessibility identifiers on `ModuleCard` views keyed by `LifeModule` enum value (e.g., `accessibilityIdentifier = "module-card-gym"`) for reliable XCUITest targeting
- Status summary text: use `ModuleStatusFormatter.format()` unit tests to cover string assembly logic separately from UI tests — pure function is directly unit testable
- Shimmer state: use network intercept (URLProtocol stub) to delay the summary response and assert shimmer is visible before the response arrives
- API failure simulation: use a URLProtocol-based stub that returns 500 for `/api/v1/modules/summary`
- Flakiness risk: timing-based shimmer tests — add explicit wait assertions (XCTNSPredicateExpectation) rather than fixed sleeps
- Framework: XCTest (unit) + XCUITest (E2E); Swift

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

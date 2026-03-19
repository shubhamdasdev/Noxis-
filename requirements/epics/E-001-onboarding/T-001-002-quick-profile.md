# QA Test Plan — Quick Profile Screen

---

**Plan ID:** T-001-002
**Story:** S-001-002
**Epic:** E-001
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the Quick Profile screen — a single glass screen with three multi-choice pill-selector questions (primary focus, fitness frequency, cooking habit) that seeds the onboarding state before conversational onboarding. Testing covers correct screen layout and question rendering, pill tap and deselect behavior, single-selection-per-question enforcement, Continue with selections, Skip behavior, selection persistence on back navigation, and the negative scenario of proceeding with no answers selected.

## Out of Scope

- Text input fields — this screen uses visual pill selectors only (Do Not Do)
- Making any question required — all are optional (Do Not Do)
- Extended onboarding questions (style aesthetic, relationship status, city) — Future Scope
- API calls at this step — data is held in memory only (no network interaction to test here)

## Prerequisites

- S-001-001 (Tone Selection) is completed and the selected tone is stored in `OnboardingState`
- S-007-003 (Design system) is implemented — pill selector component is available
- App is in a post-tone-selection navigation state

---

## Core Test Flow

### TC-001-002-001: Full quick profile flow — all questions answered, Continue proceeds with data stored
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-001-001, S-007-003

**Preconditions:**
- User has completed Tone Selection (any tone) and has been navigated to the Quick Profile screen
- `OnboardingState` is initialized with the selected tone

**Steps:**
1. Observe the Quick Profile screen — confirm header "Quick setup" in Headline style and subtext "Takes 20 seconds. Skip anytime." in Caption muted style are visible
2. Confirm three question blocks are visible, each with pill options in their initial unselected state (no accent borders):
   - Q1: "What do you most want to level up?" with pills: Style | Fitness | Daily Routine | All of the above
   - Q2: "Do you train?" with pills: Don't train | 1-2x/week | 3-4x/week | 5+ days/week
   - Q3: "Do you cook?" with pills: Never — I order everything | Sometimes | Regularly
3. Confirm "Continue" button is visible and active (it is always active per FR)
4. Confirm "Skip" is visible as a muted text link
5. Tap **Style** in Q1 — confirm it highlights with an accent border; all other Q1 pills remain unselected
6. Tap **Style** again — confirm it deselects (accent border removed)
7. Tap **Fitness** in Q1 — confirm it selects; **Style** remains unselected
8. Tap **3-4x/week** in Q2 — confirm it selects with accent border; other Q2 pills remain unselected
9. Tap **Sometimes** in Q3 — confirm it selects
10. Tap **Continue**
11. Confirm the conversational onboarding screen opens
12. Confirm that `OnboardingState` contains `{ primaryFocus: Fitness, fitnessLevel: 3to4PerWeek, cookingLevel: Sometimes }` (verify via debug output or test spy)

**Expected Result:**
- Screen renders with correct header, subtext, three question blocks, all pill options unselected on arrival
- Tapping a pill highlights it with accent border; tapping the same pill deselects it
- Only one pill per question can be selected at a time
- Continue is active regardless of selection state
- After tapping Continue, conversational onboarding opens with the three selections stored in `OnboardingState`

**Failure Indicators:**
- Any question is missing, has wrong options, or has wrong label text
- A pill fails to highlight on tap or fails to deselect on second tap
- Two pills in the same question are selected simultaneously
- Continue button is disabled (it must always be active)
- Navigating forward does not preserve the selections in `OnboardingState`
- A text input field appears anywhere on this screen

### TC-001-002-002: Skip tapped — proceeds with no selections stored
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-001-001, S-007-003

**Preconditions:**
- User is on the Quick Profile screen with no pills selected

**Steps:**
1. Without selecting any pills, tap the "Skip" muted text link
2. Confirm navigation proceeds to conversational onboarding
3. Confirm `OnboardingState` contains no quick profile selections (all three fields are nil/unset)

**Expected Result:**
- Tapping Skip navigates to conversational onboarding immediately
- `OnboardingState.primaryFocus`, `fitnessLevel`, and `cookingLevel` are all nil/unset

**Failure Indicators:**
- Skip does not navigate forward
- Skip stores default or empty values instead of nil/unset
- Skip navigates to a different screen than Continue would

### TC-001-002-003: No questions answered, Continue tapped — proceeds without error
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-003
**Dependencies:** S-001-001, S-007-003

**Preconditions:**
- User is on the Quick Profile screen; no pills have been tapped

**Steps:**
1. Confirm all pills are unselected
2. Tap "Continue" without selecting any answers
3. Observe navigation and state

**Expected Result:**
- The app navigates to conversational onboarding
- No validation error, alert, or toast appears
- `OnboardingState` reflects no selections (nil for all three fields)

**Failure Indicators:**
- An error or validation message fires preventing navigation
- Continue is disabled when no answers are selected (it must always be active)
- A default selection is silently applied to any question

### TC-001-002-004: Back navigation — prior selections remain highlighted
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-001-001, S-007-003

**Preconditions:**
- User has selected Fitness (Q1), 5+ days/week (Q2), and Regularly (Q3) on the Quick Profile screen, then navigated forward to conversational onboarding

**Steps:**
1. Tap the back button from the conversational onboarding screen to return to Quick Profile
2. Observe the pill states

**Expected Result:**
- Fitness pill in Q1 shows accent border (still selected)
- 5+ days/week pill in Q2 shows accent border (still selected)
- Regularly pill in Q3 shows accent border (still selected)
- No pill state has been reset by the back navigation

**Failure Indicators:**
- Any previously selected pill appears unselected after back navigation
- A different pill appears selected than what the user originally chose

---

## Automation Notes

- **Accessibility identifiers:** Assign `pill-q1-style`, `pill-q1-fitness`, `pill-q1-daily`, `pill-q1-all`, `pill-q2-none`, `pill-q2-1to2`, `pill-q2-3to4`, `pill-q2-5plus`, `pill-q3-never`, `pill-q3-sometimes`, `pill-q3-regularly`, `btn-continue`, `btn-skip`
- **Selection state:** Assert accessibility `isSelected` trait or a custom selected/unselected accessibility value on each pill element; XCUITest can check `element.isSelected`
- **OnboardingState inspection:** Use a test observer or protocol mock to capture the state written to `OnboardingState` after Continue/Skip — do not rely on UI-only assertions for data correctness
- **Flakiness risk:** Pill highlight (accent border) may be visually subtle — supplement XCUITest `isSelected` assertions with a snapshot test for selected vs unselected pill states to catch styling regressions

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# QA Test Plan — Welcome & Tone Selection Screens

---

**Plan ID:** T-001-001
**Story:** S-001-001
**Epic:** E-001
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the two-screen Welcome and Tone Selection sequence for unauthenticated Noxis users. Testing covers the Welcome screen visual layout (wordmark, tagline, CTA on pure black), navigation to Tone Selection, rendering of all three tone cards with correct content, tap-to-select behavior (accent glow border), Continue button enabled/disabled state, tone selection persistence on back navigation, and the negative scenario of tapping Continue without a selection. The edge case of a small screen (iPhone SE) with scrollable card layout is also covered.

## Out of Scope

- Authentication implementation — auth is S-001-005 (Do Not Do for this story)
- Feature preview or app tour (Do Not Do)
- Animated Noxis logo or particle effect on Welcome screen (Future Scope)
- Any screen after Quick Profile — this plan covers only Welcome and Tone Selection

## Prerequisites

- S-007-003 (Design system) is implemented
- S-007-004 (App launch routing) is implemented and routes unauthenticated users to the Welcome screen
- App is in a clean unauthenticated state (no JWT in Keychain)

---

## Core Test Flow

### TC-001-001-001: Welcome screen render → tone selection → card select → proceed to Quick Profile
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-007-003, S-007-004

**Preconditions:**
- Device is in unauthenticated state (no Keychain JWT)
- App is launched fresh so routing lands on the Welcome screen

**Steps:**
1. Observe the Welcome screen — confirm: pure black background (#000000), Noxis wordmark centered in SF Pro Display Bold 28pt white, tagline "Your standards, elevated." in SF Pro Text Regular 16pt secondary white, and a single "Get Started" glass button with accent border
2. Confirm no sign-in option is visible on this screen
3. Tap "Get Started"
4. Confirm the Tone Selection screen appears with the header "How should I talk to you?" in Headline style
5. Confirm three glass cards are visible, stacked vertically, with 16pt horizontal padding, each showing: mode name (Headline, white), one-line description (Body, secondary), and a sample exchange with a muted "User:" prompt and a "Noxis:" response in Caption style
6. Confirm the three cards are: **Brother** (sample: "That hoodie is doing nothing for you. Swap it for the bomber — you have 3 minutes."), **Consultant** ("The hoodie adds bulk without structure. Your bomber would shift this to casual-sharp."), **Peer** ("Honest take — the bomber you wore last week hit way harder. Switch it in.")
7. Confirm the "Continue" button is visible but disabled (not tappable or visually inactive)
8. Tap the **Brother** card — confirm it shows an accent glow border; confirm "Continue" becomes active
9. Tap the **Consultant** card — confirm the accent glow border moves to Consultant and is removed from Brother
10. Tap "Continue" — confirm navigation proceeds to the Quick Profile screen (S-001-002) and the selected tone (Consultant) is passed forward
11. Navigate back to the Tone Selection screen — confirm the Consultant card still shows the accent glow border (selection persisted)

**Expected Result:**
- Welcome screen: black background, wordmark, tagline, and "Get Started" CTA exactly as specified; no sign-in UI
- Tapping "Get Started" navigates to Tone Selection with no loading state
- All three tone cards render with correct mode names, descriptions, and sample exchanges in the correct type styles
- Continue button is disabled until exactly one card is selected
- Selecting a card applies an accent glow border and enables Continue; selecting another card moves the glow
- Tapping Continue navigates to Quick Profile with the selected tone stored
- Returning via back navigation shows the previously selected tone card still highlighted

**Failure Indicators:**
- Welcome screen has any non-black background, missing wordmark, or a sign-in option
- Tone Selection shows fewer than three cards, incorrect card text, or missing sample exchanges
- Continue button is active before any card is selected
- Accent glow border does not appear on tap, or remains on the deselected card
- Navigating back to Tone Selection clears the selection

### TC-001-001-002: Continue tapped with no tone selected — button stays disabled
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-003
**Dependencies:** S-007-003

**Preconditions:**
- User is on the Tone Selection screen with no card selected

**Steps:**
1. Observe the Continue button — confirm it appears visually inactive (disabled state)
2. Attempt to tap the Continue button directly
3. Confirm no navigation occurs and no error is shown

**Expected Result:**
- The Continue button does not respond to taps while no tone is selected
- The user remains on the Tone Selection screen
- No error message, alert, or toast appears — the button simply does not activate

**Failure Indicators:**
- Continue button navigates forward despite no selection
- An error toast or alert fires when tapping the disabled button
- Button appears active (no visual disabled state) when no card is selected

### TC-001-001-003: Tapping a selected card again — deselects or switches to another card
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-003
**Dependencies:** S-007-003

**Preconditions:**
- User is on the Tone Selection screen with Brother card selected (accent glow visible, Continue active)

**Steps:**
1. Tap the **Brother** card again (already selected)
2. Observe whether the selection is deselected (no glow, Continue disabled) or remains selected
3. Tap the **Peer** card
4. Confirm accent glow moves to Peer and Brother no longer has a glow

**Expected Result:**
- Tapping the already-selected card either deselects it (glow removed, Continue disabled) or remains selected — behavior is consistent per the FR ("tapping again or tapping another card switches selection")
- Tapping a different card always moves the glow to the new card

**Failure Indicators:**
- Two cards show the accent glow simultaneously
- Tapping a new card does not remove the glow from the previously selected card

### TC-001-001-004: Small screen (iPhone SE) — cards scroll without hidden content
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-001, AC-002, AC-003
**Dependencies:** S-007-003

**Preconditions:**
- App is running on an iPhone SE (3rd gen) simulator or device (375×667 logical points)

**Steps:**
1. Navigate to the Tone Selection screen
2. Observe whether all three cards are fully visible without scrolling
3. If cards extend beyond the viewport, scroll down to confirm all cards and the Continue button are reachable
4. Confirm no card content (mode name, description, sample exchange) is clipped or cut off

**Expected Result:**
- All three cards and the Continue button are accessible — either all visible at once or via scrolling
- No card content is horizontally clipped or obscured
- Footnote "You can change this anytime in settings." is reachable by scrolling if necessary

**Failure Indicators:**
- The third card or Continue button is not reachable (no scroll available and content is below the fold)
- Card text is horizontally clipped on SE screen width

---

## Automation Notes

- **Welcome screen:** XCUITest can assert `staticText` elements for "Your standards, elevated." and "Get Started"; use accessibility identifiers `welcome-wordmark`, `welcome-tagline`, `welcome-cta`
- **Card selection:** Assign accessibility identifiers `tone-card-brother`, `tone-card-consultant`, `tone-card-peer` and `continue-button`; assert `isEnabled` state on `continue-button` before and after card tap
- **Accent glow border:** Not directly assertable via XCUITest — verify via snapshot test (`swift-snapshot-testing`) comparing selected vs unselected card states
- **Tone persistence on back nav:** Automate by navigating forward and back programmatically and asserting the selected card's accessibility state
- **Flakiness risk:** Glass blur rendering and glow border may vary slightly between simulator renders — run snapshot tests on a consistent simulator image version and do not update baselines on CI without human review

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

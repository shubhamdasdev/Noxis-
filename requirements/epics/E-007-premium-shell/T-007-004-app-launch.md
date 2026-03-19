# QA Test Plan — App Launch & Splash Screen

---

**Plan ID:** T-007-004
**Story:** S-007-004
**Epic:** E-007
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the app launch sequence including the pure-black splash screen (no white flash), auth state routing logic (authenticated user before 10am → Daily tab; authenticated user after 10am → Chat tab; unauthenticated user → Welcome onboarding), the 2-second auth timeout fallback to onboarding, and the background/resume behavior that skips the splash. Both negative scenarios (auth service unavailable, corrupted token) and both edge cases (post-sign-out state, app resume) receive dedicated test cases.

## Out of Scope

- Animated logo on launch (Future Scope — v2 polish pass)
- Building the onboarding flow itself — this story only routes to it (Do Not Do)
- Showing a loading spinner during auth check (Do Not Do)
- Content of the Chat tab, Daily tab, or Welcome screen — only the routing is validated here

## Prerequisites

- S-007-003 (Design system) is implemented — `LaunchScreen.storyboard` background set to #000000
- S-001-005 (Authentication) is implemented — `AuthManager.shared.isAuthenticated` returns valid state
- Three test accounts are available: one authenticated (valid JWT in Keychain), one with a corrupted/expired JWT, and one unauthenticated (no Keychain entry)
- Device clock can be manipulated (via simulator time override or device Settings) to test before/after 10am routing

---

## Core Test Flow

### TC-007-004-001: Full launch sequence — no white flash, all routing branches verified
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-007-003, S-001-005

**Preconditions:**
- Test device is a physical iPhone (white-flash testing requires hardware; simulator compositing may differ)
- Three Keychain states are prepared: valid JWT, no JWT (new user), and a state where auth service response can be delayed > 2 seconds (proxy or mock)
- Device time is configurable

**Steps:**
1. **Unauthenticated cold launch:**
   - Clear Keychain (no JWT stored)
   - Force-quit and relaunch the app
   - Observe the launch screen: confirm pure black background with Noxis wordmark centered, white text, no white flash at any point during the transition
   - Confirm the app routes to the Welcome onboarding screen after the auth check

2. **Authenticated cold launch — after 10am:**
   - Set device time to 11:00am (user's local timezone)
   - Install a valid JWT in Keychain
   - Force-quit and relaunch the app
   - Observe the splash and transition
   - Confirm the app opens to the Chat tab (not Daily, not onboarding)

3. **Authenticated cold launch — before 10am:**
   - Set device time to 07:30am (user's local timezone)
   - Valid JWT remains in Keychain
   - Force-quit and relaunch
   - Confirm the app opens to the Daily tab

4. **Auth timeout — delayed response > 2 seconds:**
   - Block or delay the auth check to exceed 2 seconds (use a mock that introduces a 2.5s delay)
   - Launch the app
   - Confirm the app holds on the splash screen (no spinner appears) until the 2-second threshold, then routes to the Welcome onboarding screen
   - Confirm no spinner or `ProgressView` appears at any point

**Expected Result:**
- Launch screen background is #000000 with no white frame visible during launch or the fade transition (300ms ease-in-out) to the first screen
- Unauthenticated user lands on Welcome onboarding
- Authenticated user after 10am lands on Chat tab
- Authenticated user before 10am lands on Daily tab
- Auth check timing out at 2 seconds routes to onboarding; splash holds steady without a spinner until then

**Failure Indicators:**
- Any white, grey, or non-black frame visible at any point during launch or transition
- Authenticated user routes to onboarding instead of Chat or Daily
- Before/after 10am routing is inverted (Daily after 10am, Chat before 10am)
- A spinner or activity indicator appears during the auth wait
- Timeout routes the user somewhere other than onboarding, or does not trigger at all (app hangs indefinitely)

### TC-007-004-002: Auth service unavailable at launch — routes to onboarding
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-007-003, S-001-005

**Preconditions:**
- Device has no network connectivity (Airplane Mode enabled) or auth service is mocked to return an error
- Keychain contains a JWT (authenticated state pre-existing) to confirm the failure path overrides it

**Steps:**
1. Enable Airplane Mode or block the auth endpoint
2. Force-quit and cold-launch the app
3. Observe routing

**Expected Result:**
- App routes to the Welcome onboarding screen
- No crash occurs
- Splash screen displays correctly during the wait period before timeout/failure

**Failure Indicators:**
- App crashes on auth service failure
- App hangs indefinitely without routing anywhere
- App routes to Chat or Daily despite auth service being unavailable

### TC-007-004-003: Corrupted local auth token — cleared and routes to onboarding
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-007-003, S-001-005

**Preconditions:**
- Keychain contains a deliberately corrupted or expired JWT string (e.g. a malformed token or one past its expiry timestamp)

**Steps:**
1. Ensure the corrupted JWT is in Keychain
2. Force-quit and cold-launch the app
3. Observe routing and Keychain state post-launch

**Expected Result:**
- App detects the token is invalid or corrupted
- Keychain entry is cleared
- App routes to the Welcome onboarding screen
- No crash or unhandled error occurs

**Failure Indicators:**
- App routes to Chat or Daily using a corrupted token
- Keychain entry is not cleared (corrupted token persists for next launch)
- App crashes or shows an unhandled exception during token validation

### TC-007-004-004: User has just signed out — next cold launch routes to onboarding
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-007-003, S-001-005

**Preconditions:**
- User was previously authenticated and has just completed sign-out within the app session

**Steps:**
1. Sign out from within the app (via Profile or Settings)
2. Force-quit the app
3. Cold-launch the app

**Expected Result:**
- Auth state is immediately unauthenticated after sign-out (Keychain cleared)
- On next cold launch, the app routes to the Welcome onboarding screen — not to Chat or Daily

**Failure Indicators:**
- App routes to Chat or Daily after sign-out and re-launch
- Sign-out does not clear the Keychain, leaving a stale token

### TC-007-004-005: App backgrounded and resumed — skips splash, resumes last active screen
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-001
**Dependencies:** S-007-003

**Preconditions:**
- App is running (authenticated, on the Chat tab)
- App is sent to background by pressing the home button or swiping up

**Steps:**
1. Open the app to the Chat tab
2. Background the app
3. Wait 5 seconds
4. Foreground the app by tapping its icon

**Expected Result:**
- The splash screen does not appear on resume
- The app returns directly to the Chat tab in the state it was in when backgrounded
- No auth check delay or routing logic fires for a warm resume

**Failure Indicators:**
- Splash screen is shown on resume (warm launch)
- App re-routes the user to Daily or onboarding on resume instead of restoring Chat
- Any white flash occurs during the resume transition

---

## Automation Notes

- **White flash detection:** Cannot be reliably detected via XCUITest assertions — requires manual screen recording at 60fps on a physical device; review recording frame by frame for any non-black frames during launch
- **Time-based routing:** Use `XCTestCase` with a mock clock injected into `AppRouter.swift` rather than manipulating device time; mock should allow setting `currentHour` to 7 (before 10am) and 11 (after 10am) without changing device settings
- **Keychain setup in tests:** Use a test helper to write/delete Keychain entries before each test case; never rely on leftover state from prior runs
- **Auth timeout simulation:** Inject a configurable delay into `AuthManager` via a protocol/mock; set delay to 2500ms to reliably trigger the 2-second timeout
- **Flakiness risk:** Cold-launch timing can be variable on a loaded device — allow up to 500ms margin when asserting the 300ms fade transition has completed; do not assert pixel-perfect timing

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

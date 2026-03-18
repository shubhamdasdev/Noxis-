# Story: App Launch & Splash Screen

---

**ID:** S-007-004
**Epic:** E-007
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **the app to launch instantly with a premium, clean entry experience**, so that **from the very first second Noxis feels intentional and high-end**.

## Goal

App launch shows a pure black splash screen with the Noxis wordmark that fades into the main app shell. For returning authenticated users, the app goes directly to Chat (or Daily if it's morning). For unauthenticated users, it routes to the onboarding Welcome screen. The transition is smooth with no white flashes or jarring jumps.

## Why / Rationale

The first 2 seconds of app launch set the tone for everything that follows. A cheap splash screen or a white flash undermines the premium identity before the user sees a single feature. This story locks in a consistent, brand-appropriate launch sequence.

## Functional Requirements

**Launch sequence:**
1. iOS launch screen (static): pure black background + Noxis wordmark centered (white, SF Pro Display Bold, 28pt)
2. App loads → auth state check (< 300ms target)
3. If **authenticated + returning user**:
   - If current time is before 10am (user's local timezone): navigate to Daily tab
   - Otherwise: navigate to Chat tab
4. If **unauthenticated / new user**: navigate to onboarding Welcome screen (S-001-001)
5. Transition: fade from launch screen to first screen (300ms ease-in-out)

**No white flash:**
- `LaunchScreen.storyboard` background = #000000
- SwiftUI root view background = #000000
- `UIWindow` background = #000000
- Set in `Info.plist`: `UIUserInterfaceStyle = Dark`

**Loading state (if auth check takes > 300ms):**
- Hold on the launch screen; do not show a spinner
- Max wait: 2 seconds — if auth check times out, route to onboarding as if unauthenticated

## Prerequisites

- S-007-003 (Design system — colors and typography)
- S-001-005 (Authentication) for auth state check

## Acceptance Criteria

- [ ] Given the app is launched fresh, when the splash renders, then the background is pure black with the Noxis wordmark centered — no white flash at any point
- [ ] Given an authenticated returning user launches the app before 10am, when the auth check completes, then the app opens to the Daily tab
- [ ] Given an authenticated returning user launches the app after 10am, when the auth check completes, then the app opens to the Chat tab
- [ ] Given an unauthenticated user launches the app, when the auth check completes, then the app routes to the Welcome onboarding screen
- [ ] Given the auth check takes longer than 2 seconds, when the timeout is hit, then the app routes to onboarding

## Negative Scenarios

- Auth service unavailable at launch → route to onboarding; user can log in again once connectivity is restored
- Corrupted local auth token → clear token, route to onboarding as unauthenticated

## Edge Cases

- User has just signed out → auth state is immediately unauthenticated; next launch routes to onboarding
- App is backgrounded and resumed (not cold launch) → skip splash, resume at last active screen

## Dependencies

- S-007-003
- S-001-005

## Technical Requirements

- **`LaunchScreen.storyboard`** — black background, Noxis wordmark image asset, no animations
- **`AppRouter.swift`** — checks `AuthManager.shared.isAuthenticated` on app init; routes accordingly
- **`RootView.swift`** — SwiftUI entry with `.background(Color.black)` and `.preferredColorScheme(.dark)`

## Future Scope

- Animated logo on launch (subtle, not flashy) — v2 polish pass

## Do Not Do

- In this story, do not build the onboarding flow — only the routing to it
- Do not show a loading spinner during auth check — hold on the launch screen

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

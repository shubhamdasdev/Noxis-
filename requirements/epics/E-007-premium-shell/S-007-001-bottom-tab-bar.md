# Story: Bottom Tab Bar — 5 Tabs with Center Camera Button

---

**ID:** S-007-001
**Epic:** E-007
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a persistent bottom navigation bar with Chat, Modules, Camera, Daily, and Profile tabs**, so that **I can reach any primary surface in a single tap and send a photo instantly from anywhere in the app**.

## Goal

An Instagram-style bottom tab bar is always visible across all primary screens. The center button is a dedicated camera button — larger, elevated, and distinct — that opens the device camera for instant photo capture. Navigation between tabs is instant with no loading states.

## Why / Rationale

The camera button at center-bar is the zero-friction gateway to Noxis's most sticky use case: snap an outfit, get a judgment. Making it central and prominent uses the same muscle memory as Instagram's create button. The tab bar gives the user a persistent mental model of the app's structure from first use.

## Functional Requirements

**Tab bar layout (left to right):**
1. **Chat** — speech bubble icon; navigates to Chat screen (E-004)
2. **Modules** — grid icon; navigates to Life Modules Dashboard (E-005)
3. **📷 Camera** — center, elevated circular button with camera icon; opens native iOS camera immediately
4. **Daily** — sun/calendar icon; navigates to Daily Brief screen (E-006)
5. **Profile** — person icon; navigates to Profile & Settings screen (S-007-005)

**Camera button behavior:**
- Tapping the camera button opens the native iOS camera (photo mode by default)
- After photo capture, the photo is sent directly to Chat with an auto-generated caption "Here's a photo — [module classification will detect type]"
- Camera dismiss without capture → returns to previous screen, no action taken
- Camera button does not have a tab-selected state — it is an action button, not a destination tab

**Tab selection states:**
- Active tab: icon + label, white, slightly larger
- Inactive tab: icon + label, muted grey (60% opacity)
- No badges or notification dots in v1 — clean look

**Bar appearance:**
- Black background with glass blur effect (UIBlurEffect — dark style)
- Hairline separator or subtle gradient between content and bar
- Tab bar respects iOS safe area (home indicator padding)
- No default iOS tab bar chrome — fully custom

## Prerequisites

- S-007-003 (Glass + Black Design System) for colors and blur styles

## Acceptance Criteria

- [ ] Given the user is on any primary screen, when they look at the bottom, then the tab bar is visible with 5 items including the center camera button
- [ ] Given the user taps the Camera button, when the camera opens, then the native iOS camera launches in photo mode immediately (no intermediate screen)
- [ ] Given the user captures a photo, when they confirm, then the photo appears in Chat as a new message ready for Noxis to respond
- [ ] Given the user dismisses the camera without a photo, when returned to the app, then the previous screen is restored with no change
- [ ] Given the user taps any navigation tab, when the screen transitions, then the new screen appears with no visible loading state
- [ ] Given the app is on iPhone with a home indicator, when the tab bar renders, then it sits above the home indicator with correct safe area inset

## Negative Scenarios

- Camera permission denied → show iOS permission dialog; if denied, show an in-app prompt explaining camera is needed for outfit/meal checks with a link to Settings
- Camera unavailable (simulator) → disable camera button with a visual indicator; do not crash

## Edge Cases

- User taps camera button while already in Camera screen → no duplicate camera opens
- User rapidly taps between tabs → transitions do not stack or stutter; last tapped tab wins

## Dependencies

- S-007-003

## Technical Requirements

- **Custom tab bar component** — do not use UITabBarController default; implement as a custom SwiftUI view pinned to bottom with ZStack overlay
- **Camera integration** — use `UIImagePickerController` or `PHPickerViewController` (iOS 17 minimum); photo is returned as `UIImage` and sent to chat pipeline
- **Blur effect** — `UIBlurEffect(style: .systemUltraThinMaterialDark)` applied as tab bar background

## Future Scope

- Long-press camera button to access video or library upload (v2)
- Badge count on Daily tab for unread brief (v2)

## Do Not Do

- In this story, do not implement the content of any tab screen — only the navigation shell and camera action
- Do not use the default iOS UITabBar component — the design requires full custom implementation

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

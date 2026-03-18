# Story: Profile Tab — Settings, Tone Switch, Account

---

**ID:** S-007-005
**Epic:** E-007
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a Profile tab where I can review my settings, switch my tone mode, and manage my account**, so that **I can adjust how Noxis works for me without hunting through menus**.

## Goal

The Profile tab is a clean, glass-card settings surface. The most important setting — tone mode — is shown prominently at the top. Below it: notification preferences, memory review link, and account actions (sign out, delete account). Everything is accessible in 2 taps or fewer from the tab bar.

## Why / Rationale

Tone switching is a first-class feature — not buried in settings. A user who started with "consultant" and now wants "brother" energy should be able to change it in 5 seconds. This screen also serves as the primary settings surface for the app.

## Functional Requirements

**Layout (top to bottom):**

1. **User section** (glass card):
   - Display name or "User" if not set
   - Email address (tappable to edit)

2. **Tone Mode** (glass card, prominent):
   - Label: "How Noxis talks to you"
   - Three glass pill buttons: Brother | Consultant | Peer
   - Active mode highlighted with accent color border
   - Tapping a pill immediately changes tone mode; no confirmation needed
   - Subtext: "You can change this anytime"

3. **Daily Brief** (glass card):
   - Toggle: "Morning Brief" on/off
   - Time picker: "Deliver at" — default 7:00am; user's local time
   - Push notification status indicator (enabled / disabled with Settings deep-link)

4. **Memory** (glass card):
   - "Review My Memory" row → navigates to Memory Review screen (S-003-007)
   - Shows count: "47 memories stored"

5. **Account** (glass card):
   - "Sign Out" row
   - "Delete Account" row (destructive red)

**Sign Out:**
- Tap → confirmation alert ("Sign out of Noxis?") → confirm → clear local auth token, clear local session, route to Welcome screen

**Delete Account:**
- Tap → confirmation alert with destructive warning ("This will permanently delete your account and all your memories. This cannot be undone.") → confirm → API call to delete account → route to Welcome screen

## Prerequisites

- S-007-003 (Design system)
- S-002-002 (Tone mode switching)
- S-001-005 (Authentication)

## Acceptance Criteria

- [ ] Given the user taps the Profile tab, when the screen renders, then tone mode, daily brief settings, memory count, and account options are all visible without scrolling on a standard iPhone
- [ ] Given the user taps a different tone mode pill, when tapped, then the active state updates immediately and the next Noxis response uses the new tone
- [ ] Given the user toggles "Morning Brief" off, when toggled, then no brief notification is sent the next morning
- [ ] Given the user taps "Sign Out" and confirms, when completed, then the user is signed out and routed to the Welcome screen with all local session data cleared
- [ ] Given the user taps "Delete Account" and confirms, when completed, then the account and all associated memories are deleted and the user is routed to the Welcome screen

## Negative Scenarios

- Delete account API call fails → show error toast: "Something went wrong. Please try again." — do not clear local data on failure
- Push notifications blocked at system level → show "Notifications disabled" status with a "Enable in Settings" link (deep-links to iOS Settings app)

## Edge Cases

- User rapidly taps between tone modes → last tap wins; no race condition on tone storage

## Dependencies

- S-007-003
- S-002-002
- S-001-005

## Technical Requirements

- **`ProfileView.swift`** — SwiftUI view with glass card sections
- **Tone persistence** — `UserDefaults` or `UserProfile` model; update propagates to `buildSystemPrompt()` immediately
- **Account deletion API** — `DELETE /user/{id}` endpoint; must purge all Mem0 memories, conversation history, and user profile on backend
- **Deep-link to iOS Settings** — `UIApplication.shared.open(URL(string: UIApplication.openSettingsURLString)!)`

## Future Scope

- Edit display name and profile photo (v2)
- Notification customization beyond daily brief on/off (v2)

## Do Not Do

- In this story, do not build the Memory Review screen — that is S-003-007
- Do not add social profile features or sharing capabilities

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

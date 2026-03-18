# Story: Context-Aware App Entry (Morning → Daily Tab)

---

**ID:** S-006-003
**Epic:** E-006
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **the app to open directly to my daily brief when I pick it up in the morning**, so that **the most relevant thing is in front of me immediately rather than having to navigate to find it**.

## Goal

On first app open of the day before 10am local time, if a new unread brief exists, the app lands on the Daily tab instead of Chat. After 10am, or if the brief has already been read, the app opens to Chat as normal. The user can override this behavior in Profile settings.

## Why / Rationale

Morning entry behavior is a subtle but high-impact retention feature. The brief is only useful if the user actually reads it. Landing on the Daily tab when a brief is ready removes the navigation step. It also signals that Noxis prepared something for the user — which is a differentiator from every other chat AI. The behavior is invisible when it applies and never intrusive.

## Functional Requirements

- On cold app launch (not from a push notification), the app determines the initial tab using the following logic:
  1. User must be authenticated (`user` role)
  2. Current local time must be before 10:00 AM on the current calendar day
  3. A brief for today must exist with `status: generated`
  4. The brief must have `viewed_at: null` (not yet read today)
  5. The user's setting `app_entry_override` must not be set to `always_chat`
  6. If all conditions are true → open to Daily tab
  7. Otherwise → open to Chat tab (default)

- The tab routing decision is made synchronously before the first frame renders; there must be no visible "tab switch" animation — the user simply sees the correct tab already active
- This logic only applies to cold launches; resuming from background does not change the active tab
- A user setting in Profile (S-007-005) "Open on" with two options: "Smart (morning brief)" and "Always Chat" controls the `app_entry_override` flag
- The setting defaults to "Smart (morning brief)"
- The 10am cutoff is evaluated using the device's current local time at the moment of launch — not the brief's `generated_at` time

## Prerequisites

- S-007-004 (app launch — the launch and auth flow must complete before this routing logic runs)
- S-006-001 (daily brief generation — briefs must exist to route to)

## Acceptance Criteria

- [ ] Given an authenticated user opens the app at 08:15 and an unread brief exists for today, when the launch completes, then the Daily tab is the active tab with no visible tab animation
- [ ] Given an authenticated user opens the app at 10:30, when the launch completes, then the Chat tab is the active tab regardless of whether an unread brief exists
- [ ] Given an authenticated user has already read today's brief (`viewed_at` is set), when they open the app at 09:00, then the Chat tab is the active tab
- [ ] Given the user has set their entry preference to "Always Chat", when they open the app at 07:00 with an unread brief, then the Chat tab is the active tab
- [ ] Given the user opens the app from a push notification, when the launch completes, then this routing logic does not apply — the notification's target screen handles navigation
- [ ] Given the user backgrounds the app and foregrounds it 2 minutes later, when the app resumes, then the active tab is unchanged — no re-evaluation of the routing logic
- [ ] Given a new user with no brief generated yet opens the app before 10am, when the launch completes, then the Chat tab is shown (no brief to route to)

## Negative Scenarios

- Brief status API call fails during launch → fall back to Chat tab; routing logic must not block the app launch; if the status cannot be determined within 500ms, default to Chat
- User clears app data / reinstalls and opens at 08:00 with a brief pre-generated on the server → after auth completes, the routing logic evaluates fresh; if brief exists and is unread, routes to Daily tab correctly

## Edge Cases

- User in a timezone with no DST at 09:59 and the brief was generated for "today" in a different timezone → routing uses device local time exclusively; server-side calendar date is not consulted
- Airplane mode at launch (no API available) → routing cannot check brief status; default to Chat tab; do not attempt a network call that will hang; use cached brief data if available in local storage

## Dependencies

- S-007-004 (app launch)
- S-006-001 (daily brief generation)

## Technical Requirements

- **`AppRouter.swift`** — `determineInitialTab() async -> AppTab` function; evaluates all 5 conditions listed above; timeout: 500ms hard limit; fallback: `.chat`
- **Conditions evaluation:**
  - `isAuthenticated`: from `AuthService.currentUser != nil`
  - `isBeforeTenAM`: `Calendar.current.component(.hour, from: Date()) < 10`
  - `hasUnreadBrief`: `GET /api/v1/briefs/today` → `brief.status == .generated && brief.viewed_at == nil`; cached in `UserDefaults` with a 5-minute TTL to avoid redundant network calls on rapid restarts
  - `entryOverride`: `UserDefaults.standard.string(forKey: "app_entry_override") != "always_chat"`
- **Tab selection:** `@State var selectedTab: AppTab = .chat` in `RootView`; set synchronously from `AppRouter.determineInitialTab()` before `RootView` appears using `task(priority: .userInitiated) { selectedTab = await AppRouter.determineInitialTab() }`
- **No tab animation:** initial tab is set before the view hierarchy is on screen; `TabView` selection is set in the same render pass as the initial draw — no `.animation` modifier on `selectedTab`
- **Background resume:** `ScenePhase.background → .active` transition does NOT re-evaluate routing; tab state is preserved
- **Profile setting:** `UserDefaults.standard.set("always_chat", forKey: "app_entry_override")`; cleared when toggled back to Smart; setting is synced to backend `user_preferences.app_entry_override`

## Future Scope

- Smarter routing based on detected usage patterns ("user always checks briefs at 06:45 → pre-warm the Daily tab")
- Multiple override options: "Always Daily", "Always Chat", "Smart"
- Widget: show the brief insight on the iOS home screen widget without opening the app

## Do Not Do

- In this story, do not implement the push notification routing — that is S-006-004
- Do not add any visual indicator or animation during the routing decision
- Do not re-evaluate routing on background-to-foreground transitions

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

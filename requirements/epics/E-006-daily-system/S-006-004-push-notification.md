# Story: Push Notification for Morning Brief

---

**ID:** S-006-004
**Epic:** E-006
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **to receive a push notification when my morning brief is ready**, so that **I know to open the app and read it without having to remember to check**.

## Goal

After the daily brief is generated, a push notification is dispatched to the user's device at their configured delivery time. The notification says "Your morning brief is ready." Tapping it opens the app directly to the Daily tab. No personal content is in the notification body.

## Why / Rationale

The brief only has value if it gets read. Push notifications are the primary delivery mechanism for creating a morning ritual around the brief. The notification body is intentionally non-specific — private content stays private; the notification is a knock on the door, not a shout through it.

## Functional Requirements

- When the `DailyBriefGenerator` job (S-006-001) successfully generates a brief, it immediately dispatches a push notification to the user's device via APNs (Apple Push Notification service)
- Push notification payload:
  - Title: "Noxis"
  - Body: "Your morning brief is ready."
  - Category identifier: `DAILY_BRIEF`
  - User info: `{ "target_tab": "daily", "brief_id": "<uuid>" }`
  - Sound: default iOS notification sound
  - Badge: 1 (cleared when the Daily tab is opened)
- Tapping the notification opens the app; the `AppDelegate` / `UNUserNotificationCenterDelegate` reads `target_tab: "daily"` and routes to the Daily tab regardless of current time or brief viewed state
- The push is sent only if the user has granted notification permission; no permission → no push sent (the brief is still generated and available in the app)
- iOS push permission is requested during onboarding (first app launch post-auth); standard `UNUserNotificationCenter.requestAuthorization(options: [.alert, .sound, .badge])` with a pre-prompt screen explaining the value ("Noxis will tap you when your brief is ready — nothing else")
- If the device token is not registered (new device, reinstall), the brief is still generated; the push is queued and retried for up to 1 hour as the device registers its token; if no token is available after 1 hour, the push is not sent
- Notification badge is set to 1 on delivery; cleared by `UNUserNotificationCenter.current().setBadgeCount(0)` when the app enters foreground and the Daily tab is first displayed

## Prerequisites

- S-007-004 (app launch — APNs device token registration must be implemented)
- S-006-001 (daily brief generation — notification is dispatched only after successful generation)

## Acceptance Criteria

- [ ] Given a brief is successfully generated, when the brief job completes, then a push notification with title "Noxis" and body "Your morning brief is ready." is dispatched within 60 seconds
- [ ] Given the user taps the push notification, when the app opens, then the Daily tab is the active tab and today's brief is visible — regardless of current time
- [ ] Given the user has denied notification permission, when the brief is generated, then no push is dispatched; the brief is still generated and accessible in-app
- [ ] Given the push notification arrives, when the user has not yet opened the app today, then the app icon badge shows 1
- [ ] Given the user opens the Daily tab after receiving the notification, when the tab is displayed, then the badge count is cleared to 0
- [ ] Given the app launches from the notification tap, when `AppDelegate` processes the notification payload, then `target_tab: "daily"` is correctly read and routes to the Daily tab without triggering the normal `AppRouter` routing logic
- [ ] Given the device has no registered APNs token at brief generation time, when the token is registered within 1 hour, then the queued push notification is delivered; after 1 hour without registration, the push attempt is abandoned

## Negative Scenarios

- APNs returns an error for the device token (token invalid / device unregistered) → the token is marked invalid in the backend `device_tokens` table; the push is not retried; the next valid token registration from this user overwrites the stale entry
- Brief generation succeeds but push dispatch fails (APNs unavailable) → the push failure is logged; the brief remains available in-app; no retry for push dispatch (the brief is already there when the user opens the app)

## Edge Cases

- User receives the notification while the app is already open and on the Daily tab → the notification is received but no in-app navigation occurs (already on the right screen); badge increment is suppressed via `UNUserNotificationCenterDelegate.willPresent`
- User has multiple devices registered (iPhone + iPad) → push is sent to all registered tokens for the user; all registered devices receive the notification

## Dependencies

- S-007-004 (app launch)
- S-006-001 (daily brief generation)

## Technical Requirements

- **APNs device token registration:** `AppDelegate.application(_:didRegisterForRemoteNotificationsWithDeviceToken:)` → `POST /api/v1/device-tokens` with `{ token: String, platform: "ios", environment: "production" | "sandbox" }`; stored in `device_tokens` table per user
- **Push dispatch:** `APNSService.send(userId: String, payload: APNSPayload)` → reads all valid tokens for the user from `device_tokens`; uses the APNs HTTP/2 provider API or a library like `APNSwift`; sends to all registered tokens
- **`APNSPayload`:** `{ aps: { alert: { title: "Noxis", body: "Your morning brief is ready." }, sound: "default", badge: 1, category: "DAILY_BRIEF" }, target_tab: "daily", brief_id: "<uuid>" }`
- **Pre-prompt screen:** `NotificationPermissionView.swift` — shown once after first successful auth; displays a single-sentence value prop and a "Turn on notifications" `GlassButton`; dismissed by tapping the button (which then calls `UNUserNotificationCenter.requestAuthorization`) or an "Not now" text link
- **Notification routing in AppDelegate:** `userNotificationCenter(_:didReceive:withCompletionHandler:)` reads `userInfo["target_tab"]`; if `"daily"`, posts `Notification.Name("openDailyTab")` that `RootView` observes to set `selectedTab = .daily`
- **Badge clear:** in `DailyTabView.onAppear` → `UNUserNotificationCenter.current().setBadgeCount(0)`; also remove all pending notifications from notification center with `removeAllDeliveredNotifications()`
- **Token retry queue:** `device_tokens` table has `last_validated_at`; APNs errors update `is_valid: false`; registration webhook refreshes token

## Future Scope

- Rich push notifications with a brief excerpt (user opt-in, since the content is personal)
- Push notification customization: user chooses notification style (silent badge only vs. alert)
- Scheduled quiet hours: user sets a window where no Noxis notifications arrive regardless of triggers

## Do Not Do

- In this story, do not put brief content or insights in the notification body — privacy by default
- Do not send any push notifications for chat messages or module updates in v1 — only the morning brief
- Do not implement notification grouping or threads in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

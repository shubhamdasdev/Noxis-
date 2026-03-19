# QA Test Plan — Push Notification for Morning Brief

---

**Plan ID:** T-006-004
**Story:** S-006-004
**Epic:** E-006
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the full push notification lifecycle for the morning brief: APNs device token registration on first app launch, dispatch of the push notification payload after successful brief generation, correct notification content (title "Noxis", body "Your morning brief is ready."), notification tap routing to the Daily tab, badge count management (set to 1 on delivery, cleared to 0 when Daily tab opens), permission gate (no push if denied), device token retry queue for unregistered devices, and multi-device push for users with multiple registered tokens. Error paths cover invalid APNs tokens and APNs service unavailability.

## Out of Scope

- Brief generation engine internals (S-006-001) — tested in T-006-001
- Daily brief display screen (S-006-002) — tested in T-006-002
- Rich push notifications with brief content (future scope)
- Push notification customization / quiet hours (future scope)
- Notification grouping or threading (not in v1)
- Chat message or module update push notifications (not in v1 per Do Not Do)

## Prerequisites

- S-007-004 (app launch) is implemented and APNs device token registration (`POST /api/v1/device-tokens`) is operational
- S-006-001 (daily brief generation) is implemented and can be manually triggered in the test environment
- `device_tokens` table is deployed with `is_valid`, `last_validated_at` columns
- APNs sandbox environment is configured in the test environment (or a mock APNs endpoint is available)
- `NotificationPermissionView.swift` is implemented (pre-prompt screen)
- Test user A: registered device token on record, push permissions granted, no brief for today
- Test user B: push permissions explicitly denied, no brief for today
- Test user C: no registered device token yet (fresh install), brief generation will be triggered
- Test user D: 2 registered device tokens (iPhone + iPad both registered)
- Test user E: has an invalid/stale APNs token registered

---

## Core Test Flow

### TC-006-004-001: Full happy path — brief generated, push dispatched with correct payload, tap routes to Daily tab, badge set then cleared

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-007-004, S-006-001

**Preconditions:**
- Test user A has a valid APNs token registered and push permissions granted
- No brief exists for today; the user has activity data (session logs + module data) to satisfy the activity gate
- Device/simulator is in background or lock screen state when the notification arrives
- APNs sandbox is configured; push receipt is observable via the test device or APNs mock logs

**Steps:**
1. Confirm test user A's device token is registered in `device_tokens` with `is_valid: true`
2. Trigger brief generation for test user A (manual test endpoint or wait for cron)
3. Confirm the brief is stored with `status: generated` in `daily_briefs`
4. Observe the push notification on the test device (or APNs mock log)
5. Inspect the notification payload
6. Observe the app icon badge count on the device home screen
7. Tap the push notification
8. Observe which tab is active after app opens
9. Observe the today card on the Daily tab
10. Observe the app icon badge count after the Daily tab opens

**Expected Result:**
- Step 3: brief created within 60 seconds of brief_delivery_time with `status: generated`
- Step 4: a push notification arrives within 60 seconds of brief generation completing
- Step 5: notification title = "Noxis"; body = "Your morning brief is ready."; `userInfo["target_tab"] = "daily"`; `userInfo["brief_id"]` = the UUID of today's brief; sound = default iOS notification sound; badge = 1; category = "DAILY_BRIEF"
- Step 6: app icon badge shows 1
- Step 7–8: the app opens and the Daily tab is immediately active; no other tab is shown first; today's brief is displayed
- Step 10: app icon badge is cleared to 0; all delivered notifications are removed from the notification center

**Failure Indicators:**
- Notification payload has wrong title or body text
- Notification does not arrive within 60 seconds of brief generation
- Tapping notification lands on a different tab (e.g., Chat or Home)
- Badge is not set to 1 on delivery
- Badge is not cleared after opening the Daily tab

---

## Sub Flows

### TC-006-004-002: Push permissions denied — no push sent, brief still generated and in-app

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-3
**Dependencies:** S-006-001

**Preconditions:**
- Test user B has explicitly denied notification permission (`UNAuthorizationStatus.denied`)
- Test user B has activity data; brief generation will run

**Steps:**
1. Trigger brief generation for test user B
2. Confirm brief `status: generated` in `daily_briefs`
3. Check APNs dispatch logs for test user B
4. Open the app as test user B and navigate to the Daily tab

**Expected Result:**
- Step 3: no push notification was dispatched for test user B (APNs call was not made)
- Step 4: today's brief is visible in the Daily tab (brief was generated and is accessible in-app)

**Failure Indicators:**
- A push notification is dispatched despite permission being denied
- The brief is not accessible in-app because no push was sent

---

### TC-006-004-003: Device token not registered at generation time — push queued and delivered when token registers within 1 hour

**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-7
**Dependencies:** S-007-004, S-006-001

**Preconditions:**
- Test user C has no device token registered at the time brief generation is triggered
- Brief generation is triggered (brief is generated and stored)
- 30 minutes later, test user C opens the app for the first time, triggering APNs token registration

**Steps:**
1. Trigger brief generation for test user C (no token registered)
2. Confirm brief `status: generated`
3. Confirm no push was dispatched at generation time (no token available)
4. 30 minutes later: simulate the app registering a device token for test user C (`POST /api/v1/device-tokens`)
5. Observe push delivery

**Expected Result:**
- Step 3: no immediate push dispatch; a queued record exists for the push
- Step 5: the push notification is delivered after the token registers (within the 1-hour window from generation time)
- Notification payload is identical to TC-006-004-001

**Failure Indicators:**
- Push is attempted at generation time with no token (should not happen)
- Push is not delivered after token registration within 1 hour

---

### TC-006-004-004: No registered token after 1 hour — push attempt abandoned

**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-7
**Dependencies:** S-006-001

**Preconditions:**
- Test user C has no device token registered
- Brief is generated; 1 hour elapses without token registration (advance test clock)

**Steps:**
1. Trigger brief generation for test user C
2. Advance test clock by 61 minutes without any token registration
3. Check push dispatch logs

**Expected Result:**
- No push notification was sent
- The push attempt is marked as abandoned in the backend (no retry beyond 1 hour)
- The brief is still accessible in-app if the user opens the app manually

**Failure Indicators:**
- Push continues to be retried beyond 1 hour
- Push is sent to an empty or null token

---

### TC-006-004-005: Invalid APNs token — token marked invalid, push not retried

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-007-004

**Preconditions:**
- Test user E has a stale/invalid device token in `device_tokens` (APNs will return an error for this token)
- Brief generation is triggered for test user E

**Steps:**
1. Trigger brief generation for test user E
2. Observe APNs response for the invalid token
3. Query `device_tokens` for test user E

**Expected Result:**
- APNs returns a token-invalid error
- Step 3: the token record is updated to `is_valid: false`
- No retry of the push for this token
- A subsequent valid token registration from this user (new device install) will create a new valid entry

**Failure Indicators:**
- The invalid token is retried multiple times
- `is_valid` is not updated to false after an APNs token-invalid response
- App crash or unhandled error when APNs rejects the token

---

### TC-006-004-006: APNs service unavailable — push failure logged, brief still accessible in-app

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-006-001

**Preconditions:**
- APNs endpoint is configured to return a 5xx error (mock)
- Test user A has a valid token and brief generation will succeed

**Steps:**
1. Trigger brief generation (APNs unavailable)
2. Confirm `daily_briefs` record for test user A
3. Check push dispatch error logs
4. Open the app and navigate to the Daily tab

**Expected Result:**
- Step 2: brief has `status: generated` (brief generation was unaffected by APNs failure)
- Step 3: push failure is logged; no retry for push dispatch
- Step 4: today's brief is visible and readable in the Daily tab

**Failure Indicators:**
- Brief generation fails because APNs was unavailable
- The push failure is retried repeatedly instead of being logged and abandoned
- The app shows an error because the push was not delivered

---

### TC-006-004-007: App already open on Daily tab — notification received, badge suppressed, no redundant navigation

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-006-001

**Preconditions:**
- Test user A is actively using the app and is on the Daily tab
- A brief push notification arrives while the app is foreground on the Daily tab

**Steps:**
1. Open the app and navigate to the Daily tab
2. Trigger brief generation and push dispatch (or simulate via test endpoint)
3. Observe the notification behavior

**Expected Result:**
- The push notification is received but the foreground notification is suppressed (via `UNUserNotificationCenterDelegate.willPresent` returning empty options or `.sound` only)
- No in-app navigation occurs (already on Daily tab)
- Badge is not incremented when the user is already viewing the Daily tab

**Failure Indicators:**
- An in-app notification banner appears and tapping it triggers redundant navigation
- Badge increments to 1 even though the user is already on the Daily tab

---

### TC-006-004-008: Multiple devices — push sent to all registered tokens

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-007-004

**Preconditions:**
- Test user D has 2 registered device tokens (both `is_valid: true`) in `device_tokens`
- Brief generation will be triggered for test user D

**Steps:**
1. Trigger brief generation for test user D
2. Check APNs dispatch logs

**Expected Result:**
- 2 separate APNs requests are made, one per registered token
- Both devices receive the push notification
- Each delivery uses the same payload

**Failure Indicators:**
- Only one push is sent even though 2 tokens are registered
- One of the tokens is skipped without reason

---

### TC-006-004-009: Onboarding — push permission pre-prompt screen is shown once after first auth

**Type:** Happy Path
**Priority:** P1
**AC Covered:** None (FR)
**Dependencies:** S-007-004

**Preconditions:**
- A brand new user completes authentication for the first time
- `NotificationPermissionView.swift` has not been shown to this user before

**Steps:**
1. Complete first-time authentication as a new user
2. Observe the screen that appears post-auth
3. Read the value proposition text
4. Tap "Turn on notifications"
5. Observe the iOS system permission dialog

**Expected Result:**
- Step 2: `NotificationPermissionView` is presented (not a system dialog directly)
- Step 3: a single-sentence value proposition is shown: "Noxis will tap you when your brief is ready — nothing else"
- Step 4: `UNUserNotificationCenter.requestAuthorization(options: [.alert, .sound, .badge])` is called
- Step 5: the standard iOS system permission dialog appears
- The pre-prompt screen is never shown again to this user on subsequent app launches

**Failure Indicators:**
- The iOS system permission dialog appears directly without the pre-prompt screen
- The pre-prompt screen is shown on every app launch
- Tapping "Not now" causes an error or blocks navigation

---

## Automation Notes

- APNs testing: use the APNs sandbox in the test environment; a mock APNs server (e.g., a local HTTP/2 server mimicking the APNs provider API) is preferred for CI to avoid dependency on Apple infrastructure
- Token registration: assert via `POST /api/v1/device-tokens` call inspection (URLProtocol observer or API log) rather than by reading the `device_tokens` table directly in XCUITest
- Notification tap routing: use `UNUserNotificationCenter` test helper to simulate notification tap with `userInfo = { "target_tab": "daily", "brief_id": "<uuid>" }`; assert `selectedTab == .daily`
- Badge count: assert via `UIApplication.shared.applicationIconBadgeNumber` in unit/integration tests; use `XCUIApplication().notifications` for banner appearance detection in XCUITest
- Foreground suppression: unit test `userNotificationCenter(_:willPresent:withCompletionHandler:)` delegate method directly to confirm options returned are `.sound` or empty (not `.banner`)
- Multi-device test: seed two tokens in the test database and assert that the APNs mock received exactly two requests
- Flakiness risk: APNs delivery latency in sandbox can be variable — use a mock APNs server for CI and reserve sandbox tests for device-level acceptance testing
- Framework: XCTest (unit for AppDelegate routing logic, badge clear, permission request) + XCUITest (E2E onboarding permission flow) + backend integration tests (token registration, push dispatch)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

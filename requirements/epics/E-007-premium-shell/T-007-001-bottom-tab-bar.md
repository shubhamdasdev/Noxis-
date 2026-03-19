# QA Test Plan — Bottom Tab Bar — 5 Tabs with Center Camera Button

---

**Plan ID:** T-007-001
**Story:** S-007-001
**Epic:** E-007
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the custom bottom tab bar implementation including all five tab items (Chat, Modules, Camera, Daily, Profile), the camera button's distinct elevated appearance, native camera launch behavior, post-capture photo routing to Chat, camera dismiss behavior, tab selection states, glass blur bar appearance, safe area compliance, and the two negative scenarios (camera permission denied, camera unavailable in simulator). All acceptance criteria are exercised in the core E2E flow; each negative and edge case receives a dedicated sub-flow case.

## Out of Scope

- Content of any tab screen — only the navigation shell and camera action are in scope (Do Not Do)
- Long-press camera button for video or library upload (Future Scope)
- Badge count on Daily tab for unread brief (Future Scope)
- Default iOS UITabBar usage — this plan assumes full custom implementation (Do Not Do)

## Prerequisites

- S-007-003 (Glass + Black Design System) is implemented and passing
- App has at least one stub screen per tab so navigation can be confirmed
- Camera permission can be reset via iOS Settings → Privacy → Camera
- Physical device or simulator with camera capability (for positive camera tests, physical device is preferred)

---

## Core Test Flow

### TC-007-001-001: Full tab bar navigation and camera photo-to-chat flow
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006
**Dependencies:** S-007-003

**Preconditions:**
- App is installed on a physical iPhone with camera permission granted
- App is on the Chat tab (first launch or known state)
- Device has iOS home indicator (Face ID device)

**Steps:**
1. Observe the bottom of the screen — confirm the tab bar is visible with exactly 5 items: Chat, Modules, Camera (center), Daily, Profile
2. Confirm the Camera button is visually distinct: larger, elevated circular button with a camera icon; it is not the same size or style as the four text tabs
3. Tap the **Modules** tab — confirm the Modules screen appears with no loading state or spinner
4. Tap the **Daily** tab — confirm the Daily screen appears instantly with no loading state
5. Tap the **Profile** tab — confirm the Profile screen appears instantly with no loading state
6. Tap the **Chat** tab — confirm Chat appears instantly; the Chat tab icon and label are white and slightly larger; all other tabs are muted grey (60% opacity)
7. Confirm no badge or notification dot is visible on any tab
8. Observe the tab bar position — confirm it sits above the iOS home indicator with the correct safe area inset; the bar does not overlap the home indicator
9. Tap the **Camera** button — confirm the native iOS camera launches immediately in photo mode with no intermediate screen or confirmation dialog
10. Capture a photo — confirm the photo after capture is sent to Chat as a new message with the auto-generated caption "Here's a photo — [module classification will detect type]"
11. Return to Chat and confirm the photo message is visible in the conversation

**Expected Result:**
- Tab bar is always visible with 5 items; Camera button is visually elevated and distinct from the four navigation tabs
- Tapping Chat, Modules, Daily, or Profile navigates instantly with no visible loading state; last-tapped tab shows white icon and label at slightly larger size; all others are 60% opacity muted grey
- No badges or notification dots appear on any tab in v1
- Tab bar sits above the home indicator with correct safe area inset; no iOS default chrome visible
- Camera button opens native iOS camera in photo mode with zero intermediate screens
- After photo capture and confirmation, the photo appears in Chat with the specified auto-generated caption
- Camera button has no tab-selected state — after returning from camera, the previously active tab remains selected

**Failure Indicators:**
- Tab bar is missing from any primary screen
- Camera button is the same size and style as the other tabs
- Any tab transition shows a loading spinner or takes more than one frame to render
- Camera launches in video mode or shows a pre-camera confirmation screen
- Photo does not appear in Chat after capture, or appears without the expected caption
- Tab bar overlaps the home indicator or uses default iOS UITabBar chrome

### TC-007-001-002: Camera dismissed without photo — previous screen restored
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-007-003

**Preconditions:**
- App is on the Modules tab (to confirm return to a non-Chat screen)
- Camera permission is granted

**Steps:**
1. While on the Modules tab, tap the Camera button
2. The native iOS camera opens
3. Dismiss the camera without capturing a photo (tap Cancel or swipe down)
4. Observe the app state after camera dismissal

**Expected Result:**
- The app returns to the Modules screen with no change — the same content and tab selection as before the camera was opened
- No photo appears in Chat
- No error or empty state is shown

**Failure Indicators:**
- App returns to Chat instead of the previously active tab
- A blank or error state appears after camera dismissal
- An empty message or placeholder is added to Chat

### TC-007-001-003: Camera permission denied — in-app prompt shown
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-007-003

**Preconditions:**
- Camera permission is set to Denied for the app in iOS Settings → Privacy & Security → Camera

**Steps:**
1. Tap the Camera button
2. Observe the system response and any in-app response

**Expected Result:**
- iOS does not launch the camera
- An in-app prompt appears explaining that camera access is needed for outfit and meal checks, with a button or link that opens iOS Settings directly to the app's permission page
- The app does not crash

**Failure Indicators:**
- App crashes or shows an unhandled error
- No in-app prompt appears — user sees nothing after tapping the camera button
- The prompt appears but the Settings deep link does not navigate to the correct permission page

### TC-007-001-004: Camera unavailable (simulator) — camera button is visually disabled
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-007-003

**Preconditions:**
- App is running on a simulator that does not have camera hardware

**Steps:**
1. Observe the Camera button in the tab bar
2. Attempt to tap the Camera button

**Expected Result:**
- The Camera button displays a visual disabled indicator (e.g. reduced opacity or alternative icon state)
- Tapping the button does nothing — the camera does not open and the app does not crash
- All other four tabs remain fully functional

**Failure Indicators:**
- App crashes when Camera button is tapped on a simulator
- Camera button appears fully active with no visual indication of unavailability
- Tapping the button shows an unhandled exception or blank screen

### TC-007-001-005: Camera button tapped while already in camera — no duplicate camera opens
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-002
**Dependencies:** S-007-003

**Preconditions:**
- Native camera is already open (launched from the tab bar)

**Steps:**
1. With the native camera open, attempt to tap the Camera button in the tab bar (if it is visible through the camera UI, or via any accessible gesture)

**Expected Result:**
- A second camera instance does not open
- The existing camera session remains active and unaffected

**Failure Indicators:**
- A second camera layer is pushed on top of the first
- App enters an inconsistent navigation state

### TC-007-001-006: Rapid tab tapping — last tapped tab wins, no stacking or stutter
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-005
**Dependencies:** S-007-003

**Preconditions:**
- App is on the Chat tab

**Steps:**
1. Rapidly tap Chat → Modules → Daily → Profile → Chat in quick succession (approximately 5 taps in 1 second)
2. After all taps register, wait for the UI to settle

**Expected Result:**
- The app lands on the Chat tab (the last tab tapped) with no intermediate screens stacked
- No animation stutter, duplicate screens, or navigation state corruption is visible
- The Chat tab icon and label show the active (white, larger) state; all other tabs are muted

**Failure Indicators:**
- The app ends up on a tab other than Chat
- Multiple screens appear stacked or a back button appears that should not be present
- Visible jank or dropped frames during the rapid-tap sequence

---

## Automation Notes

- **Tab navigation:** XCUITest can tap tab bar items by accessibility identifier — assign identifiers `tab-chat`, `tab-modules`, `tab-camera`, `tab-daily`, `tab-profile` to each item
- **Camera tests:** Cannot be fully automated via XCUITest on simulator; use `XCUIApplication().buttons["tab-camera"]` to verify the button's disabled state on simulator; physical device tests for camera launch must be manual
- **Photo-to-chat routing:** Automate with a mock `UIImagePickerController` that returns a test image; verify Chat receives the expected message object via the chat pipeline
- **Safe area:** Verify programmatically using `UIApplication.shared.windows.first?.safeAreaInsets.bottom` and asserting the tab bar frame origin.y accounts for this inset
- **Flakiness risk:** Camera permission state can be stale in the simulator — always reset permissions between test runs using `xcrun simctl privacy <device> reset camera <bundle-id>`

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

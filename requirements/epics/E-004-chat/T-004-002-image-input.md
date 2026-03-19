# QA Test Plan — Image Input: Camera Button & Photo Send

---

**Plan ID:** T-004-002
**Story:** S-004-002
**Epic:** E-004
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

Validates the full image input flow from camera icon tap through to the image appearing in the chat list with an upload spinner and subsequent success or failure handling. Covers action sheet options, photo library selection, camera capture, thumbnail preview, X-dismiss, compression loop, combined text+image send, upload timeout, and rapid-tap debounce. Does not cover image classification logic (S-004-003) or the AI vision response (S-004-005).

## Out of Scope

- Image classification and module-aware judgment (S-004-003)
- AI vision call or response content (S-004-005)
- Camera tab bar entry point (S-007-001)
- Multi-image selection — Future Scope
- Video support — Future Scope
- Drag-and-drop image attachment — Future Scope

## Prerequisites

- S-007-001 deployed so the bottom tab bar disambiguates from the center camera tab
- S-007-003 design system deployed: `GlassCard` component available
- Test device has at least one photo in the photo library
- Test device has a physical or simulated camera available
- Camera permission state is configurable (granted, denied) for negative tests
- Backend `/api/v1/media/upload` endpoint is reachable for upload tests; can be stubbed for failure/timeout tests
- Authenticated user session active

---

## Core Test Flow

### TC-004-002-001: Full image send happy path covering all acceptance criteria

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-007-001, S-007-003

**Preconditions:**
- Camera permission is granted
- Photo library contains at least one image under 10MB (post-compression)
- Backend upload endpoint returns success with `{ url, media_id }`
- User is on the Chat tab

**Steps:**
1. Tap the camera icon (SF Symbol `photo`) in the chat input bar.
2. Observe the action sheet that appears.
3. Tap "Choose from Library".
4. Observe the photo picker presentation.
5. Select a photo from the library.
6. Observe the input bar after selection.
7. Tap the X button on the thumbnail to dismiss the image.
8. Observe the input bar state.
9. Tap the camera icon again, choose "Choose from Library", and select the same photo.
10. Type "Check this out" in the `GlassInput` field.
11. Tap Send.
12. Observe the chat list immediately after sending.
13. Wait for the upload to complete.
14. Observe the final state of the image card in the chat list.

**Expected Result:**
- Step 2: Action sheet appears with exactly two options: "Choose from Library" and "Take Photo"; no other options.
- Step 4: `PHPickerViewController` is presented with single-image selection, filtered to images only.
- Step 6: A thumbnail preview of the selected image appears in the input bar; an X dismiss button is visible on the thumbnail; the image has not yet been sent.
- Steps 7-8: Tapping X removes the thumbnail; the input bar returns to its default state with no image attached.
- Steps 11-12: A right-aligned `GlassCard` image card appears in the chat list with a spinner overlay indicating upload in progress; above the card, the text bubble "Check this out" is also visible; the image card appears above the text bubble.
- Step 14: Spinner is replaced by the fully loaded image; the card remains right-aligned, bounded to 260pt wide, with subtle rounded corners.

**Failure Indicators:**
- Action sheet shows more than two options or uses different labels
- Photo picker allows selecting multiple images or non-image files
- No thumbnail preview appears after selection
- X button on thumbnail is absent or does not clear the image
- Image card does not appear in chat until upload completes (missing optimistic display of the card frame)
- Spinner never transitions to the final image on success
- Text bubble appears below the image card instead of above
- Image card width exceeds 260pt

---

## Sub Flows

### TC-004-002-002: "Take Photo" presents camera; camera permission denied shows Settings alert

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-5
**Dependencies:** S-007-001

**Preconditions:**
- Camera permission has been denied for the app in iOS Settings
- User is on the Chat tab

**Steps:**
1. Tap the camera icon in the input bar.
2. Tap "Take Photo" in the action sheet.
3. Observe what appears on screen.
4. Tap "Go to Settings" in the alert.
5. Observe where the user is taken.

**Expected Result:**
- Step 3: An alert appears (not a crash, not the camera itself) explaining that camera permission is required; the alert includes a "Go to Settings" button and a "Cancel" / dismiss option.
- Step 5: Tapping "Go to Settings" deep-links directly to the iOS Settings app and opens the Noxis permissions page where the user can enable camera access.

**Failure Indicators:**
- App crashes or presents a blank screen when camera permission is denied
- No alert is shown — camera silently fails to open
- "Go to Settings" button navigates to the iOS Settings root rather than the Noxis-specific permissions entry
- Alert lacks a dismissal option, trapping the user

---

### TC-004-002-003: Image upload failure shows retry overlay on the image card

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-6
**Dependencies:** S-007-003

**Preconditions:**
- Backend upload endpoint is configured to return a failure response (500 error or network unreachable) for this test
- A photo is available in the library
- User is on the Chat tab

**Steps:**
1. Select a photo via the library picker.
2. Tap Send.
3. Observe the image card while upload is in progress.
4. Allow the upload to fail.
5. Observe the image card state after failure.
6. Tap the retry button on the image card.
7. Restore the backend to a success state.
8. Observe the outcome after retry.

**Expected Result:**
- Step 3: Spinner overlay is present on the image card.
- Step 5: Spinner is replaced by a retry button overlay on the image card; no pipeline (classification, AI response) has been triggered.
- Step 8: On successful retry, the retry overlay disappears; the image loads; the pipeline proceeds to generate a Noxis response.

**Failure Indicators:**
- No retry button appears on failure — the card is silently stuck with a spinner
- Pipeline proceeds with a failed upload (Noxis responds without the image being available)
- The retry button crashes the app or produces a duplicate image card
- Raw HTTP error code or server message is shown to the user

---

### TC-004-002-004: Upload timeout (>30 seconds) shows timeout error with Retry and Cancel options

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-007-003

**Preconditions:**
- Backend upload endpoint is configured to delay indefinitely (or >30 seconds) for this test
- A photo is available in the library

**Steps:**
1. Select a photo and tap Send.
2. Observe the image card with the spinner.
3. Wait 30 seconds without the upload completing.
4. Observe the image card state after timeout.
5. Tap "Cancel".
6. Observe the chat list.

**Expected Result:**
- Step 4: After 30 seconds the spinner is replaced by a timeout error indicator on the image card; two options are displayed: "Retry" and "Cancel".
- Step 6: Tapping Cancel removes the unsent image card from the chat list; the input bar is re-enabled; the user can compose a new message.

**Failure Indicators:**
- Spinner runs indefinitely past 30 seconds with no timeout state
- Cancel does not remove the image card from the list
- Only one option (Retry or Cancel) is shown, not both
- App freezes or becomes unresponsive during the timeout

---

### TC-004-002-005: Compression loop runs for oversized images; rejects at minimum quality with clear error

**Type:** Negative / Edge Case
**Priority:** P1
**AC Covered:** AC-4
**Dependencies:** None (local compression logic)

**Preconditions:**
- A high-resolution image that compresses to above 10MB even at 0.7 JPEG quality is available in the test library
- A separate image that compresses to under 10MB at 0.7 quality is available for contrast

**Steps:**
1. Select the large image from the library.
2. Tap Send.
3. Observe compression behavior (verify via logs or proxy that multiple compression passes occur).
4. If the image still exceeds 10MB at quality 0.3, observe the user-facing error.
5. Select the normal-sized image and send it.
6. Observe that it sends without an error.

**Expected Result:**
- Steps 2-3: Compression loop runs through qualities 0.7 → 0.5 → 0.3; each pass is attempted before proceeding to the next.
- Step 4: If still over 10MB at 0.3, the image is rejected; a clear, user-readable error message is shown (not a raw exception); the input bar re-enables.
- Step 6: Normal image compresses successfully at 0.7 and sends without error.

**Failure Indicators:**
- App skips compression passes and immediately rejects or crashes on a large image
- Error message is a raw exception or contains technical details (byte counts, error codes) not suitable for users
- Input bar remains disabled after rejection

---

### TC-004-002-006: HEIC/RAW photo is converted to JPEG before compression loop runs

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** None

**Preconditions:**
- Test device has taken a HEIC photo (default iOS format) available in the library
- A large RAW image (40MB+) is optionally available if the test environment supports it

**Steps:**
1. Select a HEIC photo from the library.
2. Tap Send.
3. Observe that the image uploads without format errors.
4. Confirm via server response or proxy that the uploaded file is JPEG, not HEIC.

**Expected Result:**
- Step 3: Image sends successfully; no format-unsupported error is shown.
- Step 4: The uploaded file is a JPEG; HEIC conversion occurred transparently via `UIImage(data:)` before the compression loop.

**Failure Indicators:**
- Upload fails with a format or MIME type error
- App crashes or hangs when processing a HEIC file
- Backend receives a HEIC file rather than a JPEG

---

### TC-004-002-007: Rapid taps on camera button present picker only once

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** None

**Preconditions:**
- User is on the Chat tab
- No image is currently being previewed

**Steps:**
1. Rapidly tap the camera icon 5 times in quick succession (within ~500ms).
2. Observe how many times the picker is presented.

**Expected Result:**
- The photo picker or action sheet is presented exactly once; subsequent taps while the picker is already presented are ignored; no stacked pickers or duplicate sheets appear.

**Failure Indicators:**
- Multiple action sheets stack on top of each other
- The picker is presented more than once and the user must dismiss repeatedly
- App crashes due to multiple simultaneous presentation attempts

---

### TC-004-002-008: Non-image content is not selectable from the library picker

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** None

**Preconditions:**
- Photo library contains at least one video file

**Steps:**
1. Tap the camera icon and select "Choose from Library".
2. Attempt to select a video file in the picker.
3. Observe the picker's selection behavior for the video item.

**Expected Result:**
- The video item is not selectable (grayed out or absent from the picker); the picker's filter restricts to images only; no video can be attached.

**Failure Indicators:**
- Videos are visible and selectable in the picker
- Selecting a video causes an error or crash
- A video thumbnail appears in the input bar

---

## Automation Notes

- TC-004-002-001 end-to-end should be automated with XCUITest; use a mock upload endpoint via `URLProtocol` to avoid real network dependency.
- TC-004-002-002 (permission denied) requires resetting app permissions between runs; use `XCUIApplication.resetAuthorizationStatus(for: .photos)` and `.camera`.
- TC-004-002-004 (upload timeout) and TC-004-002-003 (upload failure) are best tested with a `URLProtocol` stub that delays or errors on the upload endpoint.
- TC-004-002-005 (compression loop) should be unit-tested: pass a synthetic oversized `UIImage` into the compression function and assert the quality sequence and rejection error.
- TC-004-002-006 (HEIC conversion) can be automated with a known HEIC asset in the test bundle; assert the uploaded data's MIME type via the stub.
- TC-004-002-007 (rapid tap debounce) can be automated with `XCUIElement.tap()` called in rapid succession; assert the presented view count is exactly one.
- TC-004-002-008 (non-image filter) can be verified by asserting `PHPickerConfiguration.filter == .images` in a unit test of the picker setup code.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

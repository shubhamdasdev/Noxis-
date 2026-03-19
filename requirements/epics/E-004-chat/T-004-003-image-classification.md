# QA Test Plan — Image Classification & Module-Aware Judgment

---

**Plan ID:** T-004-003
**Story:** S-004-003
**Epic:** E-004
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

Validates that the classification step correctly categorizes incoming images into outfit, meal, body, environment, or other; applies the appropriate SOUL.md judgment lens per category; enforces the brother tone for body photos regardless of tone mode setting; requests clarification when confidence is below 0.60; stores the `image_category` field on the message record; and handles gracefully the cases of API failure, blurry images, text screenshots, multi-person frames, and category ambiguity. Does not cover dashboard sync (S-005-007), module detail screens (E-005), or on-device ML.

## Out of Scope

- Dashboard sync to life module store (S-005-007)
- Module detail screen UI (E-005 stories)
- Real-time object detection or on-device ML
- Multi-label classification — Future Scope
- User-correctable classification override — Future Scope
- Automatic module gallery tagging — Future Scope

## Prerequisites

- S-004-002 deployed: image input uploads successfully and returns an image URL
- S-002-003 module routing logic is implemented
- S-002-004 judgment framework lenses are defined in SOUL.md (outfit, meal, body, environment)
- Backend `/api/v1/classify` endpoint is reachable; can be stubbed for controlled confidence and category values
- Test image set available: outfit photo, meal photo, body/fitness photo, environment photo, text screenshot, ambiguous food+gym photo, blurry photo, photo with multiple people
- Authenticated user session with at least one tone mode configured (not brother) to verify brother-tone override on body photos

---

## Core Test Flow

### TC-004-003-001: Full classification happy path — outfit, meal, body, and other each apply the correct judgment lens

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-5, AC-6, AC-7
**Dependencies:** S-004-002, S-002-003, S-002-004

**Preconditions:**
- Classification endpoint returns high-confidence results (confidence ≥ 0.80) for each test image
- User's configured tone mode is NOT brother (e.g. set to "direct" or "mentor") to validate brother-tone override on body photo
- Four test images prepared: (a) outfit, (b) meal, (c) body/fitness, (d) screenshot of text

**Steps:**
1. Send test image (a) — outfit photo — in chat.
2. Observe the Noxis response content.
3. Check the message record's `image_category` field (via backend admin or debug log).
4. Send test image (b) — meal photo.
5. Observe the Noxis response content.
6. Check the `image_category` field for this message.
7. Send test image (c) — body/fitness photo; confirm user's tone mode is not brother before sending.
8. Observe the Noxis response content and tone.
9. Check the `image_category` field for this message.
10. Send test image (d) — text screenshot.
11. Observe the Noxis response.
12. Check the `image_category` field.

**Expected Result:**
- Steps 2-3: Outfit response explicitly addresses fit quality, color coordination, and occasion appropriateness per the SOUL.md wardrobe lens; `image_category` = "outfit".
- Steps 5-6: Meal response leads with protein assessment and overall food quality evaluation; no style commentary; `image_category` = "meal".
- Steps 8-9: Body response is in the brother tone (direct, supportive, frank) regardless of the user's configured tone mode; it addresses visible muscle development, posture, or overall physical presentation; `image_category` = "body".
- Steps 11-12: Text screenshot response reads and responds to the text content naturally; no outfit, meal, or fitness lens is applied; `image_category` = "other".

**Failure Indicators:**
- Outfit photo triggers a nutritional response (wrong lens applied)
- Meal photo triggers a style critique
- Body photo response uses the user's configured non-brother tone instead of brother tone
- Text screenshot response attempts to apply a judgment framework
- `image_category` field is null or missing for any successfully classified message

---

## Sub Flows

### TC-004-003-002: Low-confidence classification (below 0.60) prompts Noxis to ask for clarification

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-4
**Dependencies:** S-004-002, S-002-003

**Preconditions:**
- Classification endpoint is stubbed to return `{ "category": "meal", "confidence": 0.45, "sub_tags": [] }`
- Or a genuinely ambiguous image is available (blurry, dark, or abstract)

**Steps:**
1. Send a photo with stubbed low-confidence classification.
2. Observe the Noxis response.

**Expected Result:**
- Noxis does not apply any judgment lens; instead it asks the user to clarify what the photo shows (e.g. "This one's a bit hard to read — can you tell me more about what you're showing me?"); no outfit, meal, or fitness critique appears.

**Failure Indicators:**
- Noxis applies a judgment lens despite confidence being below 0.60
- Noxis responds with a generic "I can't see this" error instead of a natural clarification request
- No response is generated at all

---

### TC-004-003-003: Blurry or dark image returns low confidence and prompts for a clearer shot

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-004-002

**Preconditions:**
- A blurry or heavily underexposed test image is available
- Classification endpoint returns low confidence for this image (or real vision model is used with the expectation of low confidence)

**Steps:**
1. Send the blurry/dark image in chat.
2. Observe the Noxis response.

**Expected Result:**
- Noxis responds with the specific message: "This one's a bit hard to read — can you send a clearer shot?" (or a close equivalent matching the specified copy); no judgment framework is applied.

**Failure Indicators:**
- Noxis applies a judgment lens to a blurry image
- Response is a generic error message rather than the specified clarification copy
- App errors out or no response is generated

---

### TC-004-003-004: Classification API failure falls through to "other" category silently

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR — Negative Scenario)
**Dependencies:** S-004-002

**Preconditions:**
- Classification endpoint is stubbed to return a 500 error or timeout (>10 seconds) for this test

**Steps:**
1. Send a photo (any clear image) in chat.
2. Observe the Noxis response.
3. Check server logs for a classification failure record.
4. Verify no error message is shown to the user.

**Expected Result:**
- Noxis responds naturally to the image without applying a specific judgment lens (falls through to `other` behavior); the user sees a coherent response with no indication of the classification failure; server logs contain a classification failure entry for monitoring.

**Failure Indicators:**
- An error message, spinner, or blank state is shown to the user
- Pipeline halts and no response is generated
- Classification failure is surfaced as a UI error rather than logged silently
- Noxis applies a random judgment lens despite the classification failure

---

### TC-004-003-005: Image with multiple visible people — Noxis only references the primary subject

**Type:** Negative / Edge Case
**Priority:** P1
**AC Covered:** AC-6
**Dependencies:** S-004-002

**Preconditions:**
- A photo containing the user alongside at least one other clearly visible person is available (e.g. a group photo or a photo of someone in a gym)

**Steps:**
1. Send the multi-person photo in chat.
2. Observe the Noxis response content.

**Expected Result:**
- Noxis response acknowledges the photo and may reference the primary subject (the user); it does not comment on, judge, describe, or name any other individuals visible in the frame; no comparative assessments are made between subjects.

**Failure Indicators:**
- Noxis describes, judges, or references anyone other than the primary subject
- Noxis makes comparative statements (e.g. "the person on the left looks better")
- Response avoids commenting on the photo entirely without explanation

---

### TC-004-003-006: Text screenshot is classified as "other" and Noxis reads the text content

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** S-004-002

**Preconditions:**
- A clear screenshot of a text-based document, label, or message is available

**Steps:**
1. Send the text screenshot in chat.
2. Observe the Noxis response.
3. Confirm `image_category` is "other" on the message record.

**Expected Result:**
- Noxis reads the text visible in the screenshot and responds to its content conversationally; no outfit, meal, or fitness framework is applied; `image_category` = "other".

**Failure Indicators:**
- Noxis ignores the text content and responds to the image as a photograph
- A judgment framework is forced onto a text screenshot
- `image_category` is set to anything other than "other"

---

### TC-004-003-007: Ambiguous image with two high-confidence categories — first classification result wins

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — Edge Case)
**Dependencies:** S-004-002

**Preconditions:**
- An image that could legitimately be classified as both meal and environment (e.g. a styled restaurant table scene) is available
- Classification endpoint is configured to return two potential categories both with confidence ≥ 0.60 (use a stub or the real model with a suitable image)

**Steps:**
1. Send the ambiguous image in chat.
2. Observe which judgment lens is applied in the Noxis response.
3. Confirm that exactly one category is applied (not a blended response).

**Expected Result:**
- Noxis applies only one judgment framework — the one returned first by the classification call; the response is coherent and does not blend meal and environment lenses; `image_category` reflects the winning category.

**Failure Indicators:**
- Noxis's response blends two different judgment lenses in the same reply
- `image_category` is null or set to "other" when both candidates were above threshold
- Pipeline errors out on an ambiguous classification result

---

### TC-004-003-008: Environment image produces a brief contextual comment without judgment

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR — functional requirement)
**Dependencies:** S-004-002

**Preconditions:**
- A clear photo of an environment (a room, a street, a café) with no people is available
- Classification endpoint returns `{ "category": "environment", "confidence": 0.85 }` for this image

**Steps:**
1. Send the environment photo in chat.
2. Observe the Noxis response.

**Expected Result:**
- Noxis may make a brief contextual comment about the environment (e.g. noting the setting) but does not apply a structured judgment framework; the tone is conversational, not evaluative; `image_category` = "environment".

**Failure Indicators:**
- Noxis applies the outfit, meal, or body judgment lens to an environment photo
- Noxis makes a critical or rating-style assessment of the environment
- No response is generated for an environment image

---

## Automation Notes

- TC-004-003-001 requires a stubbed classification endpoint to guarantee deterministic category and confidence values per test image; use `URLProtocol` or a backend test fixture.
- TC-004-003-002 and TC-004-003-003 (low confidence) are best automated with a stub returning `confidence: 0.45`; assert that the response body contains a clarification phrase rather than any judgment keyword.
- TC-004-003-004 (API failure) can be automated by stubbing the `/api/v1/classify` endpoint to return a 500; assert: (a) user sees a coherent response, (b) no error UI appears, (c) server log contains the failure entry.
- TC-004-003-005 (multi-person) should be manually verified with a real photo; automated content scanning is unreliable for this assertion — use a human review checklist.
- TC-004-003-007 (ambiguity) requires a controlled stub that returns two high-confidence categories; assert `image_category` matches the first result in the response array.
- Keyword-based assertions on response content (e.g. checking for "protein" in meal responses, absence of judgment language in environment responses) are appropriate for integration tests with a mocked AI response layer.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

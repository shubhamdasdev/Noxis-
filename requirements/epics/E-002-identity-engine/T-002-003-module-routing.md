# QA Test Plan — Module-Aware Response Routing (AGENTS.md)

---

**Plan ID:** T-002-003
**Story:** S-002-003
**Epic:** E-002
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates that every incoming message and image is correctly classified into one of the seven life modules before any memory retrieval or response generation occurs. Testing covers: per-module text classification, image classification by visual content, multi-module detection with primary/secondary routing, ambiguous message fallback to `general`, confidence score logging, out-of-domain routing, and independent routing of repeated questions across modules.

## Out of Scope

- Dashboard data extraction from classified responses (S-005-007)
- Memory retrieval using the classification result (S-003-004)
- Exposing the module classification label in the UI
- User-defined custom modules (v2)

## Prerequisites

- S-002-001 (SOUL.md system prompt loading) is complete
- `classifyMessage()` function is implemented and reachable from the pipeline
- Module context files (`modules/wardrobe.md`, `modules/gym.md`, `modules/food.md`, `modules/spending.md`, `modules/routines.md`, `modules/social.md`) are committed
- A test user account exists with chat access
- Classification result and confidence score are written to the server log on each call

---

## Core Test Flow

### TC-002-003-001: Classify text messages and images across all seven modules, verify multi-module routing, confirm logging
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-002-001

**Preconditions:**
- Test user is authenticated
- Log output is capturable for assertion

**Steps:**
1. Send a text message: "Check this outfit" — inspect the server log for `{ primaryModule: "wardrobe" }` and a confidence score; confirm wardrobe judgment framework instructions are appended to the prompt
2. Send an outfit photo (an image clearly showing clothing) — inspect the log for `{ primaryModule: "wardrobe" }` and confirm the wardrobe module context is active
3. Send a food photo (a clearly visible meal) — inspect the log for `{ primaryModule: "food" }` and confirm a nutritional assessment response is returned
4. Send: "What should I eat before the gym?" — inspect the log for `{ primaryModule: "food", secondaryModule: "gym" }` and confirm both food and gym module contexts are loaded into the prompt
5. Send: "I don't know what to do today" — inspect the log for `{ primaryModule: "general" }` and confirm a cross-module response is generated without a module-specific error
6. For each classification in steps 1–5, confirm the log entry contains both the module name and a numeric confidence score
7. Send messages covering `spending` ("Should I buy these $200 sneakers?"), `routines` ("What does my morning routine say about me?"), and `social` ("How do I approach networking events?") — confirm correct module classification for each

**Expected Result:**
- Step 1: `primaryModule: wardrobe`, confidence score present in log, wardrobe module rules in prompt
- Step 2: `primaryModule: wardrobe` from image content; not defaulting to `general`
- Step 3: `primaryModule: food`, nutritional assessment in response
- Step 4: `primaryModule: food`, `secondaryModule: gym` both in log and both contexts in prompt
- Step 5: `primaryModule: general`, valid cross-module response generated, no classification error surfaced to user
- Step 6: Log entries for all five messages contain module name and confidence score
- Step 7: `spending`, `routines`, `social` classified correctly

**Failure Indicators:**
- Any message classified to the wrong module
- Multi-module message returning only one module in the log
- Ambiguous message returning a module-specific error to the user
- Log entries missing confidence score
- Image classified as `general` when visual content clearly maps to a specific module

---

## Sub Flows

### TC-002-003-002: Image cannot be classified — default to `general`, ask user for clarification
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-002-001

**Preconditions:**
- An image with ambiguous or unrecognizable content is prepared (e.g., an abstract graphic)

**Steps:**
1. Send the ambiguous image
2. Observe the response and inspect the log

**Expected Result:**
- `primaryModule: general` in the log
- Response asks the user which area they want feedback on, in Noxis's voice — no classification error shown
- No crash or unhandled exception

**Failure Indicators:**
- Classification error surfaced to the user
- App crashes or times out
- Response assumes a specific module without any visual basis

---

### TC-002-003-003: Out-of-domain request — route to `general`, apply core philosophy
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-002-001

**Preconditions:**
- Test user is authenticated

**Steps:**
1. Send: "What's the best way to maintain my car?"
2. Inspect the log and the response

**Expected Result:**
- `primaryModule: general` in the log
- Response acknowledges the topic is outside Noxis's domain; Noxis's core philosophy (value-based realism, action over rumination) is applied if applicable
- No error or robotic refusal; response is in-character

**Failure Indicators:**
- Classification returns a specific life module (wardrobe, gym, etc.) incorrectly
- Response gives car-specific advice as if it were in-domain
- Response is a robotic "I cannot help with that"

---

### TC-002-003-004: Same question across multiple modules on the same day — independent routing, no cross-contamination
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-002-001

**Preconditions:**
- Test user is authenticated

**Steps:**
1. Send: "How do I improve?" in three separate messages, one immediately after the other
2. Before each message, update the conversation context to imply a different module (e.g., precede with "Regarding my gym routine", "Regarding my diet", "Regarding my morning routine")
3. Inspect the log for each of the three classification events
4. Inspect the three responses for module-specific content

**Expected Result:**
- The three classifications return `gym`, `food`, `routines` respectively
- Responses address the correct domain for each
- No response bleeds content from a different module's context

**Failure Indicators:**
- All three messages classified to the same module
- Response for "gym" context references food or routine content not prompted by the user
- Log shows module context from a prior message leaking into the next classification

---

### TC-002-003-005: Low-confidence classification — default to `general` without surfacing error
**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR coverage — FR: low confidence defaults to `general`)
**Dependencies:** S-002-001

**Preconditions:**
- A message that is highly ambiguous across modules is prepared (e.g., "I need a change")

**Steps:**
1. Send the ambiguous message
2. Inspect the log for `primaryModule` and confidence score
3. Observe the response

**Expected Result:**
- Confidence score is low (implementation-defined threshold); `primaryModule` is `general`
- Response is a reasonable cross-module reply in Noxis's voice
- No "I don't know what you mean" classification error surfaces

**Failure Indicators:**
- Module classification error message shown to user
- System forces a specific module despite low confidence
- Log entry missing confidence score

---

## Automation Notes
- **Classification inspection:** instrument the `classifyMessage()` function in test mode to write its output to an inspectable test fixture rather than relying solely on log parsing
- **Image tests:** prepare a test image library with one clearly labeled image per module (`test-outfit.jpg`, `test-meal.jpg`, `test-gym.jpg`) to avoid ambiguity in classification assertions
- **Multi-module assertion:** assert that the prompt assembly string contains both `## Active Module: food` and `secondaryModule: gym` blocks for the multi-module test case
- **Log assertion:** use a log-capture fixture to assert the `{ module, confidence }` fields on each call; do not rely on AI response content alone
- **Flakiness risk:** classification uses a separate AI call (possibly a cheaper model) — confidence scores will vary; assert module name correctness rather than exact confidence values; set a reasonable low-confidence threshold for the `general` fallback test

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

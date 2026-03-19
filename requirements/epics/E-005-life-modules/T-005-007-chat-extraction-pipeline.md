# QA Test Plan — Chat-to-Dashboard Data Extraction Pipeline

---

**Plan ID:** T-005-007
**Story:** S-005-007
**Epic:** E-005
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the server-side chat extraction pipeline that runs after each completed orchestration response: correct identification and writing of wardrobe items, workouts, meals, purchases, and routine suggestions from user messages; outfit entry creation when an `outfit` image category is present; graceful partial failure isolation (one module write failure does not affect others); non-blocking behavior (no visible delay or UI flicker in chat); the 3-item-per-category cap; `chat_extractions` log table recording; and handling of empty/irrelevant exchanges. Error cases cover extraction AI call failure and the 15-second timeout.

## Out of Scope

- Individual module detail screens (S-005-002 through S-005-006) — this plan tests only that data is correctly written; display is tested in those stories' plans
- Memory extraction pipeline (S-003-003) — runs concurrently but tested separately
- Extraction confidence scores per item (future scope)
- Bulk historical extraction (future scope)
- Real-time module update indicator in tab bar (future scope)
- Extracting data from Noxis's own response text

## Prerequisites

- S-004-005 (orchestration pipeline) is implemented; chat exchanges can be submitted end-to-end
- S-003-003 (memory extraction) is implemented and running concurrently
- S-005-001 (dashboard grid) module stores are deployed with all write endpoints:
  - `POST /api/v1/wardrobe/items`
  - `POST /api/v1/gym/workouts`
  - `POST /api/v1/food/meals`
  - `POST /api/v1/spending/purchases`
  - `POST /api/v1/routines/suggestions`
  - `POST /api/v1/wardrobe/outfits`
- `chat_extractions` table is deployed
- A test user is authenticated with an active chat session
- Backend logging is accessible for inspection in the test environment

---

## Core Test Flow

### TC-005-007-001: Full happy path — all five module types extracted and written in a single exchange

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-004-005, S-003-003, S-005-001

**Preconditions:**
- Test user sends a message that contains all five module types: "Just finished a leg day with heavy squats. I had grilled chicken and rice for lunch. Bought new gym shoes today — about $120. I'm going to start doing cold showers every morning. Also picked up a navy blazer."
- The orchestration pipeline completes successfully (Noxis response streamed to chat)
- No active network stubs blocking module write endpoints

**Steps:**
1. Submit the above message as the test user in chat
2. Wait for the Noxis response to finish streaming (stream complete event)
3. Wait up to 15 seconds for the extraction pipeline to run asynchronously
4. Query `GET /api/v1/gym/workouts` for the test user
5. Query `GET /api/v1/food/meals` for the test user
6. Query `GET /api/v1/spending/purchases` for the test user
7. Query `GET /api/v1/routines/suggestions` for the test user
8. Query `GET /api/v1/wardrobe/items` for the test user
9. Check the `chat_extractions` table for the pipeline run record

**Expected Result:**
- Step 2: the chat response streams and completes with no visible delay or UI flicker attributable to the extraction pipeline
- Step 4: a workout record exists with `type: legs`, `notes` containing "heavy squats", `source: chat_sync`, `logged_at` within the last 2 minutes
- Step 5: a meal record exists with `description` containing "grilled chicken and rice", `timing: lunch`, `source: chat_sync`
- Step 6: a purchase record exists with `description` containing "gym shoes", `category: fitness`, `source: chat_sync`
- Step 7: a routine suggestion record exists with `name` containing "cold shower every morning" (not a full habit — a RoutineSuggestion record)
- Step 8: a wardrobe item record exists with `description` containing "navy blazer", `category: jacket`, `source: chat_sync`
- Step 9: a `chat_extractions` row exists with `status: success`, all five module counts ≥ 1, `failed_modules` = empty array

**Failure Indicators:**
- Any module write is absent from the database despite being mentioned in the message
- `chat_extractions` record is missing or has `status: failed`
- The chat response was visibly delayed by the extraction pipeline
- A `workouts` entry is created for the future intention ("cold showers") instead of a `routine_commitments` entry

---

## Sub Flows

### TC-005-007-002: outfit image classification — OutfitEntry created with photo URL and Noxis rating

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-3
**Dependencies:** S-004-005

**Preconditions:**
- Test user sends a photo classified as `outfit` (imageCategory = "outfit") with a short message like "What do you think of this fit?"
- The Noxis response contains an evaluative phrase such as "This works well — clean lines and intentional layering"

**Steps:**
1. Submit a chat message with an outfit image
2. Wait for stream complete and extraction pipeline to finish
3. Query `GET /api/v1/wardrobe/outfits` for the test user

**Expected Result:**
- An `OutfitEntry` record exists with:
  - `photo_url` matching the submitted image URL
  - `noxis_rating` extracted from the Noxis response (the first evaluative sentence, e.g., "This works well — clean lines and intentional layering")
  - `chat_message_id` referencing the exchange
  - `created_at` within the last 2 minutes

**Failure Indicators:**
- No `OutfitEntry` is created
- `noxis_rating` is empty or contains the full response text rather than an extracted rating phrase
- `photo_url` is null or mismatched

---

### TC-005-007-003: Non-relevant exchange — no module writes made, no errors

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-6
**Dependencies:** S-004-005

**Preconditions:**
- Test user sends a philosophical message with no module-relevant content: "What do you think about the nature of time and whether the past can be changed?"
- Baseline module counts are known for the test user

**Steps:**
1. Submit the message in chat
2. Wait for stream complete and extraction pipeline to finish
3. Compare all module counts to baseline

**Expected Result:**
- No new records created in gym_workouts, food_meals, spending_purchases, routines_suggestions, or wardrobe_items
- `chat_extractions` record shows all count fields = 0 and `status: success` (not failed — successfully extracted nothing)
- No errors logged in the extraction pipeline

**Failure Indicators:**
- A spurious record is written to any module
- `chat_extractions` record has `status: failed` instead of success with zero counts

---

### TC-005-007-004: Partial failure isolation — one module write fails, others succeed

**Type:** Negative
**Priority:** P1
**AC Covered:** AC-7
**Dependencies:** S-004-005, S-005-001

**Preconditions:**
- `POST /api/v1/gym/workouts` is configured to return a 500 error (network stub for this endpoint only)
- Test user sends: "Just finished a push day. I had salmon and veggies for dinner."

**Steps:**
1. Submit the message in chat
2. Wait for stream complete and extraction pipeline to finish
3. Query `GET /api/v1/gym/workouts` for the test user
4. Query `GET /api/v1/food/meals` for the test user
5. Check the `chat_extractions` table row for this run

**Expected Result:**
- Step 3: no new workout record (write failed)
- Step 4: a meal record exists for "salmon and veggies" with `timing: dinner`, `source: chat_sync`
- Step 5: `chat_extractions` row has `failed_modules = ["gym"]`, `food_count = 1`, `status: partial`
- No user-facing error appears in the chat UI

**Failure Indicators:**
- The food meal write was also skipped because gym write failed
- `failed_modules` is empty despite the gym write failure
- An error banner or message appears in the chat interface

---

### TC-005-007-005: Extraction AI call fails — no writes made, generation_failed recorded

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-004-005

**Preconditions:**
- The extraction AI call (gpt-4o-mini / structured output endpoint) is configured to return a 500 error for this test user

**Steps:**
1. Submit a message with module-relevant content: "Went to the gym today, had pasta for lunch"
2. Wait for stream complete and extraction pipeline to attempt to run
3. Check the `chat_extractions` table for this run

**Expected Result:**
- `chat_extractions` row has `status: failed` and specifically a `generation_failed` metadata indicator
- No records are written to any module store
- No user-facing error appears in the chat UI
- The chat response itself was unaffected

**Failure Indicators:**
- Module records are written despite the extraction AI call failing
- `chat_extractions` row is absent entirely
- An error surfaces in the chat interface

---

### TC-005-007-006: 3-item-per-category cap — excess items are discarded, first 3 are written

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-004-005

**Preconditions:**
- Test user sends a message that the extraction AI would identify as containing 5 workouts (e.g., "I've done push day, pull day, legs, cardio, and upper body sessions all this week — been crushing it")
- Baseline workout count is known

**Steps:**
1. Submit the message in chat
2. Wait for extraction pipeline to finish
3. Count new workout records created for this user

**Expected Result:**
- Exactly 3 new workout records are created (not 4 or 5)
- No user-facing notification about the cap being applied
- `chat_extractions` row shows `gym_count = 3`

**Failure Indicators:**
- More than 3 workout records are created from one exchange
- Fewer than 3 records are created when 3+ are clearly extractable

---

### TC-005-007-007: Future intention is captured as routine_suggestion, not as a workout

**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-004-005

**Preconditions:**
- Test user sends: "I'm going to start going to the gym next week"

**Steps:**
1. Submit the message in chat
2. Wait for extraction pipeline to finish
3. Query `GET /api/v1/gym/workouts` for new records
4. Query `GET /api/v1/routines/suggestions` for new records

**Expected Result:**
- Step 3: no new workout records created (future intention, not a past event)
- Step 4: a `RoutineSuggestion` record is created with a description capturing the gym commitment

**Failure Indicators:**
- A workout is logged to the gym module from a future intention
- No routine suggestion is created

---

### TC-005-007-008: Ambiguous meal description — extracted as-is, not rejected

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-004-005

**Preconditions:**
- Test user sends: "had some bites earlier"

**Steps:**
1. Submit the message in chat
2. Wait for extraction pipeline to finish
3. Query `GET /api/v1/food/meals` for new records

**Expected Result:**
- A meal record is created with `description: "some bites"` (or similar verbatim extraction)
- The pipeline does not reject the entry for being ambiguous
- No extraction error is recorded in `chat_extractions`

**Failure Indicators:**
- No meal record created because the description was ambiguous
- `chat_extractions` status is `failed` due to ambiguous input

---

### TC-005-007-009: meal image classification — imageUrl attached to meal record

**Type:** Happy Path
**Priority:** P2
**AC Covered:** AC-3
**Dependencies:** S-004-005

**Preconditions:**
- Test user sends a photo classified as `meal` (imageCategory = "meal") with the message "this is what I had for lunch"

**Steps:**
1. Submit the chat message with meal image
2. Wait for extraction pipeline to finish
3. Query `GET /api/v1/food/meals` for the new record

**Expected Result:**
- A meal record is created with `description` extracted from the message
- The `photo_url` field on the meal record is populated with the submitted image URL

**Failure Indicators:**
- Meal record is created but `photo_url` is null
- No meal record is created at all

---

## Automation Notes

- Extraction pipeline is fully server-side — tests should be integration/backend tests querying the database directly or via authenticated API calls, not XCUITest
- Non-blocking assertion: measure chat stream completion time with and without the extraction pipeline active; assert the difference is < 500ms (extraction must not add latency to the chat response)
- Network stubs for partial failure: use per-endpoint mock routing in the test environment (e.g., feature flag or test-only middleware that injects errors for specific endpoints)
- AI call failure simulation: use a test environment variable to force the extraction AI call to return a 500; or mock the OpenAI client in the test context
- 3-item cap test: requires crafting a message where the AI would clearly extract 5+ items; validate by inspecting `chat_extractions.gym_count` and the actual DB rows
- `chat_extractions` table assertions: run all integration tests with a unique `pipeline_run_id` per test to isolate records
- Flakiness risk: extraction AI call introduces non-determinism — for automated tests, use the structured output mock rather than the live AI call; reserve live AI tests for manual/exploratory runs
- Framework: backend integration tests (e.g., Jest/Supertest or equivalent server-side test framework) + manual exploratory testing for live AI extraction quality

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

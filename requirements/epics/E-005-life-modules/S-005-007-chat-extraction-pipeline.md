# Story: Chat-to-Dashboard Data Extraction Pipeline

---

**ID:** S-005-007
**Epic:** E-005
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **my dashboard to update automatically based on what I tell Noxis in chat**, so that **I never have to log data twice — my conversation is my input**.

## Goal

After every completed AI response, a lightweight extraction pass identifies structured data mentioned in the exchange and writes it to the appropriate module store. Wardrobe items, workouts, meals, purchases, and habits all populate automatically from chat without the user touching the module screens. The dashboard feels alive without manual upkeep.

## Why / Rationale

Manual logging is the friction point that kills every habit tracking and life management app. Noxis already knows everything the user is talking about — extracting module data from that conversation is zero-extra-effort for the user. This pipeline is what makes the modules feel like a byproduct of using Noxis rather than a separate job to maintain.

## Functional Requirements

- The chat extraction pipeline runs after the orchestration pipeline completes step 6 (stream complete), dispatched concurrently with the memory extraction step (S-003-003)
- The extraction pass receives: `userId`, `userMessage`, `noxisResponse`, `imageCategory` (if applicable), `imageUrl` (if applicable)
- An AI extraction call analyzes the exchange and returns a structured JSON object identifying any module data present:
  ```json
  {
    "wardrobe_items": [
      { "description": "navy blazer", "category": "jacket" }
    ],
    "workouts": [
      { "type": "legs", "notes": "heavy squats today" }
    ],
    "meals": [
      { "description": "grilled chicken and rice", "timing": "lunch" }
    ],
    "purchases": [
      { "description": "new gym shoes", "amount": null, "category": "fitness" }
    ],
    "routine_commitments": [
      { "name": "cold shower every morning" }
    ]
  }
  ```
- Each field is an array; empty array if nothing found for that module
- Items are written to their respective module store endpoints:
  - `wardrobe_items` → `POST /api/v1/wardrobe/items` with `source: chat_sync`
  - `workouts` → `POST /api/v1/gym/workouts` with `source: chat_sync`, `logged_at: now`
  - `meals` → `POST /api/v1/food/meals` with `source: chat_sync`
  - `purchases` → `POST /api/v1/spending/purchases` with `source: chat_sync`
  - `routine_commitments` → `POST /api/v1/routines/suggestions` (creates a `RoutineSuggestion` record, not a habit — user confirms separately)
- If an `imageCategory` is present in the exchange:
  - `outfit` → additionally create an `OutfitEntry` record in the wardrobe module: `photo_url`, `noxis_rating` (extracted from the Noxis response text)
  - `meal` → additionally attach the `imageUrl` to the extracted meal entry if one was extracted
  - `body` → no module write (body photos feed memory only, not a module log)
- All module writes are asynchronous — they do not block the user or affect the chat display
- Extraction failures for individual modules do not propagate; if the workout write fails, the meal write still proceeds
- The extraction call has a maximum of 3 module data items per category per exchange (prevents over-extraction)
- A `chat_extractions` log table records each extraction run: `{ pipeline_run_id, user_id, extracted_counts_by_module, failed_modules[], timestamp }`

## Prerequisites

- S-004-005 (orchestration pipeline — this pipeline is dispatched as step 8 of the orchestration pipeline)
- S-003-003 (memory extraction — runs concurrently; both are dispatched after step 6)
- S-005-001 (dashboard grid — module stores must exist before data can be written to them)

## Acceptance Criteria

- [ ] Given the user says "just finished a leg day" in chat, when the extraction pass runs, then a workout of type `legs` is written to the gym module with `source: chat_sync` and appears in the gym log
- [ ] Given the user mentions "I picked up a new navy blazer today", when the extraction pass runs, then a wardrobe item with description "navy blazer" and category "jacket" is written to the wardrobe module with `source: chat_sync`
- [ ] Given the user sends a photo classified as `outfit`, when the extraction pass runs, then an `OutfitEntry` is created with the photo URL and the Noxis rating text extracted from the response
- [ ] Given the user says "I'll do cold showers every morning from now on", when the extraction pass runs, then a `RoutineSuggestion` is created — not a full habit; the suggestion card appears on the Routines screen
- [ ] Given the extraction pass runs, when the chat stream is still visible to the user, then the extraction does not cause any visible delay or UI flicker in the chat response
- [ ] Given an exchange contains nothing module-relevant (a philosophical discussion), when the extraction pass runs, then no module writes are made and no errors occur
- [ ] Given the workout module write fails, when the extraction run continues, then the meal and wardrobe writes still proceed; only the failed write is recorded in `chat_extractions.failed_modules`

## Negative Scenarios

- The extraction AI call itself fails → no module data is written for that exchange; a `generation_failed` status is recorded in `chat_extractions`; no user-facing error
- User mentions a workout and a meal in the same message but both exceed the 3-item cap → the first 3 items per category are processed; the rest are discarded for that exchange; no user-facing notification

## Edge Cases

- User's message contains a future intention rather than a current fact ("I'm going to start going to the gym next week") → the extraction prompt distinguishes past/present events from future intentions; future intentions generate `routine_commitments` suggestions, not `workouts`
- User describes a meal with an unusual or ambiguous name ("had some bites") → the meal is extracted with the description as-is; the quality-tag generation step (S-005-004) handles ambiguity; the extraction pipeline does not reject incomplete descriptions

## Dependencies

- S-004-005 (orchestration pipeline)
- S-003-003 (memory extraction)
- S-005-001 (dashboard grid)

## Technical Requirements

- **`ChatExtractionPipeline.swift` (backend)** — `extractModuleData(userId: String, userMessage: String, noxisResponse: String, imageCategory: ImageCategory?, imageUrl: String?) async` — dispatched from orchestration pipeline via `Task { await ChatExtractionPipeline.run(...) }`
- **Extraction AI call:** `gpt-4o-mini` or equivalent; max tokens: 400; structured output / function calling returning the JSON schema above; prompt: "Identify structured life data from this conversation that should be logged to the user's life modules. Extract only events that have already happened or are currently happening — not future intentions (those go in routine_commitments). Maximum 3 items per category. Return an empty array for categories with nothing to extract."
- **Outfit entry creation:** `if imageCategory == .outfit { let rating = extractRatingFromResponse(noxisResponse); POST /api/v1/wardrobe/outfits }` — rating extraction uses a regex pattern for the first evaluative sentence of the response
- **Concurrent writes:** each module write is dispatched as a separate `Task`; `await withTaskGroup` collects results; failures are caught per-task and recorded
- **`chat_extractions` table:** `(id UUID, pipeline_run_id UUID, user_id STRING, wardrobe_count INT, gym_count INT, food_count INT, spending_count INT, routine_suggestions_count INT, failed_modules TEXT[], timestamp TIMESTAMP, status ENUM(success, partial, failed))`
- **3-item cap:** `candidates.prefix(3)` per category before write dispatch
- **Timeout:** extraction AI call: 15-second hard timeout; on timeout, status = `failed`, no writes made

## Future Scope

- Extraction confidence scores per item: low-confidence extractions create a "Did Noxis get this right?" confirmation card in the relevant module
- Bulk historical extraction: run the pipeline retroactively over past 30 days of chat history when a user first enables a module
- Real-time module update indicator: a subtle flash on the relevant module card icon in the tab bar when a new item is synced from chat

## Do Not Do

- In this story, do not implement the individual module detail screens — those are S-005-002 through S-005-006
- Do not extract data from Noxis's own responses — only from user messages; Noxis's words are not user facts
- Do not block or delay the chat response on extraction completion

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

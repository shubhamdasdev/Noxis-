# Story: Memory Extraction from Chat Conversations

---

**ID:** S-003-003
**Epic:** E-003
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to remember what I tell it across conversations without me having to repeat myself**, so that **the more I use it, the sharper and more personal its responses become**.

## Goal

After every completed Noxis response, a post-generation extraction pass analyzes the exchange for new facts, preferences, decisions, and context worth remembering. Extracted items are written to Mem0 with the appropriate category and confidence score. Over time, the memory layer becomes an accurate, growing model of the user's life.

## Why / Rationale

Memory extraction is the compounding mechanism. Without it, every conversation starts from the same baseline regardless of what was shared. With it, a user who mentions they train fasted, hate synthetic fabrics, and always overspend on food in January builds a progressively richer context that shapes every future response. This is what makes Noxis valuable over time rather than just at first use.

## Functional Requirements

- After the orchestration pipeline completes step 6 (stream complete), a background extraction task is triggered with: `userId`, `userMessage`, `noxisResponse`
- The extraction task sends a lightweight AI call with a structured extraction prompt to identify memory candidates from the exchange
- The extraction prompt instructs the model to return a JSON array of memory candidates:
  ```json
  [
    {
      "content": "User hates slim-fit jeans — says they cut off circulation",
      "category": "wardrobe",
      "confidence": 0.92
    }
  ]
  ```
- Valid extraction targets include: stated preferences, aversions, habits, goals, decisions, physical facts, social context, recurring patterns
- The extraction model must NOT extract: generic conversation filler, acknowledgements, questions Noxis asked, or information Noxis stated (only user-originated facts)
- Each candidate with `confidence >= 0.65` is written to Mem0 via `MemoryService.addMemory`
- Candidates with `confidence < 0.65` are discarded silently
- Maximum 5 new memories extracted per exchange — prevents over-extraction from a single long message
- If the same fact already exists in memory (idempotency check in `MemoryService.addMemory`), the confidence is updated if the new value is higher; no duplicate is created
- Extraction runs asynchronously after the response is delivered — it must not delay or interrupt the streaming response (step 6 of the pipeline)
- Extraction failures are logged but never surfaced to the user

## Prerequisites

- S-004-005 (orchestration pipeline — extraction is dispatched as step 7 of the pipeline after the stream completes)
- S-003-001 (memory storage — extracted candidates are written via MemoryService)

## Acceptance Criteria

- [ ] Given the user mentions a clear preference ("I always wear oversized fits"), when the extraction pass runs, then a memory with content reflecting that preference is stored under the `wardrobe` category with confidence >= 0.65
- [ ] Given the user states a routine fact ("I train at 6am every weekday"), when the extraction pass runs, then a memory with that fact is stored under the `routines` category
- [ ] Given an exchange contains no extractable user facts (e.g., a purely philosophical discussion), when the extraction pass runs, then no memories are written and no error occurs
- [ ] Given 6 or more extractable facts are present in a single exchange, when the extraction pass runs, then no more than 5 memories are written — the lowest-confidence candidates are dropped first
- [ ] Given the extraction task is dispatched, when the user is viewing the streaming response, then the extraction task does not delay the text stream or re-enable of the input bar
- [ ] Given a memory with identical content already exists, when a new extraction produces the same fact with higher confidence, then the existing memory's confidence and `updated_at` are updated rather than creating a duplicate
- [ ] Given an extraction AI call fails or times out, when the failure occurs, then the user sees no error and no memory is written for that exchange; the failure is written to the server error log

## Negative Scenarios

- The extraction model hallucinates a fact not present in the conversation (confidence inflation) → the idempotency check and the 0.65 threshold are the primary safeguards; a monitoring job flags extraction calls where all candidates have confidence > 0.95 (likely hallucination) for manual review
- User intentionally tries to write false memories ("remember that I hate working out") → extracted at face value if confidence threshold is met; contradiction resolution (S-003-005) handles conflicts with existing memories later; Noxis does not validate subjective user claims

## Edge Cases

- User message is a single emoji or a very short response ("k", "yeah") → extraction model returns empty array; no memories written; this is correct behavior
- Exchange happens in a non-English language → extraction prompt instructs the model to normalize facts to English before returning the JSON array; stored memory content is always in English

## Dependencies

- S-004-005 (orchestration pipeline)
- S-003-001 (memory storage)

## Technical Requirements

- **Extraction function:** `extractMemories(userId: String, userMessage: String, noxisResponse: String) async` — called from pipeline step 7 via `Task { await extractMemories(...) }`; runs fully asynchronously
- **Extraction AI call:** same AI provider as the main pipeline; model: `gpt-4o-mini` or equivalent; max tokens: 300; response format: JSON array via structured output / function calling
- **Extraction system prompt:** "You are a memory extractor. Given a user message and an AI response, identify facts about the user worth remembering for future conversations. Only extract user-stated facts — not AI-stated information. Return a JSON array with fields: content (string), category (one of: wardrobe, gym, food, spending, routines, social, general), confidence (0.0-1.0). Maximum 5 items. If nothing is worth extracting, return an empty array."
- **Timeout:** 15-second hard timeout on the extraction AI call; failure is caught and logged
- **Confidence gate:** `candidates.filter { $0.confidence >= 0.65 }.prefix(5)` before writing
- **Category validation:** extracted category must be a valid `MemoryCategory` enum value; invalid categories default to `general`
- **Dispatch:** `Task { await extractMemories(...) }` is called in `OrchestrationPipeline.run` immediately after the stream completion event; not `await`-ed on the main pipeline path

## Future Scope

- Confidence recalibration: if a user repeatedly states a fact across multiple sessions, its confidence is boosted toward 1.0 automatically
- Extraction review panel in admin dashboard: shows a sample of recent extractions for quality monitoring
- User-triggered memory correction: "That's not right — I don't hate slim fit any more" → user can correct Noxis mid-conversation and it updates the memory in real time

## Do Not Do

- In this story, do not implement contradiction resolution — that is S-003-005
- Do not display extracted memories to the user during or after extraction
- Do not block or delay the chat response on extraction completion

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

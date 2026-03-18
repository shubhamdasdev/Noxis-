# Story: Decision Policy — Relevance & Quality Filtering

---

**ID:** S-003-004
**Epic:** E-003
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to use memory selectively and accurately rather than dumping everything it knows into every response**, so that **responses stay focused, relevant, and never feel like Noxis is reciting my profile back at me**.

## Goal

Every set of retrieved memory items passes through a three-stage filter before reaching the system prompt: relevance filter (module match + recency weighting), quality filter (confidence threshold + contradiction flag), and capacity limit (max 20 items). SOUL.md always overrides conflicting memory. The filtered set is what the orchestration pipeline receives.

## Why / Rationale

Raw memory retrieval returns up to 30 candidates sorted by vector similarity — but similarity alone is not enough. A memory about spending habits should not appear in a gym conversation. A low-confidence fact should not influence a response. An old memory that contradicts a newer one should not be injected unchecked. The decision policy is the quality gate that keeps Noxis sharp instead of verbose.

## Functional Requirements

**Stage 1 — Relevance Filter:**
- Each retrieved memory item has a `module` tag from its category
- If the current conversation module (from pipeline step 1) is a specific module (e.g., `wardrobe`), only memories with `category: wardrobe` or `category: general` pass through
- Recency weight: memories updated within the last 7 days receive a +0.10 relevance bonus; memories older than 90 days receive a -0.10 penalty; the adjusted score is used to rank the filtered set
- `general` category memories always pass the module filter; they are ranked below module-specific memories

**Stage 2 — Quality Filter:**
- Drop any memory item with `confidence < 0.50`
- Flag any memory item that semantically contradicts another item in the same retrieved set — flagged pairs are reduced to the more recently updated item only (the older one is removed from the passing set)
- Contradiction detection uses a lightweight semantic comparison: if two items share the same subject and have opposing predicate sentiment (e.g., "loves slim fit" vs. "hates slim fit"), the older one is flagged and removed

**Stage 3 — Capacity Limit:**
- After stages 1 and 2, sort remaining items by adjusted relevance score descending
- Pass the top 20 items to the prompt; discard the rest for this request (they remain in Mem0)

**SOUL.md Override Rule:**
- If any retrieved memory item conflicts with an explicit SOUL.md instruction (e.g., a boundary or a voice constant), the SOUL.md instruction takes precedence and the conflicting memory item is removed from the set before injection
- This check happens after stage 3; the implementation compares memory content against a pre-parsed list of SOUL.md rules

## Prerequisites

- S-003-001 (memory storage — memories must exist to be filtered)
- S-002-001 (SOUL.md — required for the override rule)

## Acceptance Criteria

- [ ] Given a gym-context conversation and 30 retrieved memories across all categories, when the relevance filter runs, then only `gym` and `general` category memories remain in the passing set
- [ ] Given a memory item has `confidence: 0.45`, when the quality filter runs, then that item is excluded from the passing set regardless of its relevance score
- [ ] Given two retrieved memories conflict ("user loves early morning training" vs. "user trains at night"), when the quality filter runs, then only the more recently updated item is included in the passing set
- [ ] Given 25 memories pass stages 1 and 2, when the capacity limit runs, then exactly the top 20 by adjusted relevance score are passed to the system prompt
- [ ] Given a memory item conflicts with a SOUL.md boundary instruction, when the override check runs, then the memory item is removed from the set and the SOUL.md instruction governs the response
- [ ] Given all 30 retrieved memories are below the confidence threshold, when the quality filter runs, then zero memories are passed and the pipeline proceeds with USER.md context only
- [ ] Given a memory updated within the last 7 days, when the relevance score is computed, then it receives a +0.10 bonus visible in the filter's scoring log

## Negative Scenarios

- All retrieved memories are filtered out by stage 1 (wrong module) → pipeline proceeds with USER.md core profile and zero memories; response is still generated; this is not an error condition
- Contradiction detection produces a false positive (removes a valid memory) → the more recent memory is retained, which is the safer default; no user-facing impact; quality of the edge case is monitored via extraction audit logs

## Edge Cases

- Message module cannot be determined (ambiguous general conversation) → all categories pass the relevance filter; stage 2 and 3 apply normally; no module-specific bonus/penalty is applied
- Two memories have identical `updated_at` timestamps and conflict → the higher-confidence item is retained; if confidence is also equal, retain the first one in the sorted set (deterministic tie-break)

## Dependencies

- S-003-001 (memory storage)
- S-002-001 (SOUL.md)

## Technical Requirements

- **`DecisionPolicy.swift` (backend)** — a pure function `filter(memories: [Memory], module: LifeModule, soulRules: [SoulRule]) -> [Memory]` with three internal passes; returns a sorted, capped array
- **Recency scoring:** `adjustedScore = baseScore + (daysSinceUpdate <= 7 ? 0.10 : daysSinceUpdate >= 90 ? -0.10 : 0.0)`
- **Contradiction detection:** `SemanticContradictionDetector` — runs a lightweight embedding cosine similarity check between pairs in the retrieved set; pairs with similarity > 0.85 and opposing sentiment are flagged; implementation: compare via a small classifier or the same AI provider with a 2-token output call
- **SOUL.md rule parsing:** `SoulRuleParser` reads SOUL.md once at startup; extracts boundary and voice constant rules as a `[SoulRule]` array; cached until SOUL.md version changes
- **Module mapping:** `LifeModule` enum: `wardrobe | gym | food | spending | routines | general`; maps 1:1 to `MemoryCategory`
- **Performance target:** `filter()` must complete in under 50ms for 30 input items; contradiction detection is the expensive step — if over budget, skip contradiction detection and log a warning
- **Logging:** `DecisionPolicyResult` struct logged per pipeline run: `{ input_count, post_stage1_count, post_stage2_count, post_stage3_count, soul_overrides_count }`

## Future Scope

- Dynamic threshold adjustment: lower the confidence threshold for users with sparse memory (< 20 items total) to ensure the pipeline has something to work with
- Contradiction resolution surfacing: when a contradiction is detected and resolved, optionally surface it to the user in the next response ("I noticed something changed — updating what I know about you")
- Per-user policy tuning: power users can adjust how many memories flow into each request (slider in advanced settings)

## Do Not Do

- In this story, do not implement the memory extraction process — that is S-003-003
- Do not implement the contradiction resolution user-facing flow — that is S-003-005
- Do not modify or delete memories during the filter pass — the filter is read-only; it only determines what is injected for this request

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

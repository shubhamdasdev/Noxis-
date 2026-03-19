# QA Test Plan — Decision Policy — Relevance & Quality Filtering

---

**Plan ID:** T-003-004
**Story:** S-003-004
**Epic:** E-003
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the three-stage memory decision policy (relevance filter, quality filter, capacity limit) and the SOUL.md override rule. Testing covers: module-based relevance filtering with `general` category pass-through, recency bonuses and penalties, confidence threshold exclusion, contradiction resolution to the more-recent item, capacity cap at 20 items, SOUL.md override removing conflicting memories, all-memories-filtered fallback to USER.md, false-positive contradiction handling, ambiguous-module all-categories-pass behavior, identical-timestamp conflict tie-breaking, performance target under 50ms, and logging of the `DecisionPolicyResult` struct. The filter is read-only — no memory modifications during the filter pass.

## Out of Scope

- Memory extraction process (S-003-003)
- Contradiction resolution user-facing flow (S-003-005)
- Modifying or deleting memories during the filter pass — this is read-only
- Dynamic threshold adjustment for sparse memory users (future scope)
- Per-user policy tuning (future scope)

## Prerequisites

- S-003-001 (memory storage) is complete — memories must exist in Mem0 to be filtered
- S-002-001 (SOUL.md) is complete — required for the override rule
- `DecisionPolicy.filter(memories:, module:, soulRules:)` function is implemented
- `SoulRuleParser` is implemented and caches parsed rules at startup
- `SemanticContradictionDetector` is implemented
- `DecisionPolicyResult` logging is implemented
- A mechanism to inspect the filter's output (logging or test fixture hook) is available

---

## Core Test Flow

### TC-003-004-001: Relevance filter, quality filter, capacity limit, SOUL.md override, and logging — full pipeline
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006, AC-007
**Dependencies:** S-003-001, S-002-001

**Preconditions:**
- Test user has 30 memories prepared across categories:
  - 15 `gym` memories (confidence 0.60–0.95, mix of recent and old)
  - 8 `wardrobe` memories (confidence 0.60–0.90)
  - 5 `general` memories (confidence 0.70–0.90)
  - 2 `food` memories with confidence 0.40 and 0.48 (below threshold)
  - 1 `gym` memory conflicting with a SOUL.md boundary (e.g., content encouraging the user to skip training — contradicts consistency principle)
  - 1 pair of contradicting `gym` memories: "User loves early morning training" (older, confidence 0.80) vs. "User trains at night" (newer, confidence 0.75)
- Current conversation module is `gym`
- Log output is inspectable

**Steps:**
1. Run `DecisionPolicy.filter()` with the 30 memories, module = `gym`, and the parsed SOUL.md rules
2. Inspect the Stage 1 output: assert only `gym` and `general` category memories remain; assert all `wardrobe` and `food` memories are excluded
3. Inspect the Stage 2 output: assert both `food` memories (confidence 0.40 and 0.48) are excluded; assert the contradiction pair is resolved to the newer item ("User trains at night"); assert the older item ("User loves early morning training") is excluded
4. Assert the SOUL.md-conflicting gym memory is removed after the override check
5. Inspect the Stage 3 output: assert exactly 20 items are passed to the prompt (the top 20 by adjusted relevance score from the remaining set)
6. Assert that a gym memory updated within the last 7 days has its relevance score boosted by +0.10 (visible in the filter's scoring log)
7. Inspect the `DecisionPolicyResult` log entry: assert it contains `{ input_count: 30, post_stage1_count, post_stage2_count, post_stage3_count, soul_overrides_count }`

**Expected Result:**
- Step 2: Only `gym` and `general` memories pass Stage 1; `wardrobe`, `food` excluded
- Step 3: Confidence 0.40 and 0.48 items excluded; "User trains at night" (newer) included; "User loves early morning training" (older) excluded from the passing set
- Step 4: SOUL.md-conflicting memory removed; count decremented before Stage 3
- Step 5: Exactly 20 memories in the final set; the lowest-ranked items from the 21+ set are discarded for this request (still in Mem0)
- Step 6: Recency bonus of +0.10 reflected in the score for the recent gym memory
- Step 7: Log entry contains all four count fields and `soul_overrides_count: 1`

**Failure Indicators:**
- Wardrobe or food memories in the Stage 1 passing set
- A low-confidence (< 0.50) memory in the Stage 2 passing set
- Older contradicting memory retained instead of the newer one
- SOUL.md-conflicting memory present in the final set
- More or fewer than 20 memories in the final output
- Recency bonus absent from the scoring log
- Log entry missing any required field

---

## Sub Flows

### TC-003-004-002: All memories filtered out by Stage 1 (wrong module) — pipeline proceeds with USER.md only
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-003-001

**Preconditions:**
- Test user has 10 `wardrobe` memories and 0 `gym` or `general` memories
- Current conversation module is `gym`

**Steps:**
1. Run `DecisionPolicy.filter()` with module = `gym`
2. Observe the output passed to the pipeline

**Expected Result:**
- Stage 1 output is zero items (no gym or general memories)
- Pipeline proceeds with USER.md core profile and zero memory items
- Response is generated normally; this is not treated as an error condition
- `DecisionPolicyResult` log shows `post_stage1_count: 0`

**Failure Indicators:**
- Wardrobe memories pass Stage 1 for a gym-module conversation
- Pipeline throws an error on zero memories
- Response generation is blocked

---

### TC-003-004-003: All memories below confidence threshold — zero passed, pipeline uses USER.md only
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-006
**Dependencies:** S-003-001

**Preconditions:**
- Test user has 30 memories, all with `confidence < 0.50` (e.g., all set to 0.45)
- Current conversation module is `general` (so all pass Stage 1)

**Steps:**
1. Run `DecisionPolicy.filter()` with all 30 memories
2. Observe Stage 2 output and what reaches the pipeline

**Expected Result:**
- Stage 2 output is zero items
- Pipeline proceeds with USER.md core profile only
- No error; response is generated normally
- `DecisionPolicyResult` log: `post_stage2_count: 0`

**Failure Indicators:**
- Any low-confidence memory passes Stage 2
- Pipeline errors on zero memories after Stage 2

---

### TC-003-004-004: Contradiction false positive — more recent memory retained (safer default)
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-003-001

**Preconditions:**
- Two gym memories that are semantically similar but not truly contradictory (e.g., "User prefers morning training" and "User sometimes trains in the morning") — the detector may flag these as contradicting
- One is older than the other

**Steps:**
1. Run `DecisionPolicy.filter()` with these two memories included
2. If the detector flags a contradiction, observe which item is retained

**Expected Result:**
- The more recently updated memory is retained (safer default)
- No user-facing impact
- The filter log notes the contradiction flag and resolution

**Failure Indicators:**
- The older memory is retained instead of the newer one
- Both memories are removed (over-aggressive removal)
- Filter throws an error on the flagged pair

---

### TC-003-004-005: Ambiguous module (general conversation) — all categories pass Stage 1
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-003-001

**Preconditions:**
- Test user has memories across all 6 categories (wardrobe, gym, food, spending, routines, general)
- Current conversation module cannot be determined — the pipeline passes `module: general` to the filter

**Steps:**
1. Run `DecisionPolicy.filter()` with `module: general`
2. Inspect Stage 1 output

**Expected Result:**
- All categories pass Stage 1 (no module-specific filtering applied)
- Stage 2 and Stage 3 run normally on the full set
- No module-specific bonus or penalty is applied to any item's score

**Failure Indicators:**
- Any category filtered out in Stage 1 for a `general` module conversation
- Module-specific scoring bonuses applied in error

---

### TC-003-004-006: Identical `updated_at` timestamps on conflicting memories — higher-confidence item retained
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case: tie-break)
**Dependencies:** S-003-001

**Preconditions:**
- Two gym memories with identical `updated_at` timestamps (same second), one with `confidence: 0.80` and one with `confidence: 0.75`, with contradicting content

**Steps:**
1. Run `DecisionPolicy.filter()` with these two memories
2. Inspect which item is retained after the contradiction resolution pass

**Expected Result:**
- The item with `confidence: 0.80` is retained
- The item with `confidence: 0.75` is excluded
- Tie-break is deterministic (same result every run)

**Failure Indicators:**
- Lower-confidence item retained
- Non-deterministic result (sometimes 0.80 wins, sometimes 0.75 wins)
- Both items removed

---

### TC-003-004-007: Performance — `filter()` completes in under 50ms for 30 inputs
**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR coverage — performance target)
**Dependencies:** S-003-001

**Preconditions:**
- 30 memories are prepared in memory (not requiring live Mem0 calls during the filter)
- Test environment is a representative server (not local dev laptop)

**Steps:**
1. Run `DecisionPolicy.filter()` 100 times with 30 inputs each run
2. Record execution time for each run
3. Calculate p95 execution time

**Expected Result:**
- p95 execution time is under 50ms
- If contradiction detection exceeds budget, a warning is logged and the step is skipped for that run (per technical requirement)

**Failure Indicators:**
- p95 execution time exceeds 50ms consistently
- No warning logged when contradiction detection is skipped due to budget overrun
- Filter errors instead of gracefully skipping the expensive step

---

## Automation Notes
- **Test fixture setup:** use a factory function to generate 30 memories with configurable categories, confidence, and `updated_at` values; this avoids relying on live Mem0 state for each test run
- **Filter output inspection:** add a test-mode output hook to `DecisionPolicy.filter()` that returns the intermediate counts (post_stage1, post_stage2, post_stage3) alongside the final set; do not rely solely on log parsing
- **Recency scoring assertion (AC-007):** set memory `updated_at` to "now minus 3 days" for the recent memory and assert the scoring log shows `adjustedScore = baseScore + 0.10`; set another to "now minus 100 days" and assert `adjustedScore = baseScore - 0.10`
- **SOUL.md override test:** maintain a test fixture SOUL.md with a single explicit, easily-matchable rule; create a memory that clearly violates it; run the filter and assert the memory is removed
- **Performance test (TC-003-004-007):** run in CI on a consistent hardware profile; exclude this test from developer-local runs where environment variability is high
- **Contradiction detection:** unit-test `SemanticContradictionDetector` separately with known contradicting pairs and known non-contradicting pairs before running integration tests that depend on it

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

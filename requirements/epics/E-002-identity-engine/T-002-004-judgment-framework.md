# QA Test Plan — Judgment Framework Application per Module

---

**Plan ID:** T-002-004
**Story:** S-002-004
**Epic:** E-002
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates that the correct judgment principles from SOUL.md are applied when generating responses across each life module. Testing covers: wardrobe fit-over-brand feedback, gym consistency-over-intensity advice, food protein-first assessment, spending compounding-vs-consumption framing, position maintenance under user pushback, out-of-scope requests routed to `general`, and multi-principle trade-off handling. Memory-informed adjustments are explicitly out of scope.

## Out of Scope

- Memory-informed judgment adjustments (requires E-003)
- Exposing judgment rules directly to the user ("I'm applying rule X")
- User feedback loop for refining the judgment framework (future scope)

## Prerequisites

- S-002-001 (SOUL.md system prompt loading) is complete
- S-002-003 (module routing) is complete — module classification must be correct before judgment can be tested
- `buildModuleContext(module)` function is implemented
- Test user account exists with chat access
- The last-5-message consistency check is implemented

---

## Core Test Flow

### TC-002-004-001: Apply correct judgment principle for each module, maintain position under pushback
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-002-001, S-002-003

**Preconditions:**
- Test user is authenticated
- Conversation history is clear (fresh session)

**Steps:**
1. Send an outfit photo of a clearly oversized hoodie — inspect the response for fit/silhouette-based feedback (e.g., references to proportion, silhouette, fit — not just "looks casual" or "it's comfy")
2. Send: "Should I do chest today even though I'm tired?" — inspect the response for the consistency-over-intensity principle (e.g., encouragement to train even lighter vs. skipping; not "listen to your body and rest")
3. Send a meal photo showing mostly carbs with little to no protein (e.g., a plate of pasta) — inspect the response for protein-first framing (e.g., suggests adding protein source, assesses the meal against protein requirements)
4. Send: "Should I buy these $200 sneakers?" — inspect the response for the compounding-vs-consumption framework (e.g., asks what value they provide, whether it serves a purpose vs. status consumption)
5. After step 1's response, send a pushback: "I like oversized, it's comfortable" — inspect the response to confirm Noxis maintains its recommendation with reasoning and does not reverse its position
6. Confirm across all responses that judgment rules are applied implicitly (reasoning reflects the principle) and not cited explicitly as "Rule 3 says..."

**Expected Result:**
- Step 1: Response references fit, silhouette, or proportion — not a generic style comment
- Step 2: Response applies consistency-over-intensity; does not simply validate skipping the workout
- Step 3: Response flags protein gap; applies protein-first rule
- Step 4: Response frames the purchase through a compounding vs. consumption lens; asks about utility or trajectory value
- Step 5: Noxis holds its position; acknowledges the user's preference but does not reverse the recommendation; reasoning is added, not removed
- Step 6: No response contains explicit rule citations like "According to my wardrobe rules..."

**Failure Indicators:**
- Wardrobe response is purely aesthetic ("looks casual") with no fit/silhouette reference
- Gym response tells the user to rest without applying consistency principle
- Food response does not mention protein deficit
- Spending response validates the purchase without a value-lens challenge
- Noxis reverses its wardrobe recommendation after one pushback
- Any response explicitly cites a rule number or says "according to my principles"

---

## Sub Flows

### TC-002-004-002: Out-of-SOUL.md-scope request — route to `general`, apply core philosophy
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-002-001, S-002-003

**Preconditions:**
- Test user is authenticated

**Steps:**
1. Send a request outside all module judgment scopes: "What's the best way to negotiate a car purchase?"
2. Inspect the module classification (should be `general`) and the response

**Expected Result:**
- Module classified as `general`
- Response applies Noxis's core philosophy (value-based realism, directness) where applicable
- No response pretends to have a SOUL.md rule for car negotiation; Noxis acknowledges the scope limit naturally in-character

**Failure Indicators:**
- Response fabricates a specific SOUL.md rule for the topic
- Module is incorrectly classified to a specific life module
- Response is a robotic refusal

---

### TC-002-004-003: Multiple applicable judgment principles — primary principle applied, trade-off acknowledged
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-002-001, S-002-003

**Preconditions:**
- Test user is authenticated

**Steps:**
1. Send: "I want to eat a big meal before an important client dinner tonight — good idea?" (touches food + social + spending + routines principles simultaneously)
2. Inspect the response

**Expected Result:**
- Response identifies and applies the most relevant primary judgment principle (e.g., food/eating-out-is-presentation or routines/performance)
- If a trade-off exists between principles, the response acknowledges it rather than ignoring the tension
- Response is not a list of all five principles applied sequentially — it reads naturally

**Failure Indicators:**
- Response applies all five principles in a mechanical list format
- Response ignores the trade-off and gives an unqualified answer
- Response applies a principle irrelevant to the question

---

### TC-002-004-004: Consistency check — no flip-flopping across last 5 messages
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — FR: consistency enforcement)
**Dependencies:** S-002-001, S-002-003

**Preconditions:**
- A conversation exists with 4 prior messages where Noxis clearly stated a judgment-based position (e.g., "tailored fit is better")
- The 5th message restates the topic in a new framing designed to elicit a contradicting answer

**Steps:**
1. Send the 5th message (e.g., "But wouldn't baggy be better for my body type?")
2. Inspect the response

**Expected Result:**
- Response maintains the previously stated position (tailored is better) with context-aware reasoning
- No contradiction with the prior 4 messages in the conversation
- Response does not read as if it has forgotten the prior discussion

**Failure Indicators:**
- Response contradicts the prior judgment position without a principled reason
- Response gives a hedge ("it depends") when a clear principle applies
- Consistency check does not appear to have run (prior messages not considered)

---

## Automation Notes
- **Judgment principle assertions:** build a per-module keyword/phrase checklist (e.g., wardrobe: ["fit", "silhouette", "proportion", "tailored"]; gym: ["consistency", "show up", "compounds"]); assert at least one keyword from the relevant module list is present in the response
- **Pushback test (TC-002-004-001 step 5):** run this as a multi-turn conversation fixture; assert that the position stated in turn 1 is not reversed in turn 2 using semantic similarity checks or keyword guards
- **Module context injection:** instrument `buildModuleContext(module)` in test mode to log which judgment section was appended; assert that the logged section matches the expected module
- **Explicit rule citation check:** assert that no response contains phrases like "According to rule", "My wardrobe principles state", or similar citation patterns
- **Flakiness risk:** LLM responses are non-deterministic; run each judgment test case 3 times and assert that the keyword/principle appears in at least 2 of 3 runs before flagging as a failure

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

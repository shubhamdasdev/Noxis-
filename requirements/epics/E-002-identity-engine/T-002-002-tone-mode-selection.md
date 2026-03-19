# QA Test Plan — Implement Tone Mode Selection & Switching

---

**Plan ID:** T-002-002
**Story:** S-002-002
**Epic:** E-002
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates that tone mode selection works correctly during onboarding and post-onboarding switching from the Profile tab. Testing covers: onboarding card UI and selection storage, correct tone injection into the system prompt for each of the three modes, persistence across sessions, Profile tab switching with immediate effect on next message, SOUL.md core identity preservation across all tones, graceful fallback when tone is unset or corrupted, and race condition handling on rapid switching.

## Out of Scope

- User-editable or custom tone instructions
- More than three tone modes
- Tone analytics (which mode correlates with engagement)
- Profile tab implementation beyond tone switching (S-007-005)

## Prerequisites

- S-002-001 (SOUL.md system prompt loading) is complete and passing
- S-007-005 (Profile tab) is complete for post-onboarding switching tests
- `tones/brother.md`, `tones/consultant.md`, `tones/peer.md` files are committed to the repo
- `buildSystemPrompt(toneMode)` is implemented
- Onboarding flow is testable end-to-end

---

## Core Test Flow

### TC-002-002-001: Select each tone during onboarding, verify injection, switch in Profile, confirm core identity unchanged
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006
**Dependencies:** S-002-001

**Preconditions:**
- Fresh test user account (no prior tone selection)
- Onboarding flow is accessible

**Steps:**
1. Open the Tone Selection screen during onboarding; confirm three glass cards are displayed — one each for "brother", "consultant", and "peer" — each with a mode name, one-line description, and a sample response
2. Tap the "brother" card; confirm it highlights with a glass glow border; confirm the other two cards are not highlighted
3. Complete onboarding; send the first message ("Rate my morning routine")
4. Intercept the AI provider request payload; confirm `## Active Tone Mode: brother` and the Sharp Older Brother tone instructions are appended to the system prompt below SOUL.md content
5. Inspect the response for Sharp Older Brother register markers: direct, slightly warm, occasional push — confirm the response uses this register
6. Confirm SOUL.md core identity is unchanged: opinions, judgment principles, and boundaries are consistent with baseline
7. Navigate to Profile tab → Tone Mode setting; confirm the same three-card selector is present with "brother" pre-selected
8. Tap "consultant"; return to chat; send a follow-up message ("What should I focus on this week?")
9. Intercept the payload; confirm `## Active Tone Mode: consultant` is now in the system prompt; confirm tone instructions switched
10. Inspect the response for High-End Consultant register: precise, structured, no wasted words
11. Confirm the previous "brother" conversation history is not retroactively re-toned
12. Navigate back to Profile → Tone Mode; tap "peer"; send another message ("Am I on track?")
13. Confirm the response uses Confident Peer register: conversational, equal-to-equal

**Expected Result:**
- Step 1: Three cards visible with name, description, and sample copy
- Step 2: Tapped card shows glow border; others do not
- Step 4: Payload contains `## Active Tone Mode: brother` plus brother-specific instructions below SOUL.md
- Step 5: Response uses Sharp Older Brother register
- Step 6: Judgment positions and boundaries identical to SOUL.md baseline regardless of tone
- Step 8: Tone stored to user profile before onboarding proceeds
- Step 9: Payload reflects `consultant` tone; no trace of `brother` instructions
- Step 10: Response is precise and structured
- Step 11: Prior messages display as originally generated (not re-toned)
- Step 13: Response is conversational, peer-level

**Failure Indicators:**
- Cards missing sample copy or glow highlight on selection
- Tone mode not persisted between onboarding completion and first message
- Wrong tone instructions injected for the selected mode
- SOUL.md core opinions reversed or softened under any tone
- Prior conversation history changes tone retroactively

---

## Sub Flows

### TC-002-002-002: No tone selected (edge case) — system defaults to "peer"
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-006
**Dependencies:** S-002-001

**Preconditions:**
- User account exists with no tone value in the USER.md / user profile record (simulate by manually clearing the `tone` field in the database)

**Steps:**
1. Send a message with no tone mode set in the user profile
2. Intercept the AI provider request payload
3. Confirm `## Active Tone Mode: peer` is present in the system prompt
4. Confirm the response uses Confident Peer register

**Expected Result:**
- System defaults silently to "peer"; no error is shown to the user
- Payload contains peer tone instructions

**Failure Indicators:**
- Error surfaced to user about missing tone
- No tone instructions in the payload (bare SOUL.md only)
- System defaults to a mode other than "peer"

---

### TC-002-002-003: Corrupted tone value in storage — silent default to "peer"
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-002-001

**Preconditions:**
- User profile `tone` field is set to an invalid value (e.g., "aggressive" or a numeric string) directly in the database

**Steps:**
1. Send a message
2. Observe user-facing response and inspect the payload

**Expected Result:**
- No error is shown to the user
- System silently falls back to "peer" tone instructions in the payload
- Response uses Confident Peer register

**Failure Indicators:**
- Error message visible to the user referencing tone mode
- App crashes or returns a 500 error
- Payload contains no tone instructions or an invalid tone block

---

### TC-002-002-004: Rapid tone switching — last selection wins, no race condition
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-002-001

**Preconditions:**
- User is on the Profile tab tone selector
- Network latency is simulated (optional, to exaggerate race conditions)

**Steps:**
1. Rapidly tap between "brother" → "consultant" → "peer" → "brother" in quick succession (within ~500ms)
2. Send a message immediately after the rapid taps
3. Inspect the payload and user profile record

**Expected Result:**
- The tone in the payload matches the last card tapped ("brother" in this sequence)
- User profile stores only the last selection — no intermediate values or duplicates
- No race condition causes an incorrect tone to be injected

**Failure Indicators:**
- Tone in the payload does not match the final tap
- Multiple tone blocks are injected into the system prompt
- User profile stores a stale intermediate selection

---

### TC-002-002-005: Tone change mid-conversation — next response reflects new tone, prior messages unchanged
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-002-001

**Preconditions:**
- User has an active conversation with "brother" tone (3+ messages exchanged)

**Steps:**
1. Switch tone to "consultant" in Profile tab
2. Return to the active conversation
3. Send a new message in the same thread
4. Inspect the response and the prior messages in the conversation

**Expected Result:**
- The new message response uses Consultant register
- Prior messages in the thread remain as originally generated — no retroactive re-toning
- No loading state or delay beyond the normal AI response time

**Failure Indicators:**
- Prior messages visually change tone or content
- New response still uses "brother" tone
- Any error or loading failure on tone switch

---

## Automation Notes
- **Tone register assertion:** use keyword presence checks for mode-specific markers rather than exact string matching (e.g., "consultant" responses should contain structured formatting or numbered lists; "brother" responses should contain direct address language); maintain a per-mode positive and negative keyword list
- **Payload inspection:** instrument `buildSystemPrompt(toneMode)` in test mode to capture the assembled string before it is sent to the AI provider
- **Race condition test:** run TC-002-002-004 in a loop (50 iterations) to confirm deterministic behavior; log all profile writes during the test
- **Selector strategy:** target tone card selection by `data-testid="tone-card-{mode}"` attributes; confirm glow class application in DOM
- **Flakiness risk:** AI response content is non-deterministic; assert structural/register markers rather than exact phrasing

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# QA Test Plan — Conversational Onboarding in Chat

---

**Plan ID:** T-001-003
**Story:** S-001-003
**Epic:** E-001
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the conversational onboarding flow that runs inside the Chat interface after Quick Profile. Testing covers automatic first-message delivery (within 2 seconds, no user prompt), tone-appropriate opening message content that references Quick Profile selections, wardrobe classification when a style photo is sent during onboarding, the "let's go" / "done" / "skip" completion signals that exit onboarding early, and the graceful handling of off-topic questions and minimal one-word engagement. Both the negative scenario (user immediately goes off-topic) and the edge case (only one-word answers given) are covered.

## Out of Scope

- Scripted chatbot / hardcoded response flow — responses must go through the full Noxis pipeline (Do Not Do)
- Blocking the user from using the app if onboarding is skipped entirely (Do Not Do)
- Richer onboarding with wardrobe gallery import (Future Scope)
- Style quiz with visual reference picks (Future Scope)

## Prerequisites

- S-001-001 (Tone selection completed — tone stored in `OnboardingState`)
- S-001-002 (Quick Profile data available in `OnboardingState`)
- S-002-001 (SOUL.md loaded by the Noxis engine)
- S-002-002 (Tone mode applied to message pipeline)
- S-004-001 (Chat interface operational — message send/receive cycle working)
- `OnboardingManager` is initialized with `isOnboarding = true`

---

## Core Test Flow

### TC-001-003-001: Full conversational onboarding — opening message, Q&A, completion signal, first value trigger
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-001-001, S-001-002, S-002-001, S-002-002, S-004-001

**Preconditions:**
- User has completed Quick Profile and has been navigated to the Chat screen
- `isOnboarding = true` in `OnboardingManager`
- Tone is set to **Brother**, Quick Profile selection was **Style** as primary focus

**Steps:**
1. Observe the Chat screen immediately on load — confirm Noxis sends the first onboarding message automatically within 2 seconds, without the user having typed anything
2. Confirm the message uses the Brother tone register: direct, slightly teasing, references Style as focus — matches the pattern "Alright, I've got the basics. Now show me what I'm working with..."
3. Confirm the message does not use Consultant or Peer register (not structured, not peer-casual)
4. Send a style photo using the camera or photo library button
5. Confirm Noxis responds with: (a) wardrobe classification of the photo, (b) specific feedback on the outfit, and (c) the photo is stored (wardrobe module receives first item)
6. Send two text answers to subsequent onboarding questions (goal and context questions)
7. After 3 answers total (photo + 2 text), send "let's go"
8. Confirm Noxis does not ask any remaining onboarding questions and instead triggers first value delivery (S-001-004 message fires in chat)
9. Confirm `OnboardingManager.isOnboarding` is set to `false` after the completion signal

**Expected Result:**
- Noxis opening message appears in Chat within 2 seconds of screen mount, with no user prompt
- Message uses Brother tone register and references the Style focus from Quick Profile (not a generic opening)
- Style photo sent during onboarding is classified as wardrobe, receives feedback, and is stored to the wardrobe module
- After 3 exchanges and the "let's go" signal, remaining questions are skipped and first value delivery fires
- `isOnboarding` is set to `false` and persisted after completion

**Failure Indicators:**
- Noxis opening message does not appear within 2 seconds of Chat screen mount
- Opening message uses wrong tone (e.g. Consultant-structured language in Brother mode)
- Opening message is generic and does not reference Style or any Quick Profile context
- Photo is not classified, no feedback is given, or photo is not stored to wardrobe
- Noxis ignores "let's go" and continues asking onboarding questions
- `isOnboarding` remains `true` after completion signal

### TC-001-003-002: Consultant tone — opening message uses correct register
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-001-001, S-002-001, S-002-002, S-004-001

**Preconditions:**
- Tone is set to **Consultant**, Quick Profile selection was **Fitness** as primary focus

**Steps:**
1. Navigate to Chat after Quick Profile
2. Observe Noxis's automatic opening message
3. Confirm the message uses the precise, structured Consultant register — matches pattern: "Setup noted. Primary focus: fitness. To give you useful recommendations from day one, I need your current program..."
4. Confirm the message does not use casual Brother language or informal Peer phrasing

**Expected Result:**
- Opening message is structured, precise, and references fitness as the stated focus
- Language matches the Consultant register distinctly — not generic and not another tone

**Failure Indicators:**
- Opening message uses casual or teasing language (wrong tone)
- Opening message is identical regardless of tone selection (tone mode not applied)

### TC-001-003-003: Peer tone, no Quick Profile selections — opening message uses blank-slate variant
**Type:** Happy Path
**Priority:** P2
**AC Covered:** AC-001, AC-002
**Dependencies:** S-001-001, S-002-001, S-002-002, S-004-001

**Preconditions:**
- Tone is set to **Peer**, Quick Profile was skipped (no selections stored)

**Steps:**
1. Navigate to Chat after skipping Quick Profile
2. Observe Noxis's automatic opening message
3. Confirm the message acknowledges the blank slate and uses Peer register — matches pattern: "Cool — blank slate. I work best when I know a bit about you. Drop a photo of your last outfit..."

**Expected Result:**
- Opening message uses Peer register (casual, friendly, not structured)
- Message does not reference any specific focus area (since none was selected)

**Failure Indicators:**
- Message references a focus area (Style, Fitness, etc.) that was never selected
- Message uses Consultant or Brother register

### TC-001-003-004: User immediately asks off-topic question — Noxis answers and redirects once
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-005
**Dependencies:** S-001-001, S-002-001, S-002-002, S-004-001

**Preconditions:**
- Onboarding is active (`isOnboarding = true`); Noxis has sent the opening message

**Steps:**
1. Instead of responding to the opening onboarding prompt, send a completely unrelated question: "What's a good recipe for steak?"
2. Observe Noxis's response

**Expected Result:**
- Noxis answers the steak question substantively
- In the same response or the next message, Noxis gently redirects once — saying something like "Before we go further — drop me one thing: [the most important missing context question]"
- Noxis does not refuse to answer the off-topic question or block the user
- Onboarding context is gathered opportunistically — it does not block normal chat usage

**Failure Indicators:**
- Noxis refuses to answer the off-topic question and only shows an onboarding prompt
- Noxis does not redirect back to onboarding at all after answering
- Noxis redirects more than once in the same conversation (the redirect is one-time-only)

### TC-001-003-005: User sends only one-word answers — onboarding completes after 3 exchanges
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-004
**Dependencies:** S-001-001, S-002-001, S-002-002, S-004-001

**Preconditions:**
- Onboarding is active; Noxis has sent the opening message

**Steps:**
1. Reply to Noxis's opening message with "okay"
2. Reply to the next prompt with "fine"
3. Reply to the next prompt with "sure"
4. Observe whether Noxis moves to first value delivery after 3 exchanges without forcing more questions

**Expected Result:**
- After 3 exchanges (regardless of answer quality), Noxis accepts what it has and moves to first value delivery
- Noxis does not force the user to provide more detailed answers
- `isOnboarding` is set to `false` after the 3rd exchange triggers completion

**Failure Indicators:**
- Noxis continues asking questions beyond 3 exchanges when one-word answers are given
- Noxis explicitly requests more detail and refuses to proceed with minimal answers
- `isOnboarding` is not set to `false` after 3 exchanges

---

## Automation Notes

- **Auto-send timing:** Assert that `OnboardingManager` calls the message pipeline within 2000ms of the Chat view's `onAppear` — use an `XCTestExpectation` with `fulfillmentInterval: 2.0`
- **Tone verification:** Tone-appropriate language cannot be reliably asserted via XCUITest — use unit tests on the prompt construction logic in `OnboardingManager` to verify the correct SOUL.md context + tone mode is passed for each tone variant
- **Photo wardrobe storage:** Use a mock wardrobe module in tests to assert that a photo sent during `isOnboarding == true` results in a call to `WardrobeModule.store(image:)` with the captured image
- **Completion signal:** Unit-test `OnboardingManager.processMessage(_:)` with "let's go", "done", and "skip" inputs to confirm `isOnboarding` transitions to `false` and the first value delivery trigger fires
- **Flakiness risk:** LLM-generated response content is non-deterministic — do not assert exact message text in automated tests; assert structural properties (message arrives within timeout, sender is Noxis, response is non-empty, tone mode was applied to the request context)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

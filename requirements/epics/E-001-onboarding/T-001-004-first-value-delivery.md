# QA Test Plan — First Value Delivery

---

**Plan ID:** T-001-004
**Story:** S-001-004
**Epic:** E-001
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the proactive first value delivery message that Noxis sends automatically at the end of conversational onboarding. Testing covers automatic delivery without a user prompt, specificity of the recommendation relative to the user's stated focus and onboarding context, tone-mode correctness, transition to normal chat mode after delivery, persistence of the first value message in conversation history on next session open, the negative scenario of empty onboarding context (generic fallback message), and the edge case of API failure with retry logic.

## Out of Scope

- Hardcoded template generation — first value must go through the full Noxis pipeline (Do Not Do)
- Onboarding completion screen or animation (Do Not Do)
- Personalisation improvements in future onboarding iterations (Future Scope)

## Prerequisites

- S-001-003 (Conversational onboarding has collected context and signalled completion)
- S-002-001 (SOUL.md loaded)
- S-002-002 (Tone mode applied)
- S-002-003 (Module routing implemented)
- `OnboardingManager` completion signal fires correctly (per S-001-003)

---

## Core Test Flow

### TC-001-004-001: First value delivery — proactive, specific, tone-correct, transitions to normal chat
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005
**Dependencies:** S-001-003, S-002-001, S-002-002, S-002-003

**Preconditions:**
- User has completed conversational onboarding with the following context: primary focus = **Style**, tone = **Brother**, style description provided as "streetwear, comfortable, mostly black"
- `OnboardingManager` has just signalled completion (`isOnboarding` transitioning from `true` to `false`)
- The user has not typed anything after their last onboarding answer

**Steps:**
1. Observe the Chat screen immediately after the completion signal fires — confirm Noxis sends a first value message without the user typing anything
2. Read the first value message — confirm it: (a) is specific to the Style focus area, (b) references at least one piece of context shared during onboarding (streetwear, comfortable, black), and (c) includes one concrete action the user can take today or this week (e.g. "Buy a fitted black overshirt this week" — not "dress better" or "improve your style")
3. Confirm the message does not contain generic advice ("eat healthier", "work out more", or any variant not tied to the stated context)
4. Confirm the message uses the Brother tone register (direct, slightly provocative, action-first)
5. Confirm Noxis follows up with the prompt: "What's your reaction? And feel free to ask me anything — that's what I'm here for."
6. Send a reply to Noxis (e.g. "That makes sense")
7. Confirm Noxis responds as in normal chat mode — no onboarding prompts, no "before we continue" redirects
8. Confirm `isOnboarding` is `false` and the `onboardingComplete` `UserDefaults` flag is `true`
9. Force-quit the app and relaunch
10. Open Chat — confirm the first value message is present in conversation history and onboarding does NOT restart

**Expected Result:**
- First value message appears in Chat without any user prompt, immediately after the completion signal
- Message is specific to Style focus + streetwear/comfortable/black context; contains one actionable recommendation for this week
- No generic advice present in the message
- Message uses Brother register (not Consultant or Peer)
- Follow-up "What's your reaction?" prompt appears after the first value message
- After the user replies, chat is in normal mode — no onboarding machinery is running
- `onboardingComplete` flag is persisted; on next app open, chat history is present and onboarding does not restart

**Failure Indicators:**
- First value message requires the user to send a prompt to trigger it
- Message contains generic advice not tied to the user's specific stated context or focus
- Message uses wrong tone register (Consultant structure or Peer casualness in Brother mode)
- Follow-up "What's your reaction?" message does not appear
- After replying, Noxis still shows onboarding redirects or context-gathering prompts
- Restarting the app causes onboarding to restart or the first value message is absent from history

### TC-001-004-002: Consultant tone, fitness focus — first value message uses precise structured register
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-002, AC-003
**Dependencies:** S-001-003, S-002-001, S-002-002

**Preconditions:**
- Onboarding context: focus = **Fitness**, tone = **Consultant**, user trains 3x/week

**Steps:**
1. Wait for the first value message to fire after completion signal
2. Read the message — confirm it: (a) references the fitness focus and 3x/week frequency, (b) gives a specific structural recommendation (e.g. "Add a fourth training day focused on weak points"), (c) does not contain vague advice ("train harder", "get fit")
3. Confirm the language is precise, structured, and analytical — matching Consultant register

**Expected Result:**
- Message references 3x/week training frequency and recommends a specific program change
- Language is structured and data-driven, not casual or teasing
- Recommendation is a concrete action (specific day, specific focus area) — not a general principle

**Failure Indicators:**
- Message is generic fitness advice with no reference to the 3x/week stated context
- Message uses casual Brother-style language or informal Peer phrasing

### TC-001-004-003: Empty onboarding context (user skipped everything) — generic fallback message
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-001
**Dependencies:** S-001-003, S-002-001, S-002-002

**Preconditions:**
- User skipped Quick Profile and sent only "skip" during conversational onboarding
- `OnboardingState` contains no selections for primaryFocus, fitnessLevel, or cookingLevel

**Steps:**
1. Trigger the completion signal (user sent "skip" after Noxis's opening message)
2. Observe the first value message that fires

**Expected Result:**
- First value message fires automatically (no user prompt)
- Message is the defined fallback: "Tell me one thing about your current situation — style, fitness, food, spending, or routines — and I'll give you something specific to work with today."
- Message is not a generic lifestyle tip or blank/empty message
- Follow-up "What's your reaction?" prompt is not sent (the fallback itself is an open question)

**Failure Indicators:**
- No message is sent when context is empty
- A generic lifestyle tip appears instead of the defined fallback copy
- The fallback message is sent multiple times or triggers a loop

### TC-001-004-004: API call for first value delivery fails — retry once, then show error with Try Again
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-001
**Dependencies:** S-001-003, S-002-001, S-002-002

**Preconditions:**
- Onboarding completion is triggered
- The message API is mocked to fail on the first call and succeed on the second (first retry succeeds)

**Steps:**
1. Trigger the completion signal
2. Mock API returns an error on the first call — confirm the app silently retries without showing an error to the user
3. Mock API returns success on the retry — confirm the first value message appears normally

**Steps (failure branch):**
4. Reset: mock API is configured to fail on both the first call and the retry
5. Trigger the completion signal again
6. Confirm a "Something went wrong" message appears in Chat with a "Try again" tap target
7. Tap "Try again" — confirm the API call is retried and (if successful) the first value message appears

**Expected Result:**
- On first failure, the app silently retries — no error appears to the user
- On second failure (retry also fails), "Something went wrong" appears with a "Try again" tap target
- Tapping "Try again" triggers another attempt at the first value delivery
- No crash or hang occurs in either failure scenario

**Failure Indicators:**
- Error is shown to the user on the first attempt (no silent retry)
- After two failures, no "Something went wrong" / "Try again" appears
- Tapping "Try again" does not re-trigger the delivery attempt
- App crashes or hangs when the API fails during first value delivery

---

## Automation Notes

- **Auto-delivery timing:** Use `XCTestExpectation` to assert Noxis's message appears in the chat message list within 5 seconds of `OnboardingManager.complete()` being called (allow additional time over the 2s opening message due to LLM pipeline latency)
- **Content specificity:** Cannot reliably assert exact LLM output — instead, unit-test `FirstValueDelivery.buildPrompt(onboardingContext:)` to confirm the correct context fields (focus, tone, onboarding answers) are embedded in the prompt sent to the pipeline; assert the built prompt is not identical to the generic fallback prompt when context is present
- **`onboardingComplete` flag:** Assert `UserDefaults.standard.bool(forKey: "onboardingComplete") == true` after delivery; assert the flag persists across simulated app restarts
- **Retry logic:** Unit-test the retry in `FirstValueDelivery.swift` by injecting a mock that fails N times; assert the retry fires exactly once before showing the error message
- **Flakiness risk:** LLM response time is variable — do not use tight timing assertions on the delivery; use generous timeouts (10s) and assert message existence, not delivery speed

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# Story: First Value Delivery

---

**ID:** S-001-004
**Epic:** E-001
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to give me one specific, actionable recommendation immediately after onboarding**, so that **I understand what this app actually does before I have to ask it anything**.

## Goal

At the completion of conversational onboarding, Noxis proactively delivers one concrete recommendation tailored to what it just learned — without the user having to prompt it. This is the "aha moment" that proves the product works.

## Why / Rationale

Most apps make users do work before they get value. Noxis flips this: the first thing that happens after setup is Noxis delivering value unprompted. This is the moment that converts a curious installer into a retained user. The recommendation must be specific — generic advice here kills the impression.

## Functional Requirements

**Trigger:**
- `OnboardingManager` signals completion → first value delivery fires as a Noxis message in chat
- No user prompt required — Noxis sends it proactively

**Message content rules:**
- Must be specific to the user's stated focus area and at least one piece of context shared during onboarding
- Must include one concrete action the user can take today or this week
- Must not be generic ("eat healthier", "work out more") — must name a specific thing
- Delivered in the selected tone mode

**Examples by focus area:**

*Style focus, brother tone, user described style as "streetwear, comfortable, mostly black":*
> "Based on what you've told me — here's your first move: get one tailored piece. You don't have to abandon streetwear. But right now everything you're wearing is saying 'comfortable.' One fitted black overshirt changes the whole equation. Buy it this week."

*Fitness focus, consultant tone, user trains 3x/week:*
> "First recommendation: add a fourth training day, focused on weak points. You're at 3 sessions — the jump from 3 to 4 is where visible change accelerates. Review your current split and identify the most underdeveloped area. That's your fourth day."

*Routine focus, peer tone, no cooking:*
> "Here's where I'd start: cook one meal this week. Doesn't have to be impressive — chicken, rice, a vegetable. The habit of cooking once a week is the foundation everything else builds on. If you're ordering every meal, you're spending money and eating worse than you think."

**Follow-up prompt:**
- After the first value message, Noxis adds a short follow-up: "What's your reaction? And feel free to ask me anything — that's what I'm here for."
- This opens the floor to real conversation

**Onboarding complete flag:**
- After first value delivery, `isOnboarding` is set to `false`
- User is now in normal chat mode
- Onboarding state is not shown again on subsequent sessions

## Prerequisites

- S-001-003 (Conversational onboarding has collected context)
- S-002-001 (SOUL.md loaded)
- S-002-002 (Tone applied)
- S-002-003 (Module routing)

## Acceptance Criteria

- [ ] Given onboarding is complete, when the completion signal fires, then Noxis sends a first value message automatically without the user typing anything
- [ ] Given the user stated "style" as their focus, when the first value message is generated, then it contains a specific style action — not generic advice
- [ ] Given the user selected consultant tone, when the first value message is generated, then it uses the precise, structured consultant register
- [ ] Given the first value message is sent, when the user responds, then normal chat mode is active — no more onboarding flow
- [ ] Given the user re-opens the app the next day, when the chat loads, then the first value message is in the conversation history but onboarding does not restart

## Negative Scenarios

- Onboarding context is empty (user skipped everything) → first value delivery falls back to a general first move: "Tell me one thing about your current situation — style, fitness, food, spending, or routines — and I'll give you something specific to work with today."

## Edge Cases

- API call for first value delivery fails → retry once silently; if second attempt fails, show a standard "Something went wrong" message with a "Try again" tap

## Dependencies

- S-001-003
- S-002-001
- S-002-002
- S-002-003

## Technical Requirements

- **`FirstValueDelivery.swift`** — builds a first value prompt using onboarding context + SOUL.md + tone; sends through the standard message pipeline
- **Onboarding state persistence** — `UserDefaults` flag `onboardingComplete: Bool`; once set to `true`, never shown again

## Future Scope

- Personalised first value delivery that improves as onboarding data quality improves

## Do Not Do

- In this story, do not generate the first value message from a hardcoded template — it must go through the full Noxis pipeline so it uses SOUL.md + tone
- Do not show an onboarding completion screen or animation — just deliver the message naturally in chat

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

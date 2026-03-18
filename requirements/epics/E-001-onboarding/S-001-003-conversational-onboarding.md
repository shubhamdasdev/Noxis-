# Story: Conversational Onboarding in Chat

---

**ID:** S-001-003
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

As a **guest**, I want **Noxis to complete my setup through a natural conversation**, so that **by the time I'm done, the app feels like it already knows me — not like I just filled out a form**.

## Goal

After the visual onboarding screens, the user lands in the Chat interface where Noxis — in the selected tone mode — asks 3-5 targeted questions to complete the baseline USER.md profile. The conversation is Noxis-led, not a form. Answers are stored as memory. The flow ends with first value delivery (S-001-004).

## Why / Rationale

Dropping into chat for the final onboarding steps proves the core product immediately. The user sees how Noxis talks, asks questions, and builds context — before they've even asked anything themselves. It's the product experience, not a setup wizard.

## Functional Requirements

**Entry:**
- User arrives at the Chat screen after Quick Profile
- Noxis sends the opening onboarding message automatically (no user prompt needed)
- Opening message uses the selected tone mode and references Quick Profile selections if any were made

**Opening message examples:**

*Brother mode (user selected Style as focus):*
> "Alright, I've got the basics. Now show me what I'm working with. Drop a photo of your last outfit — or give me 3 words that describe your current style."

*Consultant mode (user selected Fitness as focus):*
> "Setup noted. Primary focus: fitness. To give you useful recommendations from day one, I need your current program and training frequency. What does your week look like?"

*Peer mode (no selections):*
> "Cool — blank slate. I work best when I know a bit about you. Drop a photo of your last outfit, or just tell me: what are you most trying to improve right now?"

**Onboarding question flow (3-5 questions, Noxis-led):**

1. **Style baseline:** photo or 3-word description of current style
2. **Goal:** "What's the one area you most want to level up right now?" (if not answered in Quick Profile)
3. **Context:** "Anything I should know upfront? Job, city, relationship status — whatever you're comfortable with." (open-ended, user can share as much or little as they want)
4. **Fitness depth** (only if fitness was selected as focus): "What's your current split and how often do you actually show up?"
5. **Food depth** (only if not already known): "Do you cook or mostly order? Any dietary restrictions I should know about?"

**Completion signal:**
- After 3+ answers received OR user sends "done", "skip", or "let's go" → trigger first value delivery (S-001-004)
- Noxis does not force all questions — if the user answers 2 questions and says "let's go", it respects that

**Memory extraction:**
- All answers are extracted and stored to USER.md immediately after each response
- Photo sent during onboarding → wardrobe module receives first item

## Prerequisites

- S-001-001 (Tone selected)
- S-001-002 (Quick profile data available)
- S-002-001 (SOUL.md loaded)
- S-002-002 (Tone mode applied)
- S-004-001 (Chat interface operational)

## Acceptance Criteria

- [ ] Given the user arrives at Chat after Quick Profile, when the screen loads, then Noxis sends the first onboarding message automatically within 2 seconds
- [ ] Given the user is in Brother tone mode, when the opening message renders, then it uses the direct, slightly teasing register — not consultant or peer
- [ ] Given the user sends a style photo during onboarding, when received, then Noxis classifies it as wardrobe, gives feedback, and stores it
- [ ] Given the user has answered 3 questions, when they send "let's go", then Noxis skips remaining questions and triggers first value delivery
- [ ] Given the user ignores a question and asks something unrelated, when Noxis responds, then it answers the question and gently redirects back to onboarding context once

## Negative Scenarios

- User immediately asks an unrelated question (bypasses onboarding entirely) → Noxis answers it, then says "Before we go further — drop me one thing: [most important missing context question]." Onboarding context is gathered opportunistically, not enforced.

## Edge Cases

- User sends only one-word answers (minimal engagement) → Noxis accepts what it gets and moves to first value delivery after 3 exchanges regardless

## Dependencies

- S-001-001
- S-001-002
- S-002-001
- S-002-002
- S-004-001

## Technical Requirements

- **Onboarding state machine** in `OnboardingManager`: tracks which questions have been answered; signals completion to `FirstValueDelivery` trigger
- **Auto-send trigger:** on Chat screen mount with `isOnboarding == true`, fire the opening message via the standard message pipeline
- **Graceful exit:** any time `isOnboarding == true` and completion signal fires, `OnboardingManager` marks onboarding complete and persists the flag

## Future Scope

- Richer onboarding with wardrobe photo gallery import (v2)
- Style quiz with visual reference picks (v2)

## Do Not Do

- In this story, do not implement a scripted chatbot flow — onboarding messages go through the full Noxis pipeline (SOUL.md + tone + context), not a hardcoded script
- Do not block the user from using the app if they skip onboarding entirely

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

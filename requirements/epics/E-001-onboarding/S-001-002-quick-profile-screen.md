# Story: Quick Profile Screen

---

**ID:** S-001-002
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

As a **guest**, I want **to answer a few quick visual questions about my focus area and current habits**, so that **Noxis starts the conversation with useful context instead of asking me everything from scratch**.

## Goal

A single glass screen with 3 quick-tap multi-choice questions seeds USER.md before the conversational onboarding begins. The screen is skippable. It takes under 30 seconds to complete. No text input required.

## Why / Rationale

Seeding basic context (primary focus, fitness frequency, cooking habit) before the chat starts means Noxis's first conversational message is specific — not generic. This makes the "first value delivery" feel immediate and earns user trust before they've typed a single word.

## Functional Requirements

**Screen layout:**
- Header: "Quick setup" (Headline)
- Subtext: "Takes 20 seconds. Skip anytime." (Caption, muted)
- Three question blocks, each with visual pill selectors:

**Q1 — Primary focus:**
- "What do you most want to level up?"
- Options (pill selectors, multi-select up to 1): Style | Fitness | Daily Routine | All of the above

**Q2 — Current fitness:**
- "Do you train?"
- Options: Don't train | 1-2x/week | 3-4x/week | 5+ days/week

**Q3 — Cooking:**
- "Do you cook?"
- Options: Never — I order everything | Sometimes | Regularly

**Buttons:**
- "Continue" (active always — selections are optional per question)
- "Skip" (text link, muted) → same destination as Continue

**Selection behaviour:**
- Tapping a pill selects it (accent border); tapping again deselects
- Only one selection per question
- Selections stored to `OnboardingState`

## Prerequisites

- S-001-001 (Tone selection must be completed first)
- S-007-003 (Design system)

## Acceptance Criteria

- [ ] Given the user arrives from the Tone Selection screen, when the Quick Profile renders, then all three questions are visible with unselected pill options
- [ ] Given the user taps a pill option, when tapped, then it highlights with an accent border; tapping again deselects it
- [ ] Given the user taps Continue with some selections made, when navigated, then the selections are stored and the conversational onboarding screen opens
- [ ] Given the user taps Skip, when navigated, then they proceed to conversational onboarding with no selections stored
- [ ] Given the user completes selections and Continue is tapped, when stored, then the data is available to the conversational onboarding flow

## Negative Scenarios

- User answers no questions and taps Continue → valid; proceed to conversational onboarding with no pre-filled data

## Edge Cases

- User goes back to this screen from conversational onboarding → prior selections are still highlighted

## Dependencies

- S-001-001
- S-007-003

## Technical Requirements

- **`QuickProfileView.swift`** — SwiftUI view with pill selector components from design system
- Selections stored in `OnboardingState` as `{ primaryFocus, fitnessLevel, cookingLevel }` enums
- No API call at this step — data is held in memory until auth is complete

## Future Scope

- More questions in a v2 extended onboarding flow (style aesthetic, relationship status, city)

## Do Not Do

- In this story, do not use text input fields — visual pill selectors only
- Do not make any question required — all are optional

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

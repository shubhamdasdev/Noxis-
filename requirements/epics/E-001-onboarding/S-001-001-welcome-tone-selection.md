# Story: Welcome & Tone Selection Screens

---

**ID:** S-001-001
**Epic:** E-001
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **guest**, I want **to see a premium welcome screen and choose how Noxis talks to me before anything else**, so that **the app immediately feels personal and sets the right expectations from the first second**.

## Goal

Two polished glass screens: (1) a welcome screen with the Noxis wordmark and a single CTA, and (2) a tone selection screen with three glass cards showing each mode's name, description, and a sample exchange. The user taps a card to choose their tone and proceeds to onboarding.

## Why / Rationale

The first two screens are the product's first impression. They must communicate premium quality, set the right tone (literally), and get the user moving without friction. Tone selection is first because it shapes the entire onboarding conversation that follows.

## Functional Requirements

**Welcome screen:**
- Pure black background
- Noxis wordmark centered (SF Pro Display Bold, 28pt, white)
- Tagline below: "Your standards, elevated." (SF Pro Text Regular, 16pt, secondary white)
- Single CTA button: "Get Started" (glass button, accent border)
- No sign-in option yet — sign-in is after tone selection

**Tone Selection screen:**
- Header: "How should I talk to you?" (Headline style)
- Three glass cards, stacked vertically, full-width with 16pt horizontal padding
- Each card contains:
  - Mode name (Headline, white)
  - One-line description (Body, secondary)
  - Sample exchange: a muted "User:" prompt and a "Noxis:" response in that tone (Caption, showing the voice difference)
- Card examples:
  - Brother: User: "Rate my outfit" / Noxis: "That hoodie is doing nothing for you. Swap it for the bomber — you have 3 minutes."
  - Consultant: User: "Rate my outfit" / Noxis: "The hoodie adds bulk without structure. Your bomber would shift this to casual-sharp."
  - Peer: User: "Rate my outfit" / Noxis: "Honest take — the bomber you wore last week hit way harder. Switch it in."
- Tapping a card selects it (accent glow border); tapping again or tapping another card switches selection
- "Continue" button (disabled until a selection is made) → Quick Profile screen (S-001-002)
- Footnote below: "You can change this anytime in settings."

## Prerequisites

- S-007-003 (Design system)
- S-007-004 (App launch routing to this screen for unauthenticated users)

## Acceptance Criteria

- [ ] Given an unauthenticated user launches the app, when routed here, then the Welcome screen renders with Noxis wordmark and "Get Started" CTA on a pure black background
- [ ] Given the user taps "Get Started", when navigated, then the Tone Selection screen appears
- [ ] Given the user taps a tone card, when tapped, then it shows an accent glow border and the Continue button becomes active
- [ ] Given the user taps Continue after selecting a tone, when navigated, then the selection is stored and the Quick Profile screen opens
- [ ] Given the user returns to this screen (back navigation), when rendered, then the previously selected tone is still highlighted

## Negative Scenarios

- User taps Continue without selecting a tone → Continue button remains disabled; no action

## Edge Cases

- Very small screen (iPhone SE) → cards stack and scroll; no content is hidden below the fold if the user can scroll

## Dependencies

- S-007-003
- S-007-004

## Technical Requirements

- **`WelcomeView.swift`** and **`ToneSelectionView.swift`** — SwiftUI views
- **`OnboardingRouter`** — manages navigation flow through onboarding screens
- Tone selection stored to `OnboardingState` in-memory until auth is completed (S-001-005), then persisted to user profile

## Future Scope

- Animated Noxis logo or subtle particle effect on Welcome screen (v2 polish)

## Do Not Do

- In this story, do not implement authentication — that is S-001-005
- Do not show any feature preview or app tour — get to tone selection immediately

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

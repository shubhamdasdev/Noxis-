# Story: Implement Tone Mode Selection & Switching

---

**ID:** S-002-002
**Epic:** E-002
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **to choose how Noxis talks to me — as a sharp older brother, a high-end consultant, or a confident peer**, so that **the delivery style fits my personality while the underlying judgment stays the same**.

## Goal

The user selects a tone mode during onboarding and it persists across all sessions. The selected mode is injected into the system prompt alongside SOUL.md, modulating the delivery style without changing Noxis's core identity or opinions. The user can switch tone at any time from the Profile tab.

## Why / Rationale

Three distinct users will come to Noxis with different preferences for how they want to be spoken to. Locking everyone into one delivery style would alienate users who need warmth (The Rebuilder) or efficiency (The Upgrader). The tone is a delivery layer — not the soul. The opinions, judgment, and boundaries never change.

## Functional Requirements

**Tone modes:**
- `brother` — Sharp Older Brother: direct, slightly teasing, warm, occasional humor, pushes hard
- `consultant` — High-End Consultant: precise, structured, data-forward, no wasted words
- `peer` — Confident Peer: conversational, equal-to-equal, confident but not preachy

**Onboarding selection:**
- Presented as three glass cards on the Tone Selection screen (S-001-001)
- Each card shows the mode name, one-line description, and a sample response to the same prompt (e.g., "Rate my outfit")
- User taps a card to select; selected card highlights with a glass glow border
- Selection stored immediately to USER.md / user profile before proceeding

**Tone injection:**
- Selected tone mode name is appended to the system prompt below SOUL.md: `## Active Tone Mode: {mode}\n{tone-specific instructions}`
- Tone-specific instructions define register, vocabulary adjustments, and example phrases for that mode — sourced from SOUL.md's Tone Modes table
- Tone applies to all responses including daily briefs, module advice, and image judgments

**Switching:**
- Profile tab → Tone Mode setting → same three-card selector
- Change takes effect on the next message sent
- No confirmation dialog needed — change is instant and reversible

## Prerequisites

- S-002-001 (SOUL.md system prompt loading)
- S-007-005 (Profile tab) for post-onboarding switching

## Acceptance Criteria

- [ ] Given a user selects "brother" during onboarding, when they send any message, then the response uses the Sharp Older Brother register (direct, slightly teasing, warm)
- [ ] Given a user selects "consultant" during onboarding, when they send any message, then the response uses the High-End Consultant register (precise, no wasted words, structured)
- [ ] Given a user selects "peer" during onboarding, when they send any message, then the response uses the Confident Peer register (conversational, equal-to-equal)
- [ ] Given a user changes tone mode in Profile, when the next message is sent, then the new tone applies immediately
- [ ] Given any tone mode, when a response is generated, then SOUL.md's core identity, opinions, and boundaries are unchanged — only delivery differs
- [ ] Given a user with no tone selected (edge case), when a message is sent, then the system defaults to "peer"

## Negative Scenarios

- Tone mode value corrupted in storage → default to "peer" silently, no error shown to user
- User switches tone mid-conversation → next response uses new tone, prior conversation history is not retroactively re-toned

## Edge Cases

- User rapidly switches tone mode multiple times → last selection wins; no race condition on system prompt assembly

## Dependencies

- S-002-001

## Technical Requirements

- **Tone instruction file:** add a `tones/` folder with `brother.md`, `consultant.md`, `peer.md` — each contains the register, vocabulary rules, and sample phrases for that mode
- **System prompt assembly update:** `buildSystemPrompt(toneMode)` appends the appropriate tone file after SOUL.md
- **USER.md field:** `tone: brother | consultant | peer` — set during onboarding, updated on switch

## Future Scope

- User-defined custom tone (v2 — power users who want to fine-tune further)
- Tone analytics: which mode correlates with highest engagement

## Do Not Do

- In this story, do not allow the user to modify the tone instructions themselves — tone files are fixed, not user-editable
- In this story, do not implement more than three tone modes

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

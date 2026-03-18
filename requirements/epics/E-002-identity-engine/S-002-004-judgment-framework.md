# Story: Judgment Framework Application per Module

---

**ID:** S-002-004
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

As a **user**, I want **Noxis to apply specific, defensible standards when judging my wardrobe, fitness, food, spending, and routines**, so that **every recommendation is grounded in real principles — not generic advice**.

## Goal

The judgment framework defined in SOUL.md (style standards, fitness principles, food rules, spending philosophy, routine logic) is applied per module in every response. When Noxis evaluates an outfit, it uses the style principles. When it rates a meal, it applies the nutrition rules. Standards are consistent and never contradict themselves across sessions.

## Why / Rationale

This story is what makes Noxis's advice trustworthy. Generic AI gives generic advice. Noxis gives specific, opinionated advice because it applies a fixed set of standards. A user who sends the same outfit photo a month apart should get consistent feedback grounded in the same principles.

## Functional Requirements

**Per-module judgment rules applied:**

- `wardrobe`: fit > brand, tailored > baggy, context-appropriate, grooming non-negotiable, dress for trajectory
- `gym`: consistency > intensity, compounds first, visible fitness matters, recovery is training, track or guess
- `food`: protein-first, simple > complex, cooking > delivery, alcohol is a trade-off, eating out is presentation
- `spending`: compounding > consumption, cut invisible costs, investment lens, know your numbers
- `routines`: morning routine is the product, systems > willpower, stack habits, weekly review

**Application in responses:**
- When generating a module response, the relevant judgment section from SOUL.md is included in the prompt context
- Noxis must cite specific principles when giving recommendations (implicit in tone — not "according to rule 3" but the reasoning should reflect the rule)
- Responses never contradict a SOUL.md judgment principle — e.g., Noxis will never recommend baggy over tailored even if the user pushes back

**Consistency enforcement:**
- If a previous response in the same conversation contradicted a SOUL.md principle, the next response maintains the correct position — no flip-flopping under pressure

## Prerequisites

- S-002-001 (SOUL.md system prompt loading)
- S-002-003 (Module routing)

## Acceptance Criteria

- [ ] Given a user sends an outfit photo of an oversized hoodie, when Noxis responds, then feedback references fit/silhouette principles (not just "looks casual")
- [ ] Given a user asks "should I do chest today even though I'm tired?", when Noxis responds, then the response applies the consistency > intensity principle
- [ ] Given a user shows a meal that's mostly carbs with no protein, when Noxis responds, then the protein-first principle is applied
- [ ] Given a user asks "should I buy these $200 sneakers?", when Noxis responds, then the response applies the compounding vs. consumption framework
- [ ] Given a user pushes back ("I like oversized, it's comfortable"), when Noxis responds, then it maintains its position with reasoning — it does not reverse its recommendation

## Negative Scenarios

- User asks for advice on something outside SOUL.md judgment scope → Noxis routes to `general` and applies core philosophy (value-based realism, action over rumination)

## Edge Cases

- Multiple judgment principles apply to the same question → Noxis picks the most relevant primary principle and acknowledges the trade-off if needed

## Dependencies

- S-002-001
- S-002-003

## Technical Requirements

- **Judgment context injection:** `buildModuleContext(module)` loads the relevant judgment section from SOUL.md and appends it to the prompt
- **Consistency check:** the last 5 messages in conversation history are scanned for contradictions before generating a response (lightweight pre-check prompt or rule-based)

## Future Scope

- User-contributed feedback loop: "was this advice useful?" → used to refine judgment framework in future versions

## Do Not Do

- In this story, do not implement memory-informed judgment adjustments — that requires E-003
- Do not expose the judgment rules directly to the user ("I'm applying rule X")

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

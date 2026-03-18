# Story: Module-Aware Response Routing (AGENTS.md)

---

**ID:** S-002-003
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

As a **user**, I want **Noxis to automatically understand what area of my life I'm asking about**, so that **every response applies the right domain expertise without me having to specify a category**.

## Goal

Every incoming message or image is classified into one of the seven life modules defined in AGENTS.md (Wardrobe, Gym, Food, Spending, Routines, Social/Dating, General). The classification determines which judgment framework is applied, which memory is retrieved, and what structured data may be extracted for dashboard sync.

## Why / Rationale

Without routing, Noxis would apply generic judgment to every request. Module classification is what makes Noxis feel like a specialist across multiple domains rather than a general chatbot. It also drives the memory and dashboard extraction logic downstream.

## Functional Requirements

**Classification:**
- Every user message and image is classified into one of: `wardrobe`, `gym`, `food`, `spending`, `routines`, `social`, `general`
- Classification happens as the first step in the response pipeline, before memory retrieval
- Classification can detect multi-module requests (e.g., "what should I eat before the gym") → route to primary module (`food`) with secondary module tag (`gym`) for memory context
- Image messages are pre-classified by visual content: outfit photo → `wardrobe`, food photo → `food`, body photo → `gym`

**Module-specific logic injection:**
- Classification result is used to select the relevant section of AGENTS.md module routing rules (triggers, actions, memory reads/writes)
- The module context is appended to the prompt assembly: `## Active Module: {module}\n{module routing rules}`

**Classification confidence:**
- If confidence is low (ambiguous message), default to `general` — never surface "I don't know what you mean" as a module error
- Log classification result and confidence score for each message

## Prerequisites

- S-002-001 (SOUL.md system prompt loading)

## Acceptance Criteria

- [ ] Given a user sends an outfit photo, when classified, then the module is `wardrobe` and the wardrobe judgment framework applies
- [ ] Given a user asks "what should I eat before the gym", when classified, then primary module is `food`, secondary is `gym`, and both are loaded into context
- [ ] Given an ambiguous message like "I don't know what to do today", when classified, then module defaults to `general` and a cross-module response is generated
- [ ] Given a food photo is sent, when classified, then module is `food` and a nutritional assessment is returned
- [ ] Given classification completes, when logged, then module name and confidence score are present in the log entry

## Negative Scenarios

- Image cannot be classified visually → default to `general`; ask the user what area they want feedback on
- User asks about something outside all modules (e.g. car advice) → route to `general`; Noxis acknowledges it's outside its domain but applies its core philosophy if applicable

## Edge Cases

- User asks the same question in three different modules (e.g., "how do I improve?" in gym, food, and routines context on the same day) → each is classified and routed independently; no cross-contamination

## Dependencies

- S-002-001

## Technical Requirements

- **`classifyMessage(message, imageData?)` function:** returns `{ primaryModule, secondaryModule?, confidence }` using a fast classification call (can use a smaller/cheaper model for classification before the full response)
- **Module context files:** `modules/wardrobe.md`, `modules/gym.md`, etc. — each contains the triggers, actions, memory read/write spec from AGENTS.md
- **Pipeline integration:** classification result must be available to memory retrieval (S-003-004) and response generation (S-004-005)

## Future Scope

- User-defined custom modules (v2)
- Module usage analytics for understanding which domains drive most engagement

## Do Not Do

- In this story, do not implement dashboard data extraction — that is S-005-007
- In this story, do not implement memory retrieval — that is S-003-004
- Do not expose the module classification to the user in the UI

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# Release: Core Loop — Onboarding, Chat, Memory, Daily Brief

---

**ID:** R-001
**Project:** noxis
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Goal

Deliver the complete end-to-end Noxis experience for a single returning user: onboarding with tone selection, chat with opinionated AI responses, persistent memory that personalises advice over time, image input for outfit/meal/body checks, life module dashboard with wardrobe and gym tracking, and a daily morning brief. This release makes the product fully functional for early beta users and proves the core value proposition — honest, personalised judgment in under 10 seconds.

## Architecture Gate

> **Gate: Closed** — coding agents must not start story work until all ATs below are `Stage: Done`.

| ID | Title | Stage |
|----|-------|-------|
| AT-001-001 | iOS Project Scaffold + Design System Foundation | Backlog |
| AT-001-002 | FastAPI Backend Scaffold + Supabase Schema | Backlog |
| AT-001-003 | Identity Layer File Structure (SOUL.md + Tones + Modules) | Backlog |
| AT-001-004 | Mem0 + OpenAI Integration Setup | Backlog |

**Gate:** Closed

---

## Stories in Scope

Stories are listed in recommended build order — shell and foundations first, then features that depend on them.

### Foundation — App Shell

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-007-003 | Glass + Black Design System — Theme, Typography, Components | P0 | E-007 |
| S-007-001 | Bottom Tab Bar — 5 Tabs with Center Camera Button | P0 | E-007 |
| S-007-004 | App Launch & Splash Screen | P1 | E-007 |

### Onboarding

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-001-001 | Welcome & Tone Selection Screens | P0 | E-001 |
| S-001-002 | Quick Profile Screen | P1 | E-001 |
| S-001-005 | Authentication (Email / Apple Sign-In) | P0 | E-001 |
| S-001-003 | Conversational Onboarding in Chat | P1 | E-001 |
| S-001-004 | First Value Delivery | P1 | E-001 |

### Identity Engine

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-002-001 | Load SOUL.md as Fixed System Prompt | P0 | E-002 |
| S-002-002 | Implement Tone Mode Selection & Switching | P1 | E-002 |
| S-002-003 | Module-Aware Response Routing (AGENTS.md) | P1 | E-002 |
| S-002-004 | Judgment Framework Application per Module | P1 | E-002 |
| S-002-005 | Boundary Enforcement & Hard Redirects | P1 | E-002 |

### Memory & Personalization

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-003-001 | Memory Storage Layer (Mem0 Integration) | P0 | E-003 |
| S-003-002 | USER.md Auto-Generation from Onboarding | P0 | E-003 |
| S-003-004 | Decision Policy — Relevance & Quality Filtering | P0 | E-003 |
| S-003-003 | Memory Extraction from Chat Conversations | P1 | E-003 |

### Chat Experience

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-004-001 | Chat Interface — Send & Receive Messages | P0 | E-004 |
| S-004-005 | Orchestration Pipeline — Personality + Memory + Context | P0 | E-004 |
| S-004-006 | Streaming Response Display | P1 | E-004 |
| S-004-002 | Image Input — Camera Button & Photo Send | P1 | E-004 |
| S-004-003 | Image Classification & Module-Aware Judgment | P1 | E-004 |

### Life Modules

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-005-001 | Dashboard Grid — Module Cards Overview | P1 | E-005 |
| S-005-007 | Chat-to-Dashboard Data Extraction Pipeline | P1 | E-005 |
| S-005-002 | Wardrobe Module — Inventory & Outfit Log | P1 | E-005 |
| S-005-003 | Gym Module — Workout Logging & Streak Tracking | P1 | E-005 |

### Daily System

| Story | Title | Priority | Epic |
|-------|-------|----------|------|
| S-006-001 | Daily Brief Generation Engine | P0 | E-006 |
| S-006-002 | Daily Brief Screen — Display & History | P1 | E-006 |
| S-006-004 | Push Notification for Morning Brief | P1 | E-006 |

**Total: 29 stories** (11 × P0, 18 × P1)

---

## Out of Scope (deferred to R-002)

| Story | Title | Priority | Reason |
|-------|-------|----------|--------|
| S-007-002 | Side Panel — Conversation History | P2 | Nice-to-have; chat works without it |
| S-007-005 | Profile Tab — Settings, Tone Switch, Account | P2 | Tone switch available in onboarding; profile deferred |
| S-004-004 | Conversation History & Thread Management | P2 | Core chat works on a single thread initially |
| S-005-004 | Food Module | P2 | Wardrobe + Gym covers the module proof-of-concept |
| S-005-005 | Spending Module | P2 | Deferred |
| S-005-006 | Routines Module | P2 | Deferred |
| S-006-003 | Context-Aware App Entry (Morning → Daily Tab) | P2 | Deferred; app opens to Chat in R-001 |
| S-006-005 | Frequency Protection — Adaptive Brief Format | P2 | Deferred |
| S-006-006 | Curated Content Integration | P3 | Deferred |
| S-003-005 | Memory Contradiction Resolution | P2 | Deferred |
| S-003-006 | Daily Session Logs | P2 | Deferred |
| S-003-007 | Memory Review in Profile | P2 | Deferred |

---

## Key Dependencies

| Story | Depends On | Type |
|-------|-----------|------|
| S-001-001 | AT-001-001 (iOS scaffold + design system) | Architecture |
| S-001-005 | AT-001-002 (backend + Supabase auth) | Architecture |
| S-001-003 | S-001-001, S-001-002, S-004-001 | Story |
| S-001-004 | S-001-003, S-002-001, S-002-002 | Story |
| S-002-001 | AT-001-003 (identity layer files) | Architecture |
| S-002-002 | S-002-001 | Story |
| S-003-001 | AT-001-004 (Mem0 setup) | Architecture |
| S-003-002 | S-003-001 | Story |
| S-003-004 | S-003-001 | Story |
| S-003-003 | S-003-001, S-004-005 | Story |
| S-004-001 | S-007-003, S-007-001 | Story |
| S-004-005 | S-002-001, S-003-001, S-003-004, AT-001-004 | Story + Architecture |
| S-004-006 | S-004-001, S-004-005 | Story |
| S-004-002 | S-004-001, S-007-001 | Story |
| S-004-003 | S-004-002, S-002-003 | Story |
| S-005-001 | S-007-001, S-007-003 | Story |
| S-005-007 | S-004-005, S-003-003 | Story |
| S-005-002 | S-005-001, S-005-007 | Story |
| S-005-003 | S-005-001, S-005-007 | Story |
| S-006-001 | AT-001-002 (Supabase), AT-001-004 (OpenAI), S-003-001 | Story + Architecture |
| S-006-002 | S-006-001, S-007-001 | Story |
| S-006-004 | S-006-001 | Story |

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created — 29 stories, Architecture Gate with 4 ATs |

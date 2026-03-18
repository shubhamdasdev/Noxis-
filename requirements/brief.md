# Product Brief: Noxis

---

**Version:** 1.0
**Updated:** 2026-03-18
**Project:** noxis
**Repo:** https://github.com/shubhamdasdev/Noxis-.git
**Local:** /Users/myspace/Desktop/Noxis
**PM Persona:** pragmatist

---

## Problem Statement

Men who want to present better, build confidence, and operate at a higher level have no single system that combines honest feedback, structured life tracking, and daily momentum into one private experience. They piece together generic fitness apps, note-taking tools, and ChatGPT conversations — none of which know them, none of which have taste, and none of which push them to be better. Generic AI assistants answer questions but hold no memory, have no defined personality, and make no attempt to raise standards over time. Noxis is a personal AI operating system with a fixed, opinionated identity — inspired by the value-based realism of Dr. Orion Tarban's framework — that delivers honest judgment across wardrobe, fitness, food, spending, and routines, improving its delivery as it learns the user over time through structured memory.

## North Star

**North Star:** A man opens Noxis, gets an honest, personalized judgment on any life decision — from outfit to spending to routine — in under 10 seconds, and that judgment gets sharper every week.

## Goals & Success Metrics

| Goal | Metric | Target |
|------|--------|--------|
| Drive daily engagement | DAU / MAU ratio | 40% by month 3 |
| Establish morning ritual habit | Users who open Daily Brief 5+ days/week | 30% of WAU by month 3 |
| Prove memory value | Users with 50+ memory entries | 500 users by month 3 |
| Image feedback adoption | Outfit/meal/body check images sent / week | 2,000 across all users by month 3 |
| Deliver consistently on-brand advice | User rates advice as "on-point" | ≥ 80% of sessions |
| Retention | D30 retention | 25% by month 3 |

## Roles

| Role | Description |
|------|-------------|
| **guest** | Unauthenticated visitor — has not yet created an account |
| **user** | Authenticated user — has a verified account and full access to personal Noxis experience |

## Personas

| Persona | Primary Role | Goals | Pain Points |
|---------|-------------|-------|-------------|
| **The Invisible Man** | guest → user | Stop being wallpaper — become visible, presentable, and memorable in social and professional settings | Doesn't know where to start; generic self-help feels fake; no honest feedback loop; women and peers overlook him |
| **The Upgrader** | user | Already competent but wants to reach top-tier — sharpen wardrobe, optimize fitness, tighten routines, spend smarter | Scattered tools that don't connect; no single system that knows his standards and holds him to them; plateau without accountability |
| **The Rebuilder** | user | Coming out of a rough patch — needs structure, daily momentum, and a system that pushes without judging his starting point | Shame about current state; overwhelmed by where to begin; needs action-first guidance, not therapy or sympathy |

## Product Areas

| Area | Description |
|------|-------------|
| Identity & Personality Engine | Noxis's fixed character (SOUL.md), judgment framework, three selectable tone modes, and behavioral boundaries — the soul of the product, same for every user |
| Memory & Personalization | Structured memory (Mem0-style) storing user preferences, habits, wardrobe, decisions, and patterns — filtered by a decision policy layer that ensures only high-quality, relevant memory influences responses |
| Chat Experience | Core conversation interface — text + image input, instant opinionated feedback via center-bar camera, the primary way users interact with Noxis |
| Life Modules Dashboard | Structured surfaces for wardrobe planning, gym tracking, food habits, spending awareness, and routines — hybrid data flow from chat extraction + manual input |
| Daily System | Cron-style morning delivery — tailored insights, progress-grounded affirmations, and curated content that drives action, not passive consumption |
| Onboarding & User Profile | Tone selection (Sharp Older Brother / High-End Consultant / Confident Peer), baseline capture, first-value delivery — hybrid visual screens → conversational completion in chat |

## Scope

### In Scope (v1.0.0)

- iOS-first mobile app (black + glass premium aesthetic)
- Fixed personality layer via SOUL.md + AGENTS.md architecture (shipped as versioned artifacts, not user-editable)
- Three tone modes selectable during onboarding: Sharp Older Brother, High-End Consultant, Confident Peer
- Chat interface with text and image input
- Center-bar camera button for instant outfit/meal/body check photos (Instagram-style tab bar)
- Vision capability for image-based judgment (outfit rating, meal feedback, body progress)
- Structured memory layer (Mem0 or equivalent) for user preferences, habits, and patterns
- Decision policy layer filtering what gets remembered and how strongly it influences responses
- Orchestration layer combining personality + memory + context + optional search
- Life modules dashboard: Wardrobe, Gym, Food, Spending, Routines
- Hybrid data flow: chat → dashboard (Noxis extracts and files mentions) + manual input per module
- Daily morning brief system with insights, progress-grounded affirmations, and curated content
- Bottom tab bar navigation: Chat | Modules | 📷 Camera | Daily | Profile
- Side panel (ChatGPT mobile-style): conversation history, module quick-jump, settings
- Hybrid onboarding: 2-3 visual glass screens → conversational completion in chat
- Authentication (email/password or Apple Sign-In)
- Single user — no multi-user or sharing
- Legal disclaimer during onboarding: lifestyle AI, not clinical

### Out of Scope (v1.0.0)

- Android app
- Web or desktop app
- Social features, sharing, or multi-user interaction
- E-commerce integrations (outfit purchasing, meal ordering)
- Therapist or mental health diagnostic features (Noxis is lifestyle, not clinical)
- User-editable personality (SOUL.md is fixed, same for everyone)
- Real-time voice interaction
- Wearable device integration (Apple Watch, fitness bands)
- Third-party integrations (calendar, health APIs, banking) — considered for v2

## Constraints

| Constraint | Detail |
|------------|--------|
| Platform | iOS-first; backend API (Python or Node) |
| AI Provider | Vision-capable model required — OpenAI GPT-4o, Anthropic Claude, or equivalent; TBD at architecture stage |
| Memory | Mem0 or equivalent structured store with metadata (category, confidence, timestamp) |
| Decision Policy | Memory must be filtered before reaching response — no raw memory dump into context |
| Design | Black + glass aesthetic; no generic iOS chrome; must feel premium and private |
| Personality | SOUL.md and AGENTS.md are version-controlled, shipped fixed — not user-modifiable |
| iOS Target | Minimum deployment target: iOS 17 |
| Security | All user data (memory, chat history, profile) encrypted at rest |
| Daily System | Must be timezone-aware — delivers at user-appropriate morning time |
| Content | No empty validation, no motivational filler, no therapy — every output grounded in judgment or action |

## Open Questions

| # | Question | Owner |
|---|----------|-------|
| 1 | Which AI provider for text + vision (OpenAI GPT-4o, Anthropic Claude, open-source, hybrid)? | Engineering |
| 2 | Mem0 hosted vs. self-hosted vs. custom memory layer? | Engineering |
| 3 | How is the daily system triggered — server cron + push notification, or device-side? | Engineering |
| 4 | Monetization model — subscription tiers, freemium, or one-time purchase? | PM |
| 5 | Content curation source for Daily system — RSS, curated editorial, AI-generated summaries? | PM |
| 6 | Offline capability — should chat work offline with cached personality, or require connectivity? | Engineering |
| 7 | Push notification strategy — daily brief notification, or silent until opened? | PM |
| 8 | Backend language — Python (FastAPI) or Node (Express/Hono)? | Engineering |

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 0.1 | — | Initial draft |
| 2026-03-18 | 1.0 | — | Full rewrite — added personas, tone modes, nav structure, image-first UX, hybrid onboarding, expanded scope and constraints |

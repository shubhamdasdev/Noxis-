# Noxis — Product Brief

**Project:** Noxis
**Repo:** https://github.com/shubhamdasdev/Noxis-.git
**Local:** ../../../../Desktop/Noxis
**PM Persona:** pragmatist
**Version:** 0.1
**Date:** 2026-03-18

---

## Problem Statement

There is no personal AI system that acts as a consistently opinionated, taste-driven advisor across the full surface area of how a person presents and operates — style, fitness, food, spending, and daily habits. Generic AI assistants answer questions but hold no memory of you, have no defined personality, and make no attempt to raise your standards over time. Noxis fixes this: a single system with a fixed identity, a growing memory of you, and a daily cadence designed to pull you toward a sharper, higher-functioning version of yourself.

---

## Goals & Success Metrics

| Goal | Metric | Target |
|------|--------|--------|
| Deliver consistently on-brand advice | User rates advice as "on-point" | ≥ 80% of sessions |
| Build a useful memory of the user over time | Memories stored and successfully retrieved | ≥ 50 memories within first 30 days |
| Drive daily engagement through the morning system | Morning brief opened | ≥ 5 days/week |
| Become the user's primary style/lifestyle reference | Session frequency | ≥ 1 session/day after week 2 |

---

## Roles

| Role | Description |
|------|-------------|
| user | The sole owner and operator of the system — the person whose life Noxis is built around |

---

## Personas

**The Operator** — ambitious, self-aware, building deliberately. Values specificity over encouragement. Wants the system to challenge his defaults, not validate them. Measures progress by concrete outcomes: better outfits worn, better food chosen, better decisions made. Has no patience for vague advice or motivational filler.

---

## Product Areas

| Area | Description |
|------|-------------|
| Personality & Behavior Layer | Fixed identity system — defines how Noxis thinks, speaks, and responds across all interactions. Markdown-based, OpenClaw-style architecture. |
| Memory Layer | Structured, persistent memory of the user — preferences, habits, wardrobe inventory, decisions, and patterns. Powered by Mem0 or equivalent. |
| Decision Policy Layer | Filters and ranks stored memories before they influence a response — ensures only high-quality, relevant context shapes outputs. |
| Orchestration Layer | Combines personality + relevant memory + real-time context + optional search into a single, consistent response pipeline. |
| Chat Interface | iOS-first, premium chat UI for instant text and image-based advice. Black + glass aesthetic. |
| Dashboard & Life Modules | Structured view of the user's life across wardrobe, gym, food, spending, and routines. Each module is a curated record and planning surface. |
| Daily System | Cron-driven morning delivery — insights, affirmations, and curated content calibrated to the user's goals and recent patterns. |

---

## Scope

### In Scope (v1)
- Personality layer defined as a fixed markdown system prompt
- Memory layer: store and retrieve user preferences, wardrobe, habits, and decisions
- Decision policy: relevance + quality filter before memory injection
- Orchestration: personality + memory + context → response
- iOS chat: text and image input, opinionated responses
- Dashboard modules: wardrobe, gym, food, spending, routines (read + write)
- Daily system: automated morning brief with insights and affirmations
- Single user — no multi-user or sharing

### Out of Scope (v1)
- Web or Android clients
- Social or sharing features
- Third-party integrations (calendar, health APIs, banking) — considered for v2
- Voice interface
- Multi-user / team accounts

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Platform | iOS-first; backend API (Python or Node) |
| AI Provider | OpenAI or Anthropic — TBD at architecture stage |
| Memory | Mem0 or equivalent vector/structured store |
| Design | Black + glass aesthetic; minimal, premium feel |
| Single user | No auth system complexity required for v1 — single-owner deployment |
| Personality locked | Casanova-inspired identity is fixed, not configurable by the user in v1 |

---

## Open Questions

| # | Question | Owner |
|---|----------|-------|
| 1 | Which AI provider — OpenAI or Anthropic? | Engineering |
| 2 | Mem0 hosted vs. self-hosted? | Engineering |
| 3 | How is the daily system triggered — device-side cron, server cron, or push notification? | Engineering |
| 4 | What is the wardrobe input method — manual entry, photo scan, or both? | PM |
| 5 | Does the dashboard persist data locally (on-device) or server-side? | Engineering |
| 6 | What curated content sources feed the daily system — web search, curated RSS, or manual? | PM |

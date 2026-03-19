# User Flow: Morning Brief

---

**ID:** UF-003
**Project:** noxis
**Epic:** E-003, E-006, E-007
**Persona:** The Rebuilder — returning user, open to the daily reset, wants to know where to focus today
**Status:** Draft
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Overview

The morning ritual loop. Covers how Noxis greets the user on their first open of the day: the app context-routes to the Daily tab if a new brief is ready, the user reads their personalized insight and action items, optionally taps into a curated content piece, then transitions to chat or a module to act on what they read. Also covers the push notification path — tapping the morning notification lands directly on the Daily tab.

Multi-epic: the app shell (E-007) handles launch routing and the tab bar; the memory engine (E-003) powers brief personalization; the daily system (E-006) generates, stores, and displays the brief.

## Entry Point

- Cold app launch before 10am, unread brief exists → context-routes to Daily tab (S-006-003)
- User taps the push notification sent at their brief delivery time (S-006-004)
- User manually taps the Daily tab from any other tab

## Stories Covered

- S-007-004 — App Launch & Splash Screen
- S-006-003 — Context-Aware App Entry (Morning → Daily Tab)
- S-006-004 — Push Notification for Daily Brief
- S-006-001 — Daily Brief Generation Engine
- S-006-002 — Daily Brief Screen — Display & History
- S-006-006 — Curated Content in Brief
- S-003-002 — Memory Retrieval for Brief (brief uses recent memories)
- S-007-001 — Bottom Tab Bar

## Flow

```mermaid
flowchart TD
    Entry_Launch([Entry: Cold launch — before 10am]) --> Splash[\"Screen: Splash\"]
    Entry_Notification([Entry: Tap push notification]) --> DailyTab[\"Screen: Daily Tab\"]
    Entry_Tab([Entry: Tap Daily tab manually]) --> DailyTab

    Splash -->|\"Auth check — authenticated, unread brief, before 10am\"| DailyTab
    Splash -->|\"Auth check — no brief or after 10am\"| Exit_chat([Exit: Chat tab])

    DailyTab -->|\"Brief exists — status: generated\"| TodayCard[\"Widget: Today Brief Card\"]
    DailyTab -->|\"Brief generating — status: pending\"| ShimmerCard[\"Widget: Loading Shimmer Card\"]
    DailyTab -->|\"No brief for today\"| EmptyState[\"Widget: No Brief State\"]

    ShimmerCard -->|\"Generation completes\"| TodayCard

    TodayCard -->|\"Viewed — viewed_at set on appear\"| TodayCard
    TodayCard -->|\"Tap curated content card\"| CuratedContent[\"Overlay: SFSafariViewController\"]
    TodayCard -->|\"Scroll down\"| BriefHistory[\"Widget: Brief History List\"]

    CuratedContent -->|\"Done / dismiss\"| TodayCard

    BriefHistory -->|\"Tap past brief row\"| BriefHistory
    BriefHistory -->|\"Tap past brief row again\"| BriefHistory

    TodayCard -->|\"Tap Chat tab or swipe\"| Exit_chat
    TodayCard -->|\"Tap Modules tab\"| Exit_modules([Exit: Modules tab])
    BriefHistory -->|\"Tap Chat tab or swipe\"| Exit_chat
    BriefHistory -->|\"Tap Modules tab\"| Exit_modules

    EmptyState -->|\"Tap Chat tab or swipe\"| Exit_chat
```

## Screens

### Screen: Splash

- **Purpose:** Hold the premium brand impression during auth state check on launch
- **Key content:** Pure black background; Noxis wordmark centered (SF Pro Display Bold, 28pt, white); no animation
- **Primary action:** None — purely transitional
- **Transitions:**
  - `Authenticated + unread brief + before 10am` → Screen: Daily Tab
  - `Authenticated + no brief or after 10am` → Exit: Chat tab
  - `Unauthenticated` → Screen: Welcome (see UF-001)
- **Stories:** S-007-004, S-006-003

### Screen: Daily Tab

- **Purpose:** The user's daily ritual surface — where Noxis's morning preparation is delivered and browsed
- **Key content:**
  - Navigation title: "Daily" (Display typography, large title)
  - Scrollable content: Today Brief Card at top, Brief History list below
  - Pull-to-refresh to reload brief status
- **Primary action:** Read today's brief; scroll for history
- **Transitions:**
  - `Brief status: generated` → Widget: Today Brief Card
  - `Brief status: pending` → Widget: Loading Shimmer Card → (auto-updates to Today Brief Card on completion)
  - `No brief today` → Widget: No Brief State
  - `Tap Chat tab` → Exit: Chat tab (session preserved)
  - `Tap Modules tab` → Exit: Modules tab
- **Stories:** S-006-002, S-007-001

### Widget: Today Brief Card

- **Purpose:** Deliver today's personalized insight and action items in a premium glass format
- **Key content:**
  - Date header: "Today — [Day, Month Date]" (Caption, muted)
  - Insight paragraph (Body typography)
  - Action items: 3 bullets max, each with a subtle left-accent bar
  - Curated content card (if present): title, type badge, description; tapping opens in-app browser
  - `viewed_at` is set on `daily_briefs` record on first `onAppear` if `viewed_at` is null
- **Primary action:** Read; optionally tap curated content
- **Transitions:**
  - `Tap curated content card` → Overlay: SFSafariViewController
  - `Scroll down` → Widget: Brief History List
- **Stories:** S-006-002, S-006-006

### Widget: Loading Shimmer Card

- **Purpose:** Hold the card space while brief generation is still in progress
- **Key content:** GlassCard of same dimensions as Today Brief Card; animated shimmer effect on all text areas; no placeholder text
- **Primary action:** None — waits for brief to complete
- **Transitions:**
  - `Brief status changes to generated` → replaced by Widget: Today Brief Card (no user interaction needed; auto-refresh on timer or pull-to-refresh)
- **Stories:** S-006-002

### Widget: No Brief State

- **Purpose:** Inform the user gracefully when no brief was generated for today
- **Key content:** State card with message: "No brief today — keep using Noxis and tomorrow's brief will reflect your activity"
- **Primary action:** None
- **Transitions:**
  - User navigates away via tab bar
- **Stories:** S-006-002

### Overlay: SFSafariViewController

- **Purpose:** Open curated content (article, video link) in-app without leaving the Daily tab context
- **Key content:** Native iOS in-app browser showing the curated URL; standard Safari controls (back/forward, share, done)
- **Primary action:** Read the content
- **Transitions:**
  - `Tap Done` → overlay dismissed; Today Brief Card restored
- **Stories:** S-006-006

### Widget: Brief History List

- **Purpose:** Let the user browse past briefs to notice patterns and revisit past insights
- **Key content:**
  - Vertical list of up to 30 past briefs (excluding today), sorted newest-first
  - Each row: date (Caption), truncated first sentence of insight, accent-color dot if `viewed_at` is null
  - Tap to expand row in-place (accordion) — shows full insight, action items, curated content
  - Tap again to collapse
- **Primary action:** Tap a row to read a past brief
- **Transitions:**
  - `Tap expanded row again` → collapses back to summary view
  - User navigates away via tab bar
- **Stories:** S-006-002

---

## Exit Points

- **Continue to Chat:** User taps Chat tab from Daily tab → same-session chat loads; brief context not passed automatically (user initiates chat about it if desired)
- **Open Modules:** User taps Modules tab → dashboard grid; see UF-004
- **Already read brief (returning to app):** If `viewed_at` is already set, the app opens to Chat tab as normal (context-aware routing bypasses Daily tab)
- **Error (brief load fails):** Inline error state: "Couldn't load your brief — pull to refresh"; both today card and history list replaced by error state; no blocking modal

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

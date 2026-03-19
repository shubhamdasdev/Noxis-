# User Flow: Dashboard & Module Logging

---

**ID:** UF-004
**Project:** noxis
**Epic:** E-003, E-005, E-007
**Persona:** The Upgrader — returning user, wants to see the data Noxis has collected and add to it
**Status:** Draft
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Overview

The module browsing and manual logging loop. Covers how a user enters the Modules tab, reads the at-a-glance dashboard grid, drills into any of the five life modules (Wardrobe, Gym, Food, Spending, Routines), views auto-extracted data surfaced by chat, and manually logs new entries directly in the module. Module data flows two ways: extracted automatically from chat (S-005-007) and entered manually via the module screens (S-005-002 through S-005-006).

Multi-epic: the app shell (E-007) provides tab navigation; the memory engine (E-003) supplies the extraction pipeline that auto-populates modules; the life modules epic (E-005) owns the dashboard grid and all five module detail screens.

## Entry Point

- User taps the Modules tab from any screen
- Or: side panel module quick-jump from Chat tab (see UF-002)

## Stories Covered

- S-005-001 — Dashboard Grid — Module Cards Overview
- S-005-002 — Wardrobe Module — Inventory & Outfit Log
- S-005-003 — Gym Module — Workout Log & Streak
- S-005-004 — Food Module — Meal Log
- S-005-005 — Spending Module — Purchase Tracker
- S-005-006 — Routines Module — Habit Tracker
- S-005-007 — Chat-to-Dashboard Extraction Pipeline
- S-007-001 — Bottom Tab Bar
- S-007-002 — Side Panel (quick-jump entry point)

## Flow

```mermaid
flowchart TD
    Entry_Tab([Entry: Tap Modules tab]) --> DashboardGrid[\"Screen: Dashboard Grid\"]
    Entry_SidePanel([Entry: Side panel module quick-jump]) --> ModuleDetail[\"Screen: Module Detail\"]

    DashboardGrid -->|\"Tap Wardrobe card\"| ModuleDetail
    DashboardGrid -->|\"Tap Gym card\"| ModuleDetail
    DashboardGrid -->|\"Tap Food card\"| ModuleDetail
    DashboardGrid -->|\"Tap Spending card\"| ModuleDetail
    DashboardGrid -->|\"Tap Routines card\"| ModuleDetail
    DashboardGrid -->|\"Pull to refresh\"| DashboardGrid

    ModuleDetail -->|\"Tap Add Item / Log Entry FAB\"| AddEntrySheet[\"Overlay: Add Entry Sheet\"]
    ModuleDetail -->|\"Tap a data row / card\"| EntryDetail[\"Screen: Entry Detail\"]
    ModuleDetail -->|\"Back button\"| DashboardGrid

    AddEntrySheet -->|\"Save\"| ModuleDetail
    AddEntrySheet -->|\"Cancel\"| ModuleDetail

    EntryDetail -->|\"Swipe to delete / delete action\"| ModuleDetail
    EntryDetail -->|\"Back button\"| ModuleDetail

    ModuleDetail -->|\"Chat extraction writes new data (async)\"| ModuleDetail

    DashboardGrid -->|\"Tap Chat tab\"| Exit_chat([Exit: Chat tab])
    ModuleDetail -->|\"Tap Chat tab\"| Exit_chat
```

## Screens

### Screen: Dashboard Grid

- **Purpose:** Single at-a-glance view of all five life modules — status summaries that make the user want to drill in
- **Key content:**
  - Screen title "Your Life" in Display typography, above the grid
  - 2-column `LazyVGrid` with 5 module cards: Wardrobe, Gym, Food, Spending, Routines (in that order)
  - Each card: module icon (SF Symbol), module name (Headline), one-line status summary (Caption, muted)
  - Status summaries: Wardrobe → "N items in your wardrobe"; Gym → "N-day streak" or "No workouts logged yet"; Food → "N meals logged this week"; Spending → "N purchases this week"; Routines → "N habits active"
  - Loading state: shimmer on status summary lines while `GET /api/v1/modules/summary` is in flight
  - Pull-to-refresh
- **Primary action:** Tap a module card to open that module's detail screen
- **Transitions:**
  - `Tap any module card` → Screen: Module Detail (for that module)
  - `Pull to refresh` → reloads summary data
- **Stories:** S-005-001, S-007-001

### Screen: Module Detail

- **Purpose:** The per-module data view — shows auto-extracted and manually logged data, with the ability to add new entries
- **Key content:** Varies by module (see sub-sections below). All modules share the same structural pattern:
  - Navigation bar: module name as title + back button
  - Primary data list or grid (auto-extracted from chat + manually added)
  - Floating Add button (bottom right) — opens Add Entry Sheet
  - Empty state when no data exists yet
  - Pull-to-refresh
- **Primary action:** Read logged data; tap to drill into an entry; tap FAB to add manually
- **Transitions:**
  - `Tap Add Item / Log Entry FAB` → Overlay: Add Entry Sheet
  - `Tap a data item / row` → Screen: Entry Detail
  - `Back button` → Screen: Dashboard Grid
  - `Chat extraction fires (async)` → module data updates in place; no disruptive reload
- **Stories:** S-005-002 through S-005-006, S-005-007

#### Module: Wardrobe

- **Two-tab layout:** Inventory tab (3-column grid of clothing items: photo thumbnail, category label, notes truncated) + Outfits tab (vertical list of Noxis-rated outfits: photo, rating phrase, date)
- **Add Item sheet:** photo picker (library or camera), category picker, optional notes (max 140 chars)
- **Item tap:** opens full photo + full notes + date added + swipe-to-delete
- **Auto-populated by:** chat image classification when outfit photos are sent (S-005-007)

#### Module: Gym

- **Layout:** streak counter (prominent, top) + chronological workout log (vertical list: date, workout type, duration, Noxis's key feedback line)
- **Add Workout sheet:** date picker, workout type (dropdown: Strength / Cardio / Mobility / Sport / Other), duration, optional notes
- **Auto-populated by:** Noxis extracting workout mentions from chat (S-005-007)

#### Module: Food

- **Layout:** this-week meal count (top) + chronological meal log (vertical list: meal name/description, meal type badge: Breakfast/Lunch/Dinner/Snack, date, Noxis's quality flag if assessed)
- **Add Meal sheet:** meal description (free text, required), meal type picker, date/time picker, optional notes
- **Auto-populated by:** Noxis extracting meal mentions from chat (S-005-007)

#### Module: Spending

- **Layout:** this-week spend total (top, accent color) + chronological purchase list (vertical list: item name, amount, category badge: Clothing/Food/Fitness/Entertainment/Other, date)
- **Add Purchase sheet:** item name, amount (numeric input), category picker, date picker
- **Auto-populated by:** Noxis extracting purchase mentions from chat (S-005-007)

#### Module: Routines

- **Layout:** active habits list with completion streaks (habit name, streak count, last completed date, completion toggle for today); below: inactive/archived habits
- **Add Habit sheet:** habit name (free text), frequency (Daily / Weekdays / Custom days), optional target time
- **Completion toggle:** tap the toggle on any habit row to mark today as complete; updates streak count immediately (optimistic UI)
- **Auto-populated by:** Noxis extracting routine commitments from chat (S-005-007)

### Overlay: Add Entry Sheet

- **Purpose:** Manual data entry without leaving the module context
- **Key content:** Modal sheet with module-specific fields (see Module Detail sub-sections above); Save CTA (primary, disabled until required fields filled); Cancel (secondary)
- **Primary action:** Fill fields and tap Save
- **Transitions:**
  - `Save (valid form)` → sheet dismissed; new entry appended to the module list immediately (optimistic UI); written to backend async
  - `Cancel` → sheet dismissed; no data written
- **Stories:** S-005-002, S-005-003, S-005-004, S-005-005, S-005-006

### Screen: Entry Detail

- **Purpose:** Full view of a single logged item — shows all fields plus Noxis's extracted context if present
- **Key content:** Full data for the entry (varies by module: full photo, full notes, Noxis rating, date, etc.); delete action (swipe gesture or trash icon in nav bar)
- **Primary action:** Read; optionally delete
- **Transitions:**
  - `Delete confirmed` → entry removed; screen pops back to Module Detail
  - `Back button` → Module Detail
- **Stories:** S-005-002, S-005-003, S-005-004, S-005-005, S-005-006

---

## Exit Points

- **Switch to Chat:** User taps Chat tab from any module screen → chat session preserved; module data updates (from extraction) continue in background
- **Return to Dashboard:** Back navigation from any module detail → Dashboard Grid
- **Chat extraction (async):** After a chat message is processed and module data is extracted, the relevant module's data store updates silently — next time the user opens that module, the new entry is already there (S-005-007)
- **Error (data write fails):** Optimistic UI rolls back; inline error toast: "Couldn't save — tap to retry"; no data loss (form input preserved)
- **Error (module data load fails):** Status summary shows "—" dash on dashboard cards; module detail shows inline error state with pull-to-refresh prompt; navigation still works

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

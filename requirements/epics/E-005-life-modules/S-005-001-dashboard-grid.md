# Story: Dashboard Grid — Module Cards Overview

---

**ID:** S-005-001
**Epic:** E-005
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **to see all my life modules at a glance on a single grid screen**, so that **I can quickly understand where I stand across each area of my life before drilling into any one module**.

## Goal

The Modules tab renders a 2-column grid of five glass module cards: Wardrobe, Gym, Food, Spending, and Routines. Each card shows the module name, a distinct icon, and a one-line status summary pulled from live module data. Tapping any card navigates to that module's detail screen.

## Why / Rationale

The dashboard grid is the navigational entry point for the entire life module system. Without it, users have no way to access module detail screens and the modules are invisible features. The one-line summary per card turns the grid into a functional at-a-glance status view rather than just navigation.

## Functional Requirements

- The Modules tab renders a `LazyVGrid` with 2 columns and 5 module cards: Wardrobe, Gym, Food, Spending, Routines — in that order
- Each card is a `ModuleCard` component (defined in S-007-003) with:
  - Module icon (SF Symbol): Wardrobe → `tshirt`, Gym → `figure.strengthtraining.traditional`, Food → `fork.knife`, Spending → `creditcard`, Routines → `checkmark.seal`
  - Module name in Headline typography
  - One-line status summary in Caption typography, muted color
- Status summary copy per module:
  - Wardrobe: "N items in your wardrobe" (N = total wardrobe item count)
  - Gym: "N-day streak" or "No workouts logged yet"
  - Food: "N meals logged this week" or "Nothing logged yet"
  - Spending: "N purchases this week" or "Nothing tracked yet"
  - Routines: "N habits active" or "No routines set up yet"
- Status data is fetched once on screen appear from `GET /api/v1/modules/summary`; a loading shimmer is shown while fetching
- Tapping any module card navigates to that module's detail screen (NavigationLink)
- Pull-to-refresh reloads the summary data
- The grid title "Your Life" is displayed in Display typography above the grid, left-aligned

## Prerequisites

- S-007-001 (bottom tab bar — the Modules tab must exist to host this view)
- S-007-003 (design system — ModuleCard component must be defined)

## Acceptance Criteria

- [ ] Given the user opens the Modules tab, when the screen renders, then exactly 5 module cards are displayed in a 2-column grid in the defined order (Wardrobe, Gym, Food, Spending, Routines)
- [ ] Given module data is available, when the screen renders, then each card shows the correct one-line status summary with accurate data (not placeholder copy)
- [ ] Given the data fetch is in progress, when the screen is first displayed, then each card shows a shimmer loading animation in place of the status summary line
- [ ] Given the user taps the Gym card, when the navigation occurs, then the Gym module detail screen is pushed onto the navigation stack
- [ ] Given the user pulls down on the grid, when the pull-to-refresh gesture completes, then the summary data is re-fetched and the status lines update
- [ ] Given a module has no data yet (e.g., no wardrobe items), when the card renders, then the appropriate "nothing logged yet" fallback copy is shown — never a blank status line
- [ ] Given the screen title "Your Life", when the Modules tab is active, then it is displayed in Display typography above the grid

## Negative Scenarios

- `GET /api/v1/modules/summary` fails → status summary lines show a muted "—" dash placeholder; the grid cards are still interactive and navigation still works; no blocking error state
- User taps a module card while the summary is still loading → navigation proceeds immediately to the module detail screen; the status summary loading state is dismissed

## Edge Cases

- Device in small screen (iPhone SE) → the 2-column grid compresses gracefully; cards maintain their aspect ratio with reduced horizontal padding; no content clipping
- User has a very large wardrobe count (999+ items) → "999+ items in your wardrobe"; no overflow

## Dependencies

- S-007-001 (bottom tab bar)
- S-007-003 (design system)

## Technical Requirements

- **`ModulesTabView.swift`** — top-level view for the Modules tab; contains the `LazyVGrid` with 2 flexible columns; `NavigationStack` for module detail pushes
- **`ModuleCard.swift`** — from design system (S-007-003); accepts `module: LifeModule`, `status: String?`, `isLoading: Bool`
- **`ModulesSummaryViewModel.swift`** — `@Published var summaries: [LifeModule: String]`; `@Published var isLoading: Bool`; `func load()` → `GET /api/v1/modules/summary` returns `{ wardrobe: { item_count }, gym: { streak }, food: { meals_this_week }, spending: { purchases_this_week }, routines: { active_habits } }`
- **`LifeModule` enum:** `wardrobe | gym | food | spending | routines` — shared across E-005 and used in memory categorization
- **Pull-to-refresh:** `.refreshable { await viewModel.load() }`
- **Navigation:** each `ModuleCard` is wrapped in a `NavigationLink(destination: moduleDetailView(for: module))`
- **Status string assembly:** `ModuleStatusFormatter.format(module:, data:) -> String` — pure function, unit testable

## Future Scope

- Module reordering (user can drag modules to their preferred order)
- Module-level notification dots (unread insights, streak alerts)
- A sixth module card for a user-defined custom life area

## Do Not Do

- In this story, do not build any module detail screen — those are S-005-002 through S-005-006
- Do not display charts, graphs, or trend indicators on the grid cards — status line text only in v1
- Do not show the full module data inline — tap to drill in

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

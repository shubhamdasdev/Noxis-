# Design Spec — Screen: Dashboard Grid

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-004
**Screen:** Screen: Dashboard Grid
**Stories:** S-005-001, S-007-001
**Components:** ModuleCard, ShimmerCard, EmptyState
**Updated:** 2026-03-18

---

## Screen: Dashboard Grid

**Story context:** The user's life at a glance — five module cards in a 2-column grid, each showing a live status summary. The entry point for all module drill-downs.

### Layout & Hierarchy

1. Primary: 2-column LazyVGrid of 5 ModuleCards — fills the scroll view body
2. Secondary: "Your Life" header — Display typography, left-aligned, above the grid
3. Pull-to-refresh over the entire ScrollView

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Screen title | Text (Display, 28pt, white) | "Your Life" — left-aligned, 16pt horizontal padding; use `.navigationTitle` with large title style OR a manual Text above the grid |
| Module grid | `LazyVGrid(columns: [GridItem(.flexible()), GridItem(.flexible())], spacing: 12)` | 16pt horizontal padding; 12pt row gap |
| Module card | ModuleCard | Each card wraps a `NavigationLink` to its module's detail screen |
| Status summary loading | ShimmerCard | Shimmer replaces the status line (Caption row) only — icon and module name still render |
| Network error fallback | `Text("—")` (Caption, Text secondary) | Replaces status summary on load failure; card still tappable |

### Module Card Content

| Module | SF Symbol | Name | Status format |
|--------|-----------|------|---------------|
| Wardrobe | `tshirt` | Wardrobe | "N items in your wardrobe" / "No items yet." |
| Gym | `figure.strengthtraining.traditional` | Gym | "N-day streak" / "No workouts logged yet." |
| Food | `fork.knife` | Food | "N meals logged this week" / "Nothing logged yet." |
| Spending | `creditcard` | Spending | "N purchases this week" / "Nothing tracked yet." |
| Routines | `checkmark.seal` | Routines | "N habits active" / "No routines set up yet." |

Cards display in the order above (top-left → top-right → row 2 left → row 2 right; Routines card spans or aligns to left in final row).

### Copy

| Element | Copy |
|---------|------|
| Screen title | Your Life |
| Wardrobe empty | No items yet. |
| Gym no data | No workouts logged yet. |
| Food no data | Nothing logged yet. |
| Spending no data | Nothing tracked yet. |
| Routines no data | No routines set up yet. |

### States

- **Default (data loaded):** Grid with all 5 cards; each card shows its live status summary
- **Loading:** All 5 cards render with icon and module name visible; status line shows ShimmerCard animation; grid is immediately interactive (tapping a card navigates before shimmer resolves)
- **Error (summary API fails):** Status lines show "—"; cards are still tappable; no error state blocks the grid
- **Module has no data:** Card renders with appropriate "nothing logged yet" copy — never a blank status line

### References

None — standard component library behavior applies.

### Interaction Notes

- Tapping a ModuleCard navigates immediately — do not wait for summary data to load
- Pull-to-refresh: `.refreshable { await viewModel.load() }` — reloads all 5 summaries
- Routines card: if the grid has an odd number of cards (5 total), the last card left-aligns in a 2-column grid — use `LazyVGrid` which handles this automatically (no special case needed)

---

## For Coding Agents

Read this file alongside `S-005-001-dashboard-grid.md` and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; never show a blank status line — always use the fallback copy
3. **States** — implement default, loading, and error states; the grid is never fully blocked
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

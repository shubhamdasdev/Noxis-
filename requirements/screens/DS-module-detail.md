# Design Spec — Screen: Module Detail

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-004
**Screen:** Screen: Module Detail
**Stories:** S-005-002, S-005-003, S-005-004, S-005-005, S-005-006, S-005-007
**Modes:** wardrobe, gym, food, spending, routines
**Components:** NavigationBar, GlassCard, GlassButton, GlassInput, GlassSheet, ShimmerCard, EmptyState, ToggleRow, Toast
**Updated:** 2026-03-18

---

## Screen: Module Detail

**Story context:** Per-module data view. Each mode shows auto-extracted + manually logged data for one life area. The user browses entries and can add new ones via a floating action button.

### Shared Layout (all modes)

1. Primary: Module data list/grid — fills scroll view; auto-populated from chat extraction and manual entries
2. Secondary: Module name as NavigationBar title + back button
3. Tertiary: Floating "Add" button — bottom-right, above tab bar

### Shared Components

| Element | Component | Notes |
|---------|-----------|-------|
| Navigation title | Text (Headline, 20pt Semibold, white) | Module name; back chevron left |
| Data content area | Mode-specific (see below) | |
| Empty state | EmptyState | Icon + headline + "Add [item type]" GlassButton |
| Loading state | `LazyVStack` of ShimmerCard rows | Full-height shimmer cards while data fetches |
| Floating Add button | `ZStack` overlay — GlassButton (compact, Accent border) | Bottom-right corner; 24pt from screen edge and tab bar; icon `plus` + label |
| Add entry sheet | GlassSheet | `.sheet` presentation; module-specific fields |
| Error Toast | Toast (error variant) | Network failures; API write failures with rollback |
| Pull-to-refresh | `.refreshable` on ScrollView | Reloads module data |

### Floating Add Button Labels Per Mode

| Mode | Button label |
|------|-------------|
| wardrobe | Add item |
| gym | Log workout |
| food | Log meal |
| spending | Log purchase |
| routines | Add habit |

---

## Mode: Wardrobe

**Layout:** Two-tab layout: "Inventory" tab (3-column photo grid) + "Outfits" tab (vertical outfit log)

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Tab switcher | SwiftUI `Picker(style: .segmented)` | "Inventory" / "Outfits"; glass-styled |
| Inventory grid | `LazyVGrid(columns: [GridItem(.flexible()), GridItem(.flexible()), GridItem(.flexible())])` | 3 columns; 4pt gap |
| Item card | GlassCard (default) | Square; image thumbnail fill (or placeholder with `tshirt` icon); category label (Caption Semibold) bottom; notes truncated (Caption) below category |
| Outfit row | `HStack` — thumbnail (52×52pt) + rating phrase (Body) + date (Caption, Text secondary, trailing) | Glass border divider below each row |
| Item detail screen | Full-screen push: full image + notes + date added + swipe-to-delete | Uses standard `NavigationLink` push |

### Add Item Sheet Fields

| Field | Component | Notes |
|-------|-----------|-------|
| Photo | `PHPickerViewController` + camera option | Photo required; full-width square thumbnail preview after selection |
| Category | `Picker` (segmented or wheel) | T-Shirt / Pants / Shoes / Jacket / Accessory / Other |
| Notes | GlassInput (multiline) | Optional; max 140 chars; character counter Caption at bottom-right |
| Save CTA | GlassButton (full-width) | Disabled until photo + category selected |
| Cancel | Text button (Caption, Accent) | Dismisses sheet |

### Copy

| Element | Copy |
|---------|------|
| Tab 1 | Inventory |
| Tab 2 | Outfits |
| Empty (inventory) | Your wardrobe is empty. |
| Empty (outfits) | No outfits rated yet. |
| Add sheet title | Add item |
| Notes placeholder | Notes (optional) |

---

## Mode: Gym

**Layout:** Streak counter (prominent) + chronological workout log (vertical list)

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Streak counter | GlassCard (elevated) full-width | Large number (Display, 48pt, Accent) + "day streak" (Caption, Text secondary) below |
| Workout row | `HStack` — workout type badge (Caption Semibold, Accent background chip) + date (Caption, Text secondary) + duration (Caption, Text secondary, trailing) | Notes line if present (Caption, Text secondary, below type/date row); Glass border divider |

### Add Workout Sheet Fields

| Field | Component | Notes |
|-------|-----------|-------|
| Workout type | `Picker` | Strength / Cardio / Mobility / Sport / Other |
| Date | `DatePicker` | Defaults to today |
| Duration | GlassInput (standard, numeric keyboard) | "minutes" label suffix |
| Notes | GlassInput (multiline) | Optional |
| Save CTA | GlassButton (full-width) | Active when type selected |

### Copy

| Element | Copy |
|---------|------|
| Streak label suffix | day streak |
| Streak zero state | Start your streak today. |
| Empty (log) | No workouts logged yet. |
| Add sheet title | Log workout |
| Duration placeholder | Duration (minutes) |

---

## Mode: Food

**Layout:** Meal count summary (top) + chronological meal log

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Weekly summary | GlassCard (elevated) full-width | "N meals this week" (Body Semibold) |
| Meal row | `VStack` — meal description (Body) + meal type badge (Caption Semibold) + date (Caption, Text secondary) + optional Noxis quality flag (Caption, Text secondary, italic) | Thumbnail if image present (44×44pt, leading); Glass border divider |

### Add Meal Sheet Fields

| Field | Component | Notes |
|-------|-----------|-------|
| Meal description | GlassInput (standard) | Required |
| Meal type | `Picker` (segmented) | Breakfast / Lunch / Dinner / Snack |
| Date/time | `DatePicker` | Defaults to now |
| Notes | GlassInput (multiline) | Optional |
| Save CTA | GlassButton (full-width) | Active when description filled |

### Copy

| Element | Copy |
|---------|------|
| Weekly summary | N meals this week |
| Empty | Nothing logged yet. |
| Add sheet title | Log meal |
| Description placeholder | What did you eat? |

---

## Mode: Spending

**Layout:** Weekly total (top, Accent color) + chronological purchase list

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Weekly total | GlassCard (elevated) full-width | "$N.NN this week" — currency formatted; amount in Accent color (Display, 28pt) + "this week" Caption below |
| Purchase row | `HStack` — item name (Body) + category badge (Caption Semibold) + amount (Body Semibold, trailing) + date (Caption, Text secondary, below amount) | Quality tag (Caption, Text secondary, italic) if present; Glass border divider |

### Add Purchase Sheet Fields

| Field | Component | Notes |
|-------|-----------|-------|
| Item name | GlassInput (standard) | Required |
| Amount | GlassInput (standard, numeric keyboard with decimal) | "$" prefix label |
| Category | `Picker` | Clothing / Food / Fitness / Entertainment / Other |
| Date | `DatePicker` | Defaults to today |
| Save CTA | GlassButton (full-width) | Active when item name and amount filled |

### Copy

| Element | Copy |
|---------|------|
| Weekly total suffix | this week |
| Empty | Nothing tracked yet. |
| Add sheet title | Log purchase |
| Item placeholder | What did you buy? |
| Amount placeholder | 0.00 |

---

## Mode: Routines

**Layout:** Active habits list with today-completion toggles + streak counts

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Habit row | ToggleRow | Habit name (Body) + streak count (Caption Semibold, Accent, e.g. "12-day streak") + toggle right; `Glass border` divider |
| Inactive habits section | `LazyVStack` with "Inactive" section header (Caption Semibold, Text secondary) | Habits with no completions in the last 7 days; toggle still works |

### Add Habit Sheet Fields

| Field | Component | Notes |
|-------|-----------|-------|
| Habit name | GlassInput (standard) | Required |
| Frequency | `Picker` | Daily / Weekdays / Custom |
| Target time | `DatePicker` (time only) | Optional; "No reminder" option |
| Save CTA | GlassButton (full-width) | Active when name filled |

### Copy

| Element | Copy |
|---------|------|
| Streak suffix | -day streak |
| First completion | Started today. |
| Empty | No routines set up yet. |
| Add sheet title | Add habit |
| Name placeholder | Habit name |
| Inactive section header | Inactive |

---

## Shared Entry Detail Screen

Accessed by tapping any item row or card in any mode. Uses `NavigationLink` push.

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Full image (wardrobe/food) | `AsyncImage` full-width, aspect fit | Tap → `.fullScreenCover` full-screen preview |
| Full notes / description | Text (Body, white) | All content fully expanded |
| Date added | Text (Caption, Text secondary) | |
| Noxis feedback (if present) | GlassCard (default) | Noxis's rating or quality flag, Body text |
| Delete | `.swipeActions(edge: .trailing)` with `Button(role: .destructive)` | Label: "Delete"; triggers `.confirmationDialog` |
| Confirmation dialog | `.confirmationDialog` | "Delete [item type]?" with "Delete" (destructive) + "Cancel" |

### Copy (Entry Detail)

| Element | Copy |
|---------|------|
| Delete swipe label | Delete |
| Confirmation title | Delete this entry? |
| Confirmation destructive | Delete |
| Confirmation cancel | Cancel |

---

## For Coding Agents

Read this file alongside relevant S-005-NNN story files and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table for all labels, empty states, and sheet titles
3. **States** — each mode needs: default (data loaded), loading (ShimmerCard), empty (EmptyState + Add CTA), and error (Toast with rollback for failed writes)
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# Design Spec — Screen: Quick Profile

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-001
**Screen:** Screen: Quick Profile
**Stories:** S-001-002
**Components:** PillSelector, GlassButton
**Updated:** 2026-03-18

---

## Screen: Quick Profile

**Story context:** User seeds basic context in under 30 seconds — three visual pill-selector questions with no text input required. Skippable at any point.

### Layout & Hierarchy

1. Primary: Three question blocks — stacked vertically with 24pt gap between blocks; 16pt horizontal padding
2. Secondary: Screen header + subtext — centered above first question
3. Tertiary: Continue button — pinned near bottom; active at all times (even with zero selections)
4. Quaternary: Skip link — 12pt below Continue button, centered

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Screen header | Text (Headline, white) | Left-aligned |
| Subtext | Text (Caption, Text secondary) | Left-aligned, 4pt below header |
| Q1 label | Text (Body Semibold, white) | "What do you most want to level up?" |
| Q1 pills | PillSelector (horizontal, wrapping) | 4 pills; single-select per group |
| Q2 label | Text (Body Semibold, white) | "Do you train?" |
| Q2 pills | PillSelector (horizontal, wrapping) | 4 pills; single-select |
| Q3 label | Text (Body Semibold, white) | "Do you cook?" |
| Q3 pills | PillSelector (horizontal) | 3 pills; single-select |
| Continue CTA | GlassButton (full-width) | Always active — zero selections is valid |
| Skip link | Text (Caption, Accent color) | Text button, no border, no background |

### Copy

| Element | Copy |
|---------|------|
| Screen header | Quick setup |
| Subtext | Takes 20 seconds. Skip anytime. |
| Q1 label | What do you most want to level up? |
| Q1 option 1 | Style |
| Q1 option 2 | Fitness |
| Q1 option 3 | Daily Routine |
| Q1 option 4 | All of the above |
| Q2 label | Do you train? |
| Q2 option 1 | Don't train |
| Q2 option 2 | 1–2×/week |
| Q2 option 3 | 3–4×/week |
| Q2 option 4 | 5+ days/week |
| Q3 label | Do you cook? |
| Q3 option 1 | Never — I order everything |
| Q3 option 2 | Sometimes |
| Q3 option 3 | Regularly |
| Continue CTA | Continue |
| Skip link | Skip |

### States

- **Default:** All pills unselected; Continue active.
- **Pill selected:** Pill shows PillSelector selected state (Accent border, Surface elevated fill); all other pills in the same question group revert to unselected.
- **Back navigation (returning):** Previously selected pills restored from `OnboardingState`.

### References

None — standard component library behavior applies.

### Interaction Notes

- Pills wrap to multiple lines if they don't fit on one row — use `FlowLayout` or `LazyVGrid` with flexible item sizing
- "Never — I order everything" pill is wider than the others; it may sit on its own row on narrow screens — acceptable
- Skip and Continue navigate to the same destination (Screen: Authentication)

---

## For Coding Agents

Read this file alongside `S-001-002-quick-profile-screen.md` and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; do not paraphrase button labels or error messages
3. **States** — implement all states listed for every screen
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

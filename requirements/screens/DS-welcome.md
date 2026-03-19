# Design Spec — Screen: Welcome

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-001
**Screen:** Screen: Welcome
**Stories:** S-001-001, S-007-004
**Components:** NavigationBar, GlassButton
**Updated:** 2026-03-18

---

## Screen: Welcome

**Story context:** The user's absolute first frame in the app — brand impression and single CTA to begin onboarding.

### Layout & Hierarchy

1. Primary: Noxis wordmark — centered horizontally and vertically (with slight upward offset, roughly 40% from top)
2. Secondary: Tagline — centered, 8pt below wordmark
3. Tertiary: "Get Started" button — centered, pinned near bottom (24pt above home indicator safe area)

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Wordmark | Text (Display, 28pt Bold, white) | "Noxis" — not a logo image in v1; use `Text("Noxis").font(.system(size: 28, weight: .bold, design: .default))` |
| Tagline | Text (Body, Text secondary) | Rendered below wordmark |
| Primary CTA | GlassButton (full-width) | Bottom-pinned; max-width: screen − 32pt horizontal padding |

### Copy

| Element | Copy |
|---------|------|
| Wordmark | Noxis |
| Tagline | Your standards, elevated. |
| Primary CTA | Get Started |

### States

- **Default:** Pure black background; wordmark and tagline visible; "Get Started" active.

### References

None — standard component library behavior applies.

### Interaction Notes

- No back navigation from this screen during onboarding — this is the funnel entry point
- "Get Started" navigates to Screen: Tone Selection; no animation delay on tap

---

## For Coding Agents

Read this file alongside `S-001-001-welcome-tone-selection.md` and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; do not paraphrase button labels or error messages
3. **States** — implement all states listed for every screen
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

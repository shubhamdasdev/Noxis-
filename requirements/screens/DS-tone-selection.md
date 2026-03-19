# Design Spec — Screen: Tone Selection

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-001
**Screen:** Screen: Tone Selection
**Stories:** S-001-001, S-002-002
**Components:** ToneCard, GlassButton
**Updated:** 2026-03-18

---

## Screen: Tone Selection

**Story context:** User picks how Noxis talks to them — this selection shapes every response they will ever receive. It must feel like a meaningful choice, not a preference setting.

### Layout & Hierarchy

1. Primary: Three ToneCards — stacked vertically, full-width, 16pt horizontal padding, 12pt gap between cards
2. Secondary: Header question — centered above cards, 24pt below navigation bar
3. Tertiary: "Continue" button — pinned near bottom (24pt above home indicator safe area); disabled until a card is selected
4. Quaternary: Footnote — 8pt below Continue button, centered, Caption

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Screen header | Text (Headline, white) | Centered |
| Brother card | ToneCard | See content below |
| Consultant card | ToneCard | See content below |
| Peer card | ToneCard | See content below |
| Continue CTA | GlassButton (full-width) | Disabled state until selection made; `opacity(0.4)` + glass border when disabled |
| Footnote | Text (Caption, Text secondary) | Centered below Continue |

### ToneCard Content

Each ToneCard contains three text layers:

| Card | Mode name | Description | Sample (User:) | Sample (Noxis:) |
|------|-----------|-------------|----------------|-----------------|
| Brother | Brother | Sharp, direct, no padding | Rate my outfit | That hoodie is doing nothing for you. Swap it for the bomber — you have 3 minutes. |
| Consultant | Consultant | Precise, structured | Rate my outfit | The hoodie adds bulk without structure. Your bomber would shift this to casual-sharp. |
| Peer | Peer | Honest, casual | Rate my outfit | Honest take — the bomber you wore last week hit way harder. Switch it in. |

Sample exchange rendered as a two-line block inside the card: "User: [text]" (Caption, Text secondary) then "Noxis: [text]" (Caption, Text secondary, slight left indent or a small Noxis label prefix). Wrapped in a GlassCard elevated chip inside the ToneCard.

### Copy

| Element | Copy |
|---------|------|
| Screen header | How should I talk to you? |
| Continue CTA | Continue |
| Footnote | You can change this anytime in settings. |

### States

- **Default:** All three cards unselected; Continue disabled.
- **Card selected:** Selected ToneCard shows GlassCard (selected) state — Accent glow border + outer glow shadow; other cards return to default; Continue becomes active.
- **Back navigation (returning):** Previously selected card is highlighted on re-render — selection is in `OnboardingState`.

### References

None — ToneCard behavior fully defined in design.md component inventory.

### Interaction Notes

- Tapping a selected card does NOT deselect it — selection persists; user must choose a different card to change selection
- Continue button is completely non-interactive when disabled — no tap feedback, no error
- Scroll enabled if cards overflow screen height (iPhone SE)

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

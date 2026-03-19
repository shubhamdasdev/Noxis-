# Design Spec — Overlay: Side Panel

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-002
**Screen:** Overlay: Side Panel
**Stories:** S-007-002, S-004-004
**Components:** SidePanel, GlassButton, EmptyState
**Updated:** 2026-03-18

---

## Overlay: Side Panel

**Story context:** Conversation history and module quick-jumps — accessible from the Chat screen via swipe or hamburger tap. The user's control center without leaving chat.

### Layout & Hierarchy

1. Primary: Conversation list — fills the bulk of the panel; scrollable; grouped by date
2. Secondary: Header (wordmark + close) + New Chat button — pinned to top
3. Tertiary: Modules section + 5 quick-jump buttons — pinned to bottom OR below the conversation list (scroll reveals it)
4. Dimmed overlay: 40% black on the right side of the screen while panel is open

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Panel container | SidePanel | 80% screen width; spring slide-in from left |
| Dimmed right overlay | `Color.black.opacity(0.4)` | Tappable to dismiss; full screen height |
| Panel header | NavigationBar (condensed) | Left: nothing; Center: "Noxis" (Display, 28pt); Right: × button (`xmark`, 22pt, Text secondary, 44pt hit area) |
| New Chat button | GlassButton (full-width) | Below header, 12pt vertical margin |
| Section divider | `Divider()` with `Glass border` color | Between New Chat and conversation list |
| Conversation date group header | Text (Caption Semibold, Text secondary) | Left-aligned; values: "Today", "Yesterday", "This Week", "Older" |
| Conversation row | `HStack` — truncated first message (Body, Text primary) + timestamp (Caption, Text secondary, trailing) | Max 40 chars then "…"; 52pt row height; `Glass border` divider below |
| Empty state (no history) | EmptyState | No icon; headline only |
| Modules section header | Text (Caption Semibold, Text secondary) | "Modules" label; left-aligned |
| Module quick-jump buttons | `HStack` of 5 icon+label buttons | SF Symbol icon (20pt) + label (Caption); arranged in a horizontal row; each 44pt tall; Text secondary default; Text primary on tap |

### Module Quick-Jump Icons

| Module | SF Symbol |
|--------|-----------|
| Wardrobe | `tshirt` |
| Gym | `figure.strengthtraining.traditional` |
| Food | `fork.knife` |
| Spending | `creditcard` |
| Routines | `checkmark.seal` |

### Copy

| Element | Copy |
|---------|------|
| Panel wordmark | Noxis |
| New Chat button | New Chat |
| Empty state headline | Your conversations will appear here. |
| Modules section header | Modules |

### States

- **Default (with history):** Panel slides in; conversations grouped and listed; modules row at bottom
- **Empty (no conversations):** EmptyState renders in the list area; "New Chat" button still visible; modules row still visible
- **List loading:** ShimmerCard rows in place of conversation rows while thread list fetches; resolves in < 500ms in normal conditions

### References

- ChatGPT iOS side panel: grouping by date (Today / Yesterday / This Week / Older), New Chat button position, × dismiss — take the information architecture pattern

### Interaction Notes

- Panel opens with spring animation: `stiffness: 300, damping: 30`
- Dismiss methods: swipe-left on panel, tap dimmed overlay, tap × button — all three must work
- Tapping a conversation: panel closes (slide-out) simultaneously as Chat loads the selected thread — no intermediate loading state
- Tapping a module quick-jump: panel closes; app switches to Modules tab; NavigationStack pushes to that module's detail screen
- Tapping New Chat: panel closes; Chat clears to empty state; new thread starts

---

## For Coding Agents

Read this file alongside `S-007-002-side-panel.md` and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; do not paraphrase button labels or error messages
3. **States** — implement default, empty, and loading states
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

# Design Spec — Screen: Daily Tab

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-003
**Screen:** Screen: Daily Tab
**Stories:** S-006-002, S-006-006, S-006-003
**Components:** NavigationBar, TodayBriefCard, BriefHistoryRow, ShimmerCard, EmptyState, Toast
**Updated:** 2026-03-18

---

## Screen: Daily Tab

**Story context:** The morning ritual surface — user reads their personalized daily brief, optionally browses past briefs. Context-aware routing lands the user here automatically on first open before 10am if an unread brief exists.

### Layout & Hierarchy

1. Primary: Today Brief Card — top of scroll view; full-width; prominent; largest element on screen
2. Secondary: Brief History List — below Today card; chronological, newest-first; collapses to one-line rows
3. Screen title "Daily" — large title in NavigationBar (Display typography)
4. Pull-to-refresh over the entire ScrollView

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Navigation title | Text (Display, 28pt) in `navigationTitle` | "Daily" — large title style (scrolls up with content) |
| Today Brief Card | TodayBriefCard | Full-width; 16pt horizontal padding; present at top of scroll view |
| Brief loading shimmer | ShimmerCard | Same dimensions as TodayBriefCard; shown when `status: pending` |
| No brief today state | EmptyState | In place of TodayBriefCard only |
| Brief History list | `LazyVStack` of BriefHistoryRow | Below TodayBriefCard; 16pt top margin; no section header needed |
| Pull-to-refresh | `.refreshable` modifier on ScrollView | Calls `viewModel.load()` |
| Network error state | EmptyState (error variant) | Replaces both TodayBriefCard and history list |

### TodayBriefCard Structure (per design.md)

- Date header: "Today — [Day, Month Date]" (e.g. "Today — Wednesday, 18 March") · Caption · Text secondary
- Insight paragraph · Body · Text primary
- Action items: 3 max · Each has a 3pt × 24pt Accent-color vertical bar left-decoration · Body text · 8pt gap between items
- Curated content chip (if present): GlassCard elevated · title (Caption Semibold) + type badge (Caption Semibold, Accent) + description (Caption, Text secondary) · full-width inside brief card · tappable

### BriefHistoryRow Structure

- Collapsed: date string (Caption, Text secondary, left) + truncated first sentence (Caption, Text primary, fills remaining width) + unread dot (6pt Accent circle, left edge, if `viewed_at == nil`)
- Expanded: full TodayBriefCard content layout (minus date header — date is already in the row header)
- Expand/collapse: spring animation `withAnimation(.spring(response: 0.3))`

### Copy

| Element | Copy |
|---------|------|
| Screen title | Daily |
| No brief today headline | No brief today |
| No brief today subtext | Keep using Noxis and tomorrow's brief will reflect your activity. |
| Load error headline | Couldn't load your brief |
| Load error subtext | Pull to refresh |
| Brief status failed | Brief unavailable today — Noxis will try again tomorrow. |

### States

- **Default (brief generated):** TodayBriefCard renders with full content; history list below; `viewed_at` set on first `onAppear` if null
- **Loading (status: pending):** ShimmerCard of TodayBriefCard dimensions; history list may or may not be available separately — show shimmer rows for history too if fetching
- **No brief today:** EmptyState in TodayBriefCard slot with specific copy; history list still renders normally below
- **Brief generation failed (status: failed):** Brief card shows "Brief unavailable today — Noxis will try again tomorrow." — no error code, no icon, plain Body text in TodayBriefCard container
- **Error (API failure):** Full-screen EmptyState (error variant) replaces both card and history list; "Pull to refresh" is the sole recovery action

### References

None — standard component library behavior applies.

### Interaction Notes

- `viewed_at` PATCH fires in `onAppear` of TodayBriefCard if `brief.viewed_at == nil` — call `PATCH /api/v1/briefs/{id}/viewed`; no UI change on this call; silent failure is acceptable
- Curated content chip tap: present `SFSafariViewController` as `.sheet`; dismiss with Done button in Safari toolbar
- BriefHistoryRow accordion: `expandedId` state in parent view; only one row expanded at a time; tapping expanded row collapses it
- Entire screen (TodayBriefCard + history) scrolls together — TodayBriefCard is NOT sticky

---

## For Coding Agents

Read this file alongside `S-006-002-brief-screen.md`, `S-006-006-curated-content.md`, and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; do not paraphrase button labels or error messages
3. **States** — implement all five states (default, loading, no-brief, failed, error) listed above
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

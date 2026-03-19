# Design Brief — Noxis

**Version:** 1.0
**Updated:** 2026-03-18
**Project:** noxis

---

## Design Philosophy

| # | Principle | Derived from |
|---|-----------|-------------|
| 1 | Every screen surfaces one clear action the user can take right now — no "explore the app" moments | The Invisible Man: "doesn't know where to start; no honest feedback loop" |
| 2 | Premium is earned before words are spoken — the glass-and-black system communicates the product's standards from the first frame | Brief constraint: "must feel premium and private"; all three personas care about visible standards |
| 3 | Speed is respect — the AI response starts streaming within 1.5s; never block the interface with a full-screen spinner | North Star: "honest judgment in under 10 seconds"; DAU/MAU 40% target requires zero friction in the daily loop |
| 4 | Context-awareness eliminates navigation burden — the app routes itself based on time, brief status, and user state | The Rebuilder: "overwhelmed by where to begin"; context-aware entry (S-006-003) removes the decision |
| 5 | Data before decoration — every module shows meaningful content or a specific action prompt; empty placeholder dashboards are never shown | The Upgrader: "scattered tools that don't connect; no single system that knows his standards" |

---

## Visual Language

> Stack: Swift + SwiftUI (iOS 17+). No NativeWind or Tailwind. All tokens are defined explicitly here and implemented in `DesignSystem/DesignSystem.swift`.

### Palette

| Token | Value | SwiftUI usage | Notes |
|-------|-------|---------------|-------|
| Background | `#000000` | `Color.black` | Pure black — entire app background |
| Surface (glass) | `rgba(255,255,255,0.05)` | `Color.white.opacity(0.05)` | Glass card fill — always paired with glass border |
| Glass border | `rgba(255,255,255,0.10)` | `Color.white.opacity(0.10)` | Stroke on all GlassCard and GlassInput components |
| Surface elevated | `rgba(255,255,255,0.08)` | `Color.white.opacity(0.08)` | Slightly brighter glass for pressed/active states |
| Text primary | `#FFFFFF` | `Color.white` | All primary text |
| Text secondary | `rgba(255,255,255,0.55)` | `Color.white.opacity(0.55)` | Muted labels, captions, subtext, placeholders |
| Accent | `#3B82F6` | `Color(hex: "3B82F6")` | Glow borders, selection indicators, accent dots, action item bars |
| Accent glow | `#3B82F6` at 40% opacity | — | Outer shadow on selected ToneCard; blur radius 12 |
| Destructive | `#EF4444` | `Color(hex: "EF4444")` | Delete actions, error states |
| Tab bar background | `UIBlurEffect(style: .systemUltraThinMaterialDark)` | `UIVisualEffectView` | Custom tab bar blur — not a SwiftUI modifier |
| Side panel background | `UIBlurEffect(style: .systemThinMaterialDark)` | `UIVisualEffectView` | Side panel blur |

### Typography

> All fonts: SF Pro (system default on iOS 17+). No custom typefaces. Sizes in points (pt).

| Token | Font | Size | Weight | Usage |
|-------|------|------|--------|-------|
| Display | SF Pro Display | 28pt | Bold | Noxis wordmark, screen large titles (Daily tab) |
| Headline | SF Pro Display | 20pt | Semibold | Screen section headers, module names, tone card titles |
| Body | SF Pro Text | 16pt | Regular | Chat messages, brief insight, story body copy |
| Body Semibold | SF Pro Text | 16pt | Semibold | Button labels, active tab labels |
| Caption | SF Pro Text | 12pt | Regular | Timestamps, status summaries, muted subtext, footnotes |
| Caption Semibold | SF Pro Text | 12pt | Semibold | Type badges (meal, spend category) |

### Spacing & Shape

| Token | Value | Notes |
|-------|-------|-------|
| Spacing unit | 4pt base | All spacing in multiples of 4 |
| Screen horizontal padding | 16pt | Standard inset from screen edges |
| Card corner radius | 16pt | All GlassCard instances |
| Button corner radius | 12pt | GlassButton |
| Input corner radius | 12pt | GlassInput |
| Pill corner radius | 20pt | PillSelector (fully rounded) |
| Touch target minimum | 44×44pt | All interactive elements |

---

## Component Library & Framework

| Concern | Decision | Notes |
|---------|----------|-------|
| Styling framework | SwiftUI native ViewModifiers + `DesignSystem.swift` token constants | No NativeWind, no third-party styling library; all tokens in `DesignSystem.swift` |
| Component library | Custom glass component library in `DesignSystem/Components/` | `GlassCard`, `GlassButton`, `GlassInput`, etc. — all custom; no RN Paper or equivalent |
| Icon set | SF Symbols (system) | iOS-native, no third-party icon pack; specific symbol names listed per component below |
| Animation library | SwiftUI `.animation()` + `withAnimation()` for simple transitions; `UIKitBridge` for gesture-driven drawer | No external animation library; Reanimated-equivalent is not applicable (SwiftUI) |
| Camera / media | `UIImagePickerController` / `PHPickerViewController` (iOS 17) | Native camera and photo library access |
| In-app browser | `SFSafariViewController` | Curated content links in Daily brief |

### Component Inventory

Every component listed here corresponds to a file in `DesignSystem/Components/`. DS screen specs reference these names exactly.

#### GlassCard
- Fill: `Surface (glass)` · Border: `Glass border` · Corner radius: 16pt · Shadow: none by default
- Variants: `default` | `elevated` (Surface elevated fill) | `selected` (Accent glow border + outer glow shadow, blur 12)
- Use for: tone selection cards, message bubbles, module cards, brief card, add entry sheets, empty states

#### GlassButton
- Fill: `Surface (glass)` · Border: `Accent` (1pt) · Corner radius: 12pt · Label: Body Semibold · Text: white
- States: `default` | `disabled` (opacity 0.4, accent border fades to glass border) | `loading` (spinner replaces label)
- Variants: `full-width` | `compact` (min-width: label + 32pt horizontal padding)
- Use for: all primary CTAs ("Get Started", "Continue", "Send", "Create account", "Save")

#### GlassInput
- Fill: `Surface (glass)` · Border: `Glass border` · Corner radius: 12pt · Text: Body, white · Placeholder: Text secondary
- Focus state: border upgrades to `Accent` (1pt)
- Variants: `standard` (single line) | `secure` (password, eye toggle right) | `multiline` (min 3 lines, grows to 6)
- Use for: email, password, notes fields, chat input bar

#### AppleSignInButton
- Component: native `ASAuthorizationAppleIDButton(type: .default, style: .black)`
- Do not re-skin — use native Apple styling exactly
- Height: 50pt · Corner radius: 12pt (`.cornerRadius(12)` wrapper)

#### ToneCard
- Base: `GlassCard (default)` · Full-width with 16pt horizontal padding · Vertical stack layout
- Selected state: `GlassCard (selected)` — Accent glow border + shadow
- Structure (top to bottom): Mode name (Headline) → Description (Body, Text secondary) → Sample exchange block (Caption, Text secondary background chip)
- Tap: toggle selected state on this card; deselect any other card

#### PillSelector
- Height: 36pt · Horizontal padding: 16pt · Corner radius: 20pt (fully rounded)
- Unselected: fill `Surface (glass)`, border `Glass border`, label Caption, Text secondary
- Selected: fill `Surface elevated`, border `Accent` (1pt), label Caption, Text primary
- Only one pill selected per question group (radio behavior)

#### BottomTabBar
- Height: 83pt (including safe area) · Background: `Tab bar background` (UIBlurEffect ultraThin dark)
- Top edge: hairline separator `Glass border` (0.5pt)
- Tab items (left to right): Chat (`bubble.left.fill`) · Modules (`square.grid.2x2.fill`) · **Camera** (center, elevated) · Daily (`sun.max.fill`) · Profile (`person.fill`)
- Active tab: icon + label, white, 1pt scale-up · Inactive tab: icon + label, Text secondary
- **Camera button (center):** 56pt diameter circle · fill `Accent` · icon `camera.fill` white 22pt · elevated 8pt above bar baseline · no label · no tab-selected state — it is an action trigger

#### SidePanel
- Width: 80% of screen width · Background: `Side panel background` (UIBlurEffect thin dark) · Border-right: `Glass border`
- Slides in from left with spring animation (`stiffness: 300, damping: 30`)
- Dimmed overlay right side: black 40% opacity
- Structure (top to bottom): Header (Noxis wordmark + × button) → New Chat button (GlassButton compact) → Conversation list → "Modules" section label → Module quick-jump row (5 buttons)
- Dismiss: swipe-left gesture OR tap dimmed overlay OR × button
- Gesture zone: 20pt from left edge of Chat screen triggers open

#### NavigationBar
- Height: 44pt (+ status bar safe area) · Background: transparent (content overlays black)
- Left slot: hamburger icon (`line.3.horizontal`, 22pt, white) — Chat screen only; back chevron on drill-down screens
- Center slot: "Noxis" wordmark (Display, 28pt, white) — or screen title on non-root screens
- Right slot: reserved (empty in v1)
- No default iOS navigation bar chrome — fully custom

#### MessageBubble
- **User message:** GlassCard (default), right-aligned, max-width 75% screen · Body text · Sent timestamp Caption below (Text secondary)
- **Image message (user):** same container, thumbnail fills card at 16:9 or square aspect; tap → full-screen preview
- **Noxis message:** no bubble background, left-aligned, max-width 85% screen · Body text · no timestamp
- Spacing between consecutive messages: 8pt · Spacing between turns: 16pt

#### TypingIndicator
- Three dots, Accent color, pulse animation (sequential opacity: 0.2 → 1.0 → 0.2, staggered 200ms)
- Appears left-aligned in message list, same position as a Noxis MessageBubble
- Shows when pipeline is running; replaced by first StreamingText token on arrival

#### StreamingText
- Text renders word-by-word (SSE token delivery) left-aligned, Body typography, no bubble
- Cursor: vertical bar `|` (2pt wide, Accent color) appended to last word while streaming, disappears on completion
- No animation on the text itself — tokens appear immediately as received

#### ModuleCard
- Base: GlassCard (default) · Aspect ratio: 1:1 (square) · Padding: 16pt
- Structure: SF Symbol icon (28pt, white, top-left) → Module name (Headline, white) → Status summary (Caption, Text secondary)
- Loading state: ShimmerCard of same dimensions replaces status line only
- Tap: NavigationLink push to Module Detail

#### TodayBriefCard
- Base: GlassCard (default) · Full-width · Vertical padding: 20pt
- Structure (top to bottom): Date header (Caption, Text secondary) → Insight paragraph (Body) → Action items list (Body, left-accent bar in Accent color 3pt wide) → Optional curated content chip (GlassCard elevated, Caption)
- Action items: max 3 bullets; each item has a 3pt × 24pt Accent-color vertical bar as left decoration
- Curated content chip: title (Caption Semibold) + type badge (Caption Semibold) + short description (Caption); tappable → SFSafariViewController

#### BriefHistoryRow
- Height: 52pt (collapsed) · Expands to full content height (accordion with spring animation)
- Left edge: unread indicator — 6pt Accent circle when `viewed_at == nil`; invisible (no reserved space) when read
- Content: date (Caption, Text secondary, left) + first sentence truncated (Caption, Text primary) + unread dot
- Expanded state: full insight + action items + curated chip, same layout as TodayBriefCard minus date header
- Divider between rows: `Glass border` hairline

#### ShimmerCard
- Same dimensions as target card · Background: Surface (glass) · Animated gradient overlay:
  - Gradient: `Surface elevated` → `Surface (glass)` → `Surface elevated`, 45° angle, sliding left-to-right on 1.2s loop
- Use for: TodayBriefCard loading, ModuleCard status line loading

#### GlassSheet
- Bottom sheet modal (`.sheet` SwiftUI modifier)
- Background: `Side panel background` (UIBlurEffect thin dark) with rounded top corners 20pt
- Handle indicator: 4pt × 36pt pill, `Glass border`, centered top
- Dismiss: drag down or tap overlay
- Use for: Add Entry forms (all modules), photo picker alternative

#### ToggleRow
- Height: 52pt · Background: transparent
- Structure: habit name (Body) + streak count (Caption Semibold, Accent) → toggle (right)
- Toggle: standard iOS `Toggle` with `.tint(Color(hex: "3B82F6"))`
- Completion: optimistic UI — toggle state updates immediately, streak increments; reverts with Toast on server error
- Divider between rows: `Glass border` hairline

#### EmptyState
- Centered vertically in available space · SF Symbol icon (40pt, Text secondary) → headline (Body Semibold, Text primary) → optional CTA (GlassButton compact)
- Use for: empty chat thread ("Ask me anything."), no conversation history ("Your conversations will appear here"), no module data, no brief today

#### Toast
- Bottom of screen, 24pt above tab bar · GlassCard (default), compact, max-width: screen − 32pt
- Auto-dismiss after 3s OR tap to dismiss · Entry animation: slide-up from bottom + fade-in · Exit: fade-out
- Variants: `neutral` (Text primary) | `error` (Destructive text) | `retry` (includes "Tap to retry" link)
- Use for: network errors, optimistic UI rollbacks, session expiry notices

---

## Navigation Patterns

| Concern | Decision | Notes |
|---------|----------|-------|
| Primary navigation | Custom 5-tab BottomTabBar (Chat · Modules · Camera · Daily · Profile) | Always visible on all primary screens; hidden inside GlassSheet and on onboarding screens before auth |
| Screen transitions | `NavigationStack` push (standard right-to-left) for drill-downs | Module detail, entry detail, profile sub-screens |
| Onboarding flow | `NavigationStack` with custom back behavior — no back button on Welcome and Tone Selection | Onboarding flow is a one-way funnel |
| Side panel | Slide-in drawer from left edge — not a navigation stack push | SidePanel component; only accessible from Chat screen |
| Modal presentation | `GlassSheet` (`.sheet`) for add-entry forms and picker variants | Full-screen modal not used in v1 |
| Camera | `.fullScreenCover` presenting `UIImagePickerController` | Triggered by center tab button or chat image icon |
| In-app browser | `SFSafariViewController` presented as sheet | Curated content in Daily brief |
| Back navigation | `NavigationStack` system back gesture + back chevron in NavigationBar left slot | No custom back handling needed |
| Deep linking | Not required in v1 | Push notification taps → Daily tab, handled by AppRouter |

---

## Authentication Gates

All onboarding screens before auth (Welcome, Tone Selection, Quick Profile) are accessible without an account. Every other surface and action requires authentication.

| Action | Auth Required | Rationale |
|--------|:---:|-----------|
| View Welcome screen | No | Entry screen — no account yet |
| Select tone mode | No | Pre-auth onboarding step |
| Complete Quick Profile | No | Pre-auth onboarding step |
| Register / Sign In | No | This is the auth action itself |
| Send a chat message | Yes | Requires user identity for memory and thread storage |
| View conversation history | Yes | Personal data |
| Send an image | Yes | Requires user identity; stored to Supabase Storage |
| View daily brief | Yes | Personal data generated from user's history |
| View module dashboard | Yes | Personal data |
| Log a module entry (any module) | Yes | Transactional — writes to user's data store |
| Mark habit complete | Yes | Transactional — updates streak |
| Delete any entry or thread | Yes | Destructive transactional action |
| View profile settings | Yes | Personal settings and account management |
| Change tone mode | Yes | Modifies user account settings |

On any gated action with an expired or missing JWT: silent token refresh attempt → if refresh fails, route to Welcome screen with Toast: "Your session expired. Please sign in again."

---

## Key UX Patterns

### Empty States

Appears in: UF-001 (chat onboarding), UF-002 (new thread, no history), UF-003 (no brief today, no history), UF-004 (no module data).

- Every surface that can be empty has an EmptyState component — never a blank screen
- EmptyState always tells the user what to do next, not just what's missing
- No illustrations in v1 — SF Symbol icon (Text secondary) + one-line directive (Body Semibold, Text primary) + optional GlassButton CTA
- Example: chat empty → "Ask me anything." (no CTA, no icon — the cursor is the affordance); wardrobe empty → icon `tshirt` + "Your wardrobe is empty." + "Add item" (GlassButton compact)

### Loading & Skeletons

Appears in: all 4 flows (pipeline, brief generation, module data, auth check).

- **List / card data loading:** ShimmerCard in place of the expected card — same dimensions, shimmer animation, no spinner
- **AI response in progress:** TypingIndicator replaces the next message slot; replaced by StreamingText on first token; never block input bar — user can continue typing while response is streaming
- **Full-screen async (splash auth check):** Noxis wordmark on black — no spinner, no progress indicator; resolves in < 500ms by design (S-007-004)
- **Optimistic writes (habit toggle, add entry):** UI updates immediately; Toast on failure with rollback

### Error Handling

Appears in: all 4 flows (API failures, auth errors, network failures, pipeline failures).

- **Inline form errors:** red text (Destructive) below the relevant field, Body size; no modal alerts
- **Network / server errors (non-blocking):** Toast with retry tap action; user is never redirected away from what they were doing
- **Module data load failure:** module cards still render (navigation still works); status line shows "—" dash instead of data
- **Chat pipeline failure:** inline message in chat thread: "Something went wrong. Tap to retry." — conversation history preserved; no redirect
- **Session expiry:** route to Welcome screen with Toast: "Your session expired. Please sign in again." — only redirect in the app

### Confirmation Dialogs

Appears in: UF-002 (delete thread), UF-004 (delete entry — swipe action in both).

- Destructive actions (delete thread, delete module entry) use a SwiftUI `.confirmationDialog` (action sheet style), not a full modal
- Dialog options: `Delete` (destructive, red) + `Cancel`
- No confirmation for reversible actions (habit toggle, mark viewed) — optimistic with rollback Toast

### Photo & Camera

Appears in: UF-001 (onboarding style photo), UF-002 (outfit/meal/body check), UF-004 (wardrobe item photo).

- Two entry points always provided: camera capture (new photo) + photo library (existing photo) — never only one
- Center tab bar camera button → `UIImagePickerController` (camera mode) → photo returned to Chat
- Chat image icon → `PHPickerViewController` (library, single select) → photo returned to Chat
- Wardrobe Add Item → `PHPickerViewController` + camera option in same sheet
- Camera permission denied → iOS permission dialog → if denied, in-app prompt: "Camera access lets Noxis see your outfits and meals. Enable in Settings." with Settings deep link
- Compress all images to < 2MB before Supabase upload (matches arch.md NFR)

---

## Accessibility

| Concern | Standard | Notes |
|---------|---------|-------|
| Color contrast | WCAG AA (4.5:1 normal text, 3:1 large text) | White text on black background passes by wide margin; glass surface elements must be checked; Accent `#3B82F6` on black = 3.6:1 — acceptable for large text / icons; use white for small text in accent contexts |
| Touch targets | 44×44pt minimum (iOS HIG) | All tab bar items, buttons, and toggle rows explicitly sized; icon-only buttons (hamburger, × button) padded to 44pt hit area |
| Screen reader labels | All interactive elements labeled | `accessibilityLabel` required on: camera button, hamburger, × close, unread indicator dots, tone card selection state, habit toggle, curated content tap target |
| Dynamic text | Support up to 200% system font scale | Use SwiftUI `.font(.body)` style modifiers (not fixed size) where layout allows; GlassCard height grows with content |
| Motion | Respect `UIAccessibility.isReduceMotionEnabled` | Skip shimmer animation and streaming cursor blink when reduce motion is on; side panel and sheet transitions use fade instead of slide |

---

## Tone & Voice

> Derived from SOUL.md philosophy (direct, specific, non-moralizing), brief persona language, and the "value-based realism" framing of Dr. Orion Tarban's framework.

| Rule | Correct | Wrong |
|------|---------|-------|
| Second-person, active voice — the user is always the subject | "Your brief is ready." | "A brief has been generated." |
| Error messages are facts, not apologies — state what happened and what to do | "Couldn't save — tap to retry." | "We're sorry, something went wrong. Please try again later." |
| CTAs say exactly what happens — no vague verbs | "Add item" · "Log workout" · "Delete" · "Try again" | "Submit" · "Continue" · "OK" · "Proceed" |
| Empty states end with action, not absence | "Ask me anything." · "Add your first item." | "No messages yet." · "Your wardrobe is empty." |
| Specificity over hedge — name the thing | "Wardrobe" · "Workout log" · "Morning brief" | "Your items" · "Your data" · "Your content" |
| No filler affirmations — skip the congratulations | (no equivalent — just proceed) | "Great choice!" · "You're all set!" · "Success!" |

---

## For Coding Agents

Before implementing any UI component:

1. **Check the component library** — use exactly the components listed in this brief; do not introduce new UI dependencies; every visual unit has a named component above
2. **Check auth gates** — if the action is gated, redirect to Welcome via `AppRouter` before allowing it; gated screens must check JWT validity on `onAppear` (not just on launch)
3. **Check UX patterns** — if the screen has a list, implement the empty state; if it has an async action, use ShimmerCard or TypingIndicator (not a spinner); chat pipeline failures get an inline retry message, not a Toast
4. **Check accessibility** — every interactive element needs an `accessibilityLabel`; touch targets must be 44×44pt minimum; test with VoiceOver and reduce-motion on

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created — full design brief from brief.md, arch.md, 4 flows, all stories |

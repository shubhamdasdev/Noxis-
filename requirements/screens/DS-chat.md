# Design Spec — Screen: Chat

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-001, UF-002
**Screen:** Screen: Chat
**Stories:** S-004-001, S-004-002, S-004-006, S-004-005, S-001-003, S-001-004
**Modes:** onboarding, normal
**Components:** NavigationBar, MessageBubble, GlassInput, GlassButton, TypingIndicator, StreamingText, EmptyState, Toast
**Updated:** 2026-03-18

---

## Screen: Chat

**Story context:** The primary interaction surface — where the user talks to Noxis. Covers onboarding mode (Noxis leads the conversation) and normal mode (open-ended chat). Both modes use identical UI; only the Noxis message content changes.

### Layout & Hierarchy

1. Primary: Message list — fills all available space between NavigationBar and input bar; scrollable; auto-scrolls to bottom on new message
2. Secondary: Input bar — pinned to bottom, above keyboard (uses `.ignoresSafeArea(.keyboard, edges: .bottom)` with `safeAreaInset`)
3. Tertiary: NavigationBar — top of screen; transparent background (overlays the black app background)

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Navigation bar | NavigationBar | Left: hamburger (`line.3.horizontal`); Center: "Noxis" wordmark (Display, 28pt) |
| Message list | `ScrollViewReader` + `LazyVStack` | Auto-scroll to bottom `.id("bottom")` on new message; padding: 16pt horizontal, 8pt vertical between messages |
| User message bubble | MessageBubble (user variant) | Right-aligned; glass card; max-width 75%; timestamp Caption below |
| User image message | MessageBubble (image variant) | Same container; image thumbnail 1:1 or 16:9 aspect; tap → `.fullScreenCover` full-screen preview |
| Noxis message | MessageBubble (Noxis variant) | No bubble; left-aligned; max-width 85%; no timestamp |
| Typing indicator | TypingIndicator | Renders in same position as Noxis MessageBubble; shown from pipeline start until first token |
| Streaming text | StreamingText | Replaces TypingIndicator on first SSE token; cursor disappears on completion |
| Empty state | EmptyState | Shown when thread has no messages |
| Input text field | GlassInput (standard) | Left inset: image icon; right inset: Send button; placeholder: Caption Text secondary |
| Image icon (input) | SF Symbol `photo.on.rectangle` | Left of text field; 22pt; Text secondary color; 44pt hit area; opens `PHPickerViewController` |
| Send button | GlassButton (compact) | Right of text field; active only when text field is non-empty; icon `arrow.up` |

### Copy

| Element | Copy |
|---------|------|
| Wordmark (nav) | Noxis |
| Empty state headline | Ask me anything. |
| Input placeholder | Message |
| Network error (inline) | Something went wrong. Tap to retry. |

### Modes

**Onboarding mode** (`isOnboarding == true`):
- UI is identical to normal mode — no onboarding UI chrome, no progress indicators
- Noxis sends the opening message automatically on `onAppear` (within 2s) via the pipeline
- Completion signal (3+ answers or "let's go") fires `FirstValueDelivery` — still rendered as a standard Noxis MessageBubble
- `isOnboarding` flag is invisible to the user; screen looks the same throughout

**Normal mode** (`isOnboarding == false`):
- Standard open-ended chat
- User taps hamburger → SidePanel slides in
- All message types available (text, image)

### States

- **Default (empty thread):** EmptyState rendered; input bar visible and active; no messages
- **Loading (pipeline running):** TypingIndicator appears; input bar remains active (user can type next message); send button disabled until current stream completes
- **Streaming:** StreamingText renders token-by-token; TypingIndicator gone; input bar re-activates immediately after stream ends
- **Error (pipeline failure):** Inline error message in place of Noxis response bubble: "Something went wrong. Tap to retry." — Body, Text secondary; tapping the message retries the last request
- **Image message loading:** Thumbnail shows ShimmerCard shimmer while uploading; replaced by actual thumbnail on upload completion

### References

- ChatGPT iOS app: side panel swipe gesture, message list layout, input bar position above keyboard — take the layout pattern, not the visual styling

### Interaction Notes

- Input bar uses `safeAreaInset(edge: .bottom)` to stay above the keyboard without screen reflowing
- Auto-scroll to bottom when a new message is added; user can scroll up freely to read history; auto-scroll resumes when user is at bottom (within 60pt)
- Swipe right from left 20pt edge → SidePanel opens (only when not inside a `.sheet`)
- Image tap → `.fullScreenCover` presenting `AsyncImage` full-screen with a close button (× top-right); background black

---

## Widget: Streaming Response

**Story context:** Noxis's response token-by-token display — the "alive" feeling of the product.

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Response text | StreamingText | Left-aligned, no bubble; tokens appended on SSE arrival |
| Cursor | Inline text `|` (Accent color, 2pt wide) | Appended after last token; removed when stream ends; hidden if `reduceMotion` is enabled |

### States

- **Streaming:** Text builds word-by-word; cursor blinks at 600ms interval
- **Complete:** Full response text rendered; cursor removed; input bar re-activates; async memory + dashboard extraction fires silently
- **Interrupted:** Partial response shown; "Continue" tappable label (Caption, Accent) appended at end of partial text

---

## For Coding Agents

Read this file alongside relevant story files and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; do not paraphrase button labels or error messages
3. **States** — implement all four states (default, loading, streaming, error) for the message list
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

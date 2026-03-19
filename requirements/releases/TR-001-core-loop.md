# Release Test Plan: R-001 Core Loop

---

**ID:** TR-001
**Release:** R-001
**Project:** noxis
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Release Goal

Deliver a fully functional end-to-end Noxis experience — onboarding with tone selection, personalised AI chat with persistent memory, image-based outfit/meal checks, wardrobe and gym module tracking, and a daily morning brief — proving the core value proposition for early beta users.

---

## Coverage Summary

| Story | Title | Test Plan | Cases | Gap |
|-------|-------|-----------|-------|-----|
| S-007-003 | Glass + Black Design System | T-007-003 ✓ | 5 | — |
| S-007-001 | Bottom Tab Bar | T-007-001 ✓ | 6 | — |
| S-007-004 | App Launch & Splash Screen | T-007-004 ✓ | 5 | — |
| S-001-001 | Welcome & Tone Selection | T-001-001 ✓ | 4 | — |
| S-001-002 | Quick Profile Screen | T-001-002 ✓ | 4 | — |
| S-001-003 | Conversational Onboarding | T-001-003 ✓ | 5 | — |
| S-001-004 | First Value Delivery | T-001-004 ✓ | 4 | — |
| S-001-005 | Authentication | T-001-005 ✓ | 10 | — |
| S-002-001 | Load SOUL.md as System Prompt | T-002-001 ✓ | 5 | — |
| S-002-002 | Tone Mode Selection & Switching | T-002-002 ✓ | 5 | — |
| S-002-003 | Module-Aware Response Routing | T-002-003 ✓ | 5 | — |
| S-002-004 | Judgment Framework per Module | T-002-004 ✓ | 4 | — |
| S-002-005 | Boundary Enforcement | T-002-005 ✓ | 5 | — |
| S-003-001 | Memory Storage Layer (Mem0) | T-003-001 ✓ | 7 | — |
| S-003-002 | USER.md Auto-Generation | T-003-002 ✓ | 7 | — |
| S-003-003 | Memory Extraction from Chat | T-003-003 ✓ | 7 | — |
| S-003-004 | Decision Policy — Filter & Rank | T-003-004 ✓ | 7 | — |
| S-004-001 | Chat Interface — Send & Receive | T-004-001 ✓ | 7 | — |
| S-004-002 | Image Input — Camera & Photo Send | T-004-002 ✓ | 8 | — |
| S-004-003 | Image Classification & Routing | T-004-003 ✓ | 8 | — |
| S-004-005 | Orchestration Pipeline | T-004-005 ✓ | 7 | — |
| S-004-006 | Streaming Response Display | T-004-006 ✓ | 6 | — |
| S-005-001 | Dashboard Grid | T-005-001 ✓ | 6 | — |
| S-005-002 | Wardrobe Module | T-005-002 ✓ | 8 | — |
| S-005-003 | Gym Module | T-005-003 ✓ | 8 | — |
| S-005-007 | Chat-to-Dashboard Extraction | T-005-007 ✓ | 9 | — |
| S-006-001 | Daily Brief Generation Engine | T-006-001 ✓ | 7 | — |
| S-006-002 | Daily Brief Screen | T-006-002 ✓ | 7 | — |
| S-006-004 | Push Notification for Brief | T-006-004 ✓ | 9 | — |

**Total:** 29 stories · 29 test plans · ~185 test cases · 0 gaps

---

## Integration Test Cases

Cross-story E2E scenarios that can only be executed once all dependent stories are `Status: Done` and deployed to the test environment.

---

### ITC-001-001: Cold Launch → Full Onboarding → First Value Delivery

**Stories Covered:** S-007-004, S-007-001, S-007-003, S-001-001, S-001-002, S-001-005, S-001-003, S-001-004, S-002-001, S-002-002, S-003-002
**Priority:** P0

**Preconditions:**
- Fresh app install — no existing account, no JWT in Keychain
- All listed stories are Done and deployed to test environment
- Backend identity layer (AT-001-003) is Done

**Steps:**
1. Cold-launch the app on a physical iPhone (iOS 17+)
2. Confirm Splash screen appears — pure black, Noxis wordmark, no spinner
3. Confirm app routes to Welcome screen (not Chat — user is unauthenticated)
4. Tap "Get Started" → Tone Selection screen appears with 3 ToneCards
5. Tap "Consultant" card → accent glow border appears, Continue activates
6. Tap "Continue" → Quick Profile screen appears
7. Tap "Fitness" for Q1, "3–4×/week" for Q2, "Sometimes" for Q3
8. Tap "Continue" → Authentication screen appears with header "Save your setup"
9. Enter a unique email and a password (8+ characters), tap "Create account"
10. Confirm navigation to Chat screen in Onboarding Mode
11. Wait up to 2 seconds — confirm Noxis sends an opening message automatically referencing fitness
12. Reply to 3 of Noxis's questions with substantive answers
13. After the 3rd answer, confirm Noxis sends a First Value Message without the user prompting
14. Confirm the First Value Message contains a specific, actionable fitness recommendation (not generic advice)
15. Confirm the follow-up: "What's your reaction? And feel free to ask me anything — that's what I'm here for."
16. Background the app, relaunch — confirm Chat screen loads directly (no onboarding restart)

**Expected Result:**
User arrives in Chat — Normal Mode with the first value message visible in conversation history. Onboarding does not restart on relaunch. USER.md is seeded in Mem0 with tone (consultant) and fitness context.

**Failure Indicators:**
- App routes to Chat instead of Welcome on cold launch as unauthenticated user
- Continue button active before any ToneCard is selected
- Opening message does not reference the Quick Profile selections (fitness, 3–4×/week)
- First Value Message is generic (e.g., "work out more") rather than specific
- Onboarding restarts on relaunch

---

### ITC-001-002: Chat Message → Orchestration → Streaming → Memory Extraction

**Stories Covered:** S-004-001, S-004-005, S-004-006, S-002-001, S-002-002, S-003-001, S-003-003, S-003-004
**Priority:** P0

**Preconditions:**
- Authenticated user with onboarding complete
- At least one prior memory entry exists in Mem0 for this user (seeded during ITC-001-001)
- Backend is running; Mem0 and OpenAI connections active

**Steps:**
1. Open the app, confirm Chat screen loads in Normal Mode
2. Type: "I'm thinking about switching from a 4-day PPL split to a 5-day programme. What do you think?" and tap Send
3. Confirm user message bubble appears immediately (right-aligned, glass card)
4. Confirm TypingIndicator (three animated dots) appears within 1.5 seconds
5. Confirm TypingIndicator is replaced by StreamingText (token-by-token) when the first token arrives
6. Confirm cursor blink at end of incomplete text; disappears when response completes
7. Read the response — confirm it references the user's existing fitness context (e.g., current split, frequency) rather than asking for it again
8. Send a second message: "Actually what's my current weekly volume?" — confirm response uses stored memory without asking "what do you currently do?"
9. Wait 5 seconds after both responses complete — verify no error states or crashes

**Expected Result:**
Both responses use stored memory. SOUL.md personality is audible (direct, specific, non-moralizing). Streaming cursor works correctly. Input bar is active immediately after each response completes. New memory entries are extracted async (verifiable in Mem0 dashboard or admin API).

**Failure Indicators:**
- Response does not begin streaming within 2 seconds of Send
- Response asks for information already provided in onboarding (memory not injected)
- Streaming text renders all at once instead of token-by-token
- Input bar stays disabled after stream completes
- Response uses generic advice unrelated to user's stated fitness context

---

### ITC-001-003: Camera Button → Image Capture → Classification → Module-Aware Response

**Stories Covered:** S-007-001, S-004-002, S-004-003, S-004-005, S-002-003, S-002-004
**Priority:** P1

**Preconditions:**
- Authenticated user with onboarding complete
- Camera permission granted
- Physical device with rear camera (not simulator)

**Steps:**
1. From any tab, tap the center Camera button in the tab bar
2. Confirm native iOS camera opens immediately in photo mode
3. Photograph a clothing item (e.g., a shirt laid flat)
4. Confirm photo and tap the confirmation control
5. Confirm app returns to Chat tab; image appears as a right-aligned message bubble with thumbnail
6. Confirm TypingIndicator appears within 1.5 seconds
7. Confirm response appears via StreamingText
8. Read the response — confirm it provides wardrobe-specific judgment (not gym or food framing)
9. Repeat with a photo of a meal — confirm response uses food module framing

**Expected Result:**
Wardrobe photo → wardrobe judgment (style, fit, color). Food photo → food judgment (quality, nutritional awareness, delivery vs. home-cooked framing). Classification is transparent to the user — module context appears in the response framing, not as a label.

**Failure Indicators:**
- Camera does not open immediately (intermediate screen shown)
- Image not routed to Chat after capture
- Response uses generic advice unrelated to the image content
- Same response framing regardless of image type (classification not working)
- Response asks "what is this a photo of?" (classification failure fallback visible to user)

---

### ITC-001-004: Chat Wardrobe Mention → Dashboard Extraction → Wardrobe Module

**Stories Covered:** S-004-001, S-004-005, S-005-007, S-005-001, S-005-002, S-003-003
**Priority:** P1

**Preconditions:**
- Authenticated user with onboarding complete
- Wardrobe module is empty (no prior items)

**Steps:**
1. Open Chat; send: "I just picked up a black bomber jacket from COS. It's a medium, fits well."
2. Confirm Noxis responds with wardrobe-relevant judgment or acknowledgment
3. Wait 5 seconds for async extraction to complete
4. Tap the Modules tab → Dashboard Grid loads
5. Confirm Wardrobe card status line shows "1 item in your wardrobe" (not "No items yet.")
6. Tap the Wardrobe card → Wardrobe module detail opens
7. Confirm a wardrobe item entry exists (black bomber jacket / jacket category / source: chat)
8. Repeat with a gym mention in chat: "Trained legs today — squats and RDLs, about 60 minutes"
9. Navigate to Modules → Gym → confirm a workout log entry exists (legs / source: chat)

**Expected Result:**
Items mentioned in chat appear in the relevant module without the user manually adding them. `source: chat` is set on extracted records. Module status lines update to reflect the new data.

**Failure Indicators:**
- Wardrobe card still shows "No items yet." after chat mention and 5-second wait
- Extracted item has wrong category or is missing
- Gym workout log has no entry despite chat mention
- Module navigation crashes or shows loading state indefinitely

---

### ITC-001-005: Daily Brief Generation → Brief Screen → Push Notification

**Stories Covered:** S-006-001, S-006-002, S-006-004, S-003-001, S-003-003
**Priority:** P1

**Preconditions:**
- Authenticated user with 3+ days of chat history and at least 10 memory entries (establishes context for the brief)
- Server cron has run (or brief is manually triggered via admin API for testing)
- Push notification permission granted on device

**Steps:**
1. Trigger brief generation for the test user (via admin endpoint or wait for cron)
2. Confirm `daily_briefs` record exists in Supabase with `status: generated` and `viewed_at: null`
3. Confirm push notification arrives on device: "Your morning brief is ready."
4. Tap the notification — confirm app opens directly to Daily tab
5. Confirm Today Brief Card renders with: date header, insight paragraph, 3 action items with left-accent bars
6. Confirm `viewed_at` is set on the record after the card renders (check Supabase)
7. Scroll down — confirm Brief History list is present (or empty state if no prior briefs)
8. Tap curated content card (if present) — confirm SFSafariViewController opens the URL in-app
9. Relaunch the app — confirm Daily tab does not show an unread indicator (brief already viewed)

**Expected Result:**
Brief is specific to this user's recent activity and memory. Insight references real patterns (not generic advice). Action items are 3 or fewer. `viewed_at` is set on first render. Push notification routes correctly to Daily tab.

**Failure Indicators:**
- Brief content is generic and does not reference any stored user context
- `viewed_at` remains null after viewing the brief
- Push notification opens the app but not the Daily tab
- Brief action items exceed 3
- Curated content tap opens the link in an external browser instead of in-app

---

### ITC-001-006: Returning User — JWT Persists, Skips Onboarding, Opens to Chat

**Stories Covered:** S-007-004, S-001-005, S-004-001
**Priority:** P1

**Preconditions:**
- User completed ITC-001-001 (has an account, onboarding complete, valid JWT in Keychain)
- Current time is after 10am (to ensure Chat tab routing, not Daily tab)

**Steps:**
1. Force-quit the app
2. Relaunch the app
3. Confirm Splash screen appears briefly
4. Confirm app navigates directly to Chat screen (not Welcome, not Tone Selection, not onboarding)
5. Confirm conversation history from the previous session is visible
6. Confirm tab bar is visible with all 5 tabs

**Expected Result:**
App skips all onboarding screens and opens to Chat with history intact. No sign-in prompt shown. JWT is valid and silently accepted.

**Failure Indicators:**
- App shows Welcome screen despite valid JWT
- App shows any onboarding screen
- Chat loads but conversation history is empty
- Tab bar missing or shows incorrect active tab

---

## Go / No-Go Criteria

All of the following must be true before R-001 ships:

- [ ] All P0 Core Test Flow cases pass across all 29 story test plans
- [ ] All P1 sub-flow cases pass across all 29 story test plans
- [ ] All 6 integration test cases (ITC-001-001 through ITC-001-006) pass on a physical iPhone (iOS 17+)
- [ ] No open P0 or P1 defects without an accepted workaround documented
- [ ] All 29 stories are `Status: Done`
- [ ] Architecture Gate is Open (all 4 ATs are `Stage: Done`)
- [ ] App builds with zero errors and zero warnings (`xcodebuild` clean build)
- [ ] Load test: 50 concurrent users sending chat messages — first token latency remains < 3 seconds at p95 (reduced from 500 target; 50 is sufficient for beta)
- [ ] No PII (JWT, email, memory content) logged to console or crash reports in production build
- [ ] Apple Sign-In tested on physical device (not simulator — ASAuthorizationController is broken in simulator)

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Mem0 API latency spikes under load | Medium | High | Decision policy has 500ms timeout; fallback to USER.md only is implemented in S-003-004 |
| OpenAI GPT-4.1 rate limits during beta | Low | High | Implement retry with exponential backoff in AT-001-004; alert on 429 responses |
| APNs certificate not configured for push in staging | High | Medium | AT-001-002 includes APNs setup; test push in staging before marking ITC-001-005 |
| Apple Sign-In breaks in simulator | High (known) | Low | ITC-001-001 specifies physical device; simulator testing uses email/password path only |
| SOUL.md personality drift if model changes | Low | High | SHA-256 hash logging per call (arch.md NFR) catches drift; pin model ID in config |
| Image compression fails on specific iPhone models | Low | Medium | S-004-002 specifies 3-pass compression loop; test on iPhone SE (smallest) and iPhone 17 Pro Max |
| `viewed_at` PATCH fails silently in poor network | Medium | Low | ITC-001-005 checks Supabase directly; brief unread state resets gracefully on next open |
| Supabase free tier connection pool exhaustion | Medium | Medium | Monitor concurrent connections in Supabase dashboard; upgrade to Pro before beta launch |

---

## Known Gaps

- **Performance testing** — load test target reduced to 50 concurrent users for beta (arch.md specifies 500; full load test deferred to R-002 pre-production)
- **Android** — out of scope for R-001; iOS only
- **Side panel (S-007-002)** — deferred to R-002; no integration test for conversation history browsing in this release
- **Profile tab (S-007-005)** — deferred to R-002; tone switching post-onboarding is not covered by ITC cases here
- **P2 test cases** — should pass before release but are not blocking go/no-go if documented exceptions exist with PM sign-off

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created — 29 stories, 6 integration test cases, full go/no-go criteria |

# QA Test Plan — USER.md Auto-Generation from Onboarding

---

**Plan ID:** T-003-002
**Story:** S-003-002
**Epic:** E-003
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates that USER.md is generated correctly from onboarding data immediately after account creation. Testing covers: timely generation after account creation, accurate mapping of all onboarding answers to USER.md sections, `unset` handling for missing fields, First Impressions paragraph generation in Noxis's voice, USER.md context injection into the first conversation, USER.md field update on high-confidence memory extraction, partial onboarding data handling, First Impressions generation failure recovery, duplicate onboarding (double-completion) merge behavior, and contradiction handling in onboarding answers.

## Out of Scope

- Showing USER.md to the user in the UI
- The memory review screen (S-003-007)
- Automatic memory extraction from ongoing chat (S-003-003) — this story covers initial generation only
- User-visible profile summary screen (future scope)
- USER.md diff view in admin/debug panel (future scope)

## Prerequisites

- S-001-003 (conversational onboarding) is complete — onboarding answers must exist in the database
- S-001-005 (authentication) is complete — user account must exist to attach the profile
- S-003-001 (memory storage) is complete — initial profile facts are seeded to Mem0 after generation
- `generateUserProfile(userId)` backend function is implemented
- `user_profiles` table exists with `content`, `structured_data`, `version`, `created_at`, `updated_at` columns
- OrchestrationPipeline USER.md injection is implemented

---

## Core Test Flow

### TC-003-002-001: Complete onboarding, verify USER.md generated within 5s, all sections correct, First Impressions present, pipeline injects on first message
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006, AC-007
**Dependencies:** S-001-003, S-001-005, S-003-001

**Preconditions:**
- A fresh test user account does not yet exist
- Full onboarding data is prepared: Name, age range, city, tone selection ("Brother"), gym routine (5 days/week, goal: cut), style (slim fit, minimal aesthetic), food (clean diet, cooks 4x/week), spending (unstated), goals ("gym consistency", "wardrobe upgrade")

**Steps:**
1. Complete onboarding end-to-end with the prepared data set; complete authentication (S-001-005)
2. Record the timestamp of account creation
3. Query the `user_profiles` table for the test user; note the `created_at` timestamp on the USER.md record
4. Assert the USER.md was created within 5 seconds of account creation
5. Inspect `user_profiles.content` (the Markdown string) for the following:
   - `## Tone Preference` contains "brother"
   - `## Physical` contains gym frequency "5 days/week" and goal "cut"
   - `## Style` contains "slim fit" and "minimal"
   - `## Food` contains "clean" and "cooks 4x/week"
   - `## Spending` contains "unset" (not blank, not omitted)
   - `## Goals` contains "gym consistency" and "wardrobe upgrade"
6. Inspect `## First Impressions` — assert it contains a non-empty paragraph written in Noxis's voice (direct, specific to the user's data — not generic)
7. Send the first real message after onboarding: "So what do you think of where I'm at?"
8. Intercept the outbound AI provider request payload; assert USER.md content appears between the tone instruction layer and the module context layer in the prompt
9. Inspect the response — assert it reflects onboarding context (e.g., references cutting goal or gym frequency) without the user restating it
10. Verify that `MemoryService.addMemory` was called for each non-`unset` USER.md field with `confidence: 1.0` and `source: "onboarding"` — check Mem0 for these seeded memories

**Expected Result:**
- Step 4: `user_profiles.created_at` is within 5 seconds of account creation timestamp
- Step 5: All stated fields correctly mapped; `## Spending` shows "unset"
- Step 6: `## First Impressions` has 2-3 sentences specific to the user's onboarding answers, in Noxis's voice
- Step 8: USER.md content present in the payload at the correct layer (after tone, before module context)
- Step 9: Response references the user's actual onboarding data; user does not need to re-explain
- Step 10: Mem0 contains seeded memories for all non-`unset` onboarding fields with `confidence: 1.0` and `source: "onboarding"`

**Failure Indicators:**
- USER.md created more than 5 seconds after account creation
- Any stated onboarding answer not reflected in the correct USER.md section
- `## Spending` is blank or omitted rather than "unset"
- `## First Impressions` is empty or generic ("The user wants to improve")
- USER.md content absent from the AI request payload
- First message response requires the user to re-explain onboarding data
- Mem0 not seeded with onboarding memories

---

## Sub Flows

### TC-003-002-002: Partial onboarding — skipped fields marked `unset`, profile usable
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-001-003, S-001-005, S-003-001

**Preconditions:**
- Test user completed onboarding but skipped the spending, food, and style screens

**Steps:**
1. Complete onboarding with spending, food, and style screens skipped
2. Authenticate and allow `generateUserProfile` to run
3. Inspect `user_profiles.content`

**Expected Result:**
- `## Spending`, `## Food`, and `## Style` sections all contain "unset"
- No section is omitted from the document
- Sections with data (e.g., `## Tone Preference`, `## Physical`, `## Goals`) are correctly populated
- Profile is usable — USER.md is stored and injected into the first conversation despite partial data

**Failure Indicators:**
- Skipped sections are missing from USER.md entirely
- Skipped sections are blank (empty string) rather than "unset"
- `generateUserProfile` fails or does not run on partial data

---

### TC-003-002-003: First Impressions generation fails — USER.md stored with `pending_generation`, retried on next app open
**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR coverage — negative scenario)
**Dependencies:** S-001-003, S-001-005

**Preconditions:**
- AI provider is configured to fail for the First Impressions generation call only (use a test shim or force token limit to 0 for that specific call)

**Steps:**
1. Complete onboarding
2. Allow `generateUserProfile` to run with the AI failure active
3. Inspect `user_profiles.content` for the `## First Impressions` section
4. Restore the AI provider; simulate the next app open (trigger the retry mechanism)
5. Re-inspect `user_profiles.content`

**Expected Result:**
- Step 3: `## First Impressions` contains "pending_generation" (not blank, not omitted)
- All other USER.md sections are correctly populated from onboarding data
- USER.md is stored and usable despite the partial failure
- Step 5: After retry, `## First Impressions` contains a valid paragraph in Noxis's voice

**Failure Indicators:**
- `generateUserProfile` fails entirely when the AI call for First Impressions fails
- `## First Impressions` is blank or omitted rather than "pending_generation"
- Retry does not run on next app open
- Other USER.md sections are affected by the First Impressions failure

---

### TC-003-002-004: USER.md field update from high-confidence memory extraction
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-007
**Dependencies:** S-001-003, S-001-005, S-003-001, S-003-003

**Preconditions:**
- Test user has an existing USER.md with `## Physical` gym frequency set to "3 days/week"
- `UserProfileUpdater` listener is implemented

**Steps:**
1. Trigger a `MemoryService.addMemory` call with `content: "User now trains 6 days/week"`, `category: "gym"`, `confidence: 0.90`, `source: "chat"`
2. Allow the `MemoryAdded` event and `UserProfileUpdater` listener to run
3. Inspect `user_profiles.content` and `user_profiles.structured_data`

**Expected Result:**
- `## Physical` gym frequency updated to "6 days/week"
- The old value "3 days/week" is stored as a previous value with a timestamp in `structured_data`
- `user_profiles.updated_at` is refreshed
- No other USER.md sections are affected

**Failure Indicators:**
- Gym frequency field not updated after high-confidence memory extraction
- Old value not retained with a timestamp
- Other USER.md sections are unintentionally modified
- Confidence below 0.85 triggers an update (threshold not enforced)

---

### TC-003-002-005: Low-confidence memory — USER.md field NOT updated
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — threshold enforcement)
**Dependencies:** S-003-001

**Preconditions:**
- Test user has an existing USER.md with `## Physical` gym frequency set to "3 days/week"

**Steps:**
1. Trigger a `MemoryService.addMemory` call with `content: "User might train 5 days/week"`, `category: "gym"`, `confidence: 0.80` (below the 0.85 threshold)
2. Allow the `UserProfileUpdater` to evaluate
3. Inspect `user_profiles.content`

**Expected Result:**
- `## Physical` gym frequency remains "3 days/week" — not updated
- Memory is stored in Mem0 at 0.80 confidence but does not trigger a USER.md field update

**Failure Indicators:**
- USER.md field updated despite confidence below threshold
- Memory is rejected (not stored in Mem0)

---

### TC-003-002-006: Double onboarding completion — merge, not overwrite
**Type:** Edge Case
**Priority:** P1
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-001-003, S-001-005

**Preconditions:**
- Test user has an existing USER.md from a first onboarding run (spending is `unset`, gym frequency is "4 days/week")
- A second onboarding run is simulated (e.g., user force-quit and restarted; spending is now stated as "mid-range", gym frequency not re-stated)

**Steps:**
1. Trigger a second `generateUserProfile` run for the same user
2. Inspect `user_profiles.content`

**Expected Result:**
- `## Spending` is updated from "unset" to "mid-range" (new data fills in the gap)
- `## Physical` gym frequency remains "4 days/week" (confirmed field not overwritten with `unset`)
- No data loss from the first onboarding run
- `user_profiles.version` incremented

**Failure Indicators:**
- Second run overwrites the entire USER.md, losing the gym frequency
- Spending remains "unset" despite the new answer
- Duplicate `user_profiles` records created instead of a merge

---

### TC-003-002-007: Contradictory onboarding answers — both recorded, First Impressions acknowledges tension
**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR coverage — edge case)
**Dependencies:** S-001-003, S-001-005

**Preconditions:**
- Onboarding data includes two contradictory statements: diet type = "clean" (from quick profile) and "I order food daily" (from conversational answers)

**Steps:**
1. Complete onboarding with the contradictory data set
2. Allow `generateUserProfile` to run
3. Inspect `user_profiles.content`

**Expected Result:**
- `## Food` section records both facts (diet type noted as "clean" with a note referencing the stated delivery frequency)
- `## First Impressions` paragraph acknowledges the apparent tension between stated diet goal and actual behavior
- No data is silently dropped to resolve the contradiction

**Failure Indicators:**
- Only one of the contradictory facts is recorded (the other is silently dropped)
- `## First Impressions` does not acknowledge the tension
- `generateUserProfile` fails when encountering contradictions

---

## Automation Notes
- **Timing assertion (AC-001):** use a server-side event timestamp on account creation and `user_profiles.created_at`; assert delta is < 5000ms; run in a controlled environment without load to avoid flakiness
- **USER.md content assertion:** parse the Markdown sections using a section parser fixture; assert field values programmatically rather than string-matching the full document
- **First Impressions quality check:** use keyword presence (user's name, stated goal, gym frequency, or style preference) to assert specificity; reject empty or generic paragraphs
- **Mem0 seeding verification:** after generation, call `listMemories` for the test user with `source: "onboarding"` filter; assert the count matches the number of non-`unset` fields
- **Retry mechanism test:** use a test flag `FIRST_IMPRESSIONS_FAIL=true` in the backend environment to force the AI call failure deterministically
- **Flakiness risk:** First Impressions paragraph content is non-deterministic; assert on structure and keyword presence, not exact text

---

## Change Log
| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

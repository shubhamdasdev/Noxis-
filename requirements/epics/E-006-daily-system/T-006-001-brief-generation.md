# QA Test Plan — Daily Brief Generation Engine

---

**Plan ID:** T-006-001
**Story:** S-006-001
**Epic:** E-006
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the server-side `DailyBriefGenerator` cron job: correct triggering at the user's configured `brief_delivery_time`, data assembly from session logs and module summaries, AI-generated brief content that references specific data points, storage of the brief with `status: generated` in the `daily_briefs` table, deduplication (one brief per user per day), the activity gate (no brief for zero-activity users), and push notification dispatch (S-006-004) within the same job run. Error paths cover session log fetch failure, AI generation failure with retry at +15 minutes, and timezone DST handling.

## Out of Scope

- Brief display screen (S-006-002) — tested in T-006-002
- Push notification delivery mechanics (S-006-004) — tested in T-006-004
- Brief personalization evolution, weekend/weekday format variation, brief quality scoring, team briefs (future scope)

## Prerequisites

- S-003-006 (session logs) is implemented and the `session_logs` table contains data for test users
- S-003-001 (memory storage) is implemented; USER.md goals section is populated for test user A
- S-005-001 (dashboard grid) module state data is accessible via `ModuleSummaryService`
- `daily_briefs` table is deployed
- Cron infrastructure is deployed and testable (or a manual trigger endpoint exists for test environments)
- Test user A: has 3 session logs in the last 7 days + gym streak 5, 7 wardrobe items, 3 meals this week, 2 purchases, 4 active routines; `brief_delivery_time` configured to a time 2 minutes in the future at test start; `timezone` configured correctly
- Test user B: zero session logs, zero module data
- Test user C: has module data only (zero session logs); `brief_delivery_time` configured
- Test user D: has an existing `daily_briefs` record for today already

---

## Core Test Flow

### TC-006-001-001: Full happy path — cron fires at delivery time, assembles real data, generates and stores a data-grounded brief

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-003-006, S-003-001, S-005-001

**Preconditions:**
- Test user A's `brief_delivery_time` is set to trigger within the next cron window (≤ 5 minutes)
- No brief exists for test user A today in `daily_briefs`
- Session logs from the past 7 days are seeded (3 sessions)
- Module data is seeded: gym streak 5, 7 wardrobe items
- S-006-004 push dispatch endpoint is available (mock APNs in test environment)

**Steps:**
1. Confirm no `daily_briefs` record exists for test user A for today's date in their timezone
2. Wait for the cron job to fire (up to 10 minutes, or trigger manually via test endpoint)
3. Query `daily_briefs` for test user A where `date = today` (in user timezone)
4. Inspect the `status` field
5. Inspect the `insight` field
6. Inspect the `action_items` array
7. Inspect the `generated_at` timestamp
8. Inspect the `context_session_logs_count` field
9. Confirm S-006-004 push dispatch was triggered

**Expected Result:**
- Step 3: exactly one record exists for today
- Step 4: `status = generated`
- Step 5: `insight` is a non-empty string that references a specific data point — e.g., contains "5" (streak number), "7" (wardrobe count), a food or spending pattern from the assembled context; does not contain generic phrases such as "keep pushing" or "great work"
- Step 6: `action_items` array has between 1 and 3 strings; each item is specific and actionable (e.g., "Cook at least 2 meals today." not "Be healthier")
- Step 7: `generated_at` is within 5 minutes of the user's configured `brief_delivery_time` on the correct local calendar date
- Step 8: `context_session_logs_count` = 3
- Step 9: a push notification dispatch was attempted for test user A (APNs mock received the payload)

**Failure Indicators:**
- No record in `daily_briefs` after cron runs
- `status` is anything other than `generated`
- `insight` contains generic motivational copy with no data references
- `action_items` array is empty or has more than 3 items
- `generated_at` is more than 5 minutes from the configured delivery time
- Push dispatch was not triggered

---

## Sub Flows

### TC-006-001-002: Deduplication — second cron run for same user same day is skipped

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-4
**Dependencies:** S-003-006, S-003-001, S-005-001

**Preconditions:**
- Test user D already has a `daily_briefs` record for today with `status: generated`
- The cron job is triggered again manually (simulating a double-run scenario)

**Steps:**
1. Confirm 1 existing `daily_briefs` record for test user D for today
2. Trigger the cron job again for test user D
3. Wait for job completion
4. Query `daily_briefs` count for test user D for today

**Expected Result:**
- Step 4: still exactly 1 record — no duplicate was created
- No error is logged for skipping (it is an expected no-op)

**Failure Indicators:**
- A second `daily_briefs` record is created for the same user on the same day
- The job errors out instead of cleanly skipping

---

### TC-006-001-003: Activity gate — zero-activity user receives no brief

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-5
**Dependencies:** None

**Preconditions:**
- Test user B has zero session logs and zero module data (gym_workouts = 0, wardrobe_items = 0, etc.)
- `brief_delivery_time` is configured for test user B within the next cron window

**Steps:**
1. Wait for the cron job to run for test user B (or trigger manually)
2. Query `daily_briefs` for test user B for today

**Expected Result:**
- No `daily_briefs` record is created for test user B
- No error is logged — skipping a zero-activity user is a clean no-op

**Failure Indicators:**
- A brief is generated and stored despite zero session logs and zero module data

---

### TC-006-001-004: Session log fetch fails — brief generated with module data only, flagged in metadata

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-003-006, S-005-001

**Preconditions:**
- `SessionLogRepository.getLogs()` is configured to throw an error for test user A (mock/stub)
- Test user A has active module data (gym streak 3, 5 wardrobe items)

**Steps:**
1. Trigger the brief generation cron for test user A with session log fetch failing
2. Query `daily_briefs` for test user A for today
3. Inspect the `metadata` field
4. Inspect the `insight` field

**Expected Result:**
- Step 2: a record exists with `status: generated` (brief was still produced)
- Step 3: `metadata` contains `session_logs_unavailable: true`
- Step 4: `insight` references module data (e.g., gym streak, wardrobe count) rather than session history; the brief is coherent and data-grounded despite missing session context
- No user-facing error; the brief is available in the app normally

**Failure Indicators:**
- Brief generation fails entirely when session logs are unavailable
- `metadata` is null or does not contain the `session_logs_unavailable` flag
- `insight` is empty or generic due to lack of session data

---

### TC-006-001-005: AI generation call fails — status set to failed, retry at +15 minutes succeeds

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-003-006, S-005-001

**Preconditions:**
- The AI generation call is mocked to fail on the first attempt and succeed on the second attempt
- Test user A has full context data

**Steps:**
1. Trigger brief generation for test user A with the first AI call configured to fail
2. Immediately query `daily_briefs` for test user A
3. Inspect the `status`
4. Wait 15 minutes (or advance test clock) for the `BriefRetryJob` to run
5. Query `daily_briefs` again

**Expected Result:**
- Step 3: `status = failed` after the first attempt
- Step 5: `status = generated` after the retry succeeds; `insight` and `action_items` are populated with real data
- No user-facing error at any point

**Failure Indicators:**
- `status` is not set to `failed` after the initial failure
- Retry job does not run after 15 minutes
- `status` remains `failed` after the retry should have run

---

### TC-006-001-006: AI generation fails twice — no brief generated, user sees nothing broken

**Type:** Negative
**Priority:** P2
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-003-006, S-005-001

**Preconditions:**
- AI call is mocked to fail on both the initial attempt and the +15-minute retry

**Steps:**
1. Trigger brief generation with AI call failing
2. Wait for retry at +15 minutes (or advance clock)
3. Query `daily_briefs` for the user today
4. Check the brief display screen (S-006-002) as the user

**Expected Result:**
- Step 3: record has `status: failed`; brief content fields are null
- Step 4: the Daily tab shows "Brief unavailable today — Noxis will try again tomorrow" (tested in T-006-002); no crash, no broken card, no error codes visible

**Failure Indicators:**
- App crashes or shows a stack trace in the Daily tab
- `daily_briefs` record is absent entirely (not even a failed record)

---

### TC-006-001-007: Delivery time change mid-day — new time takes effect from next day

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** None

**Preconditions:**
- Test user A's `brief_delivery_time` was 07:00; 07:00 has already passed today
- User updates `brief_delivery_time` to 06:00

**Steps:**
1. Update the user's `brief_delivery_time` to 06:00 (a time that has already passed today)
2. Check whether today's brief is regenerated
3. Wait until the next day's 06:00 window
4. Check whether a brief is generated at 06:00 on the next day

**Expected Result:**
- Step 2: no brief is regenerated today; the existing brief (if any) is unchanged; no new record
- Step 4: a brief is generated within 5 minutes of 06:00 on the next day

**Failure Indicators:**
- Today's brief is regenerated when the delivery time is changed after the window has passed
- No brief appears at the new time on the following day

---

## Automation Notes

- Cron testing: the test environment should expose a manual trigger endpoint (e.g., `POST /internal/test/trigger-brief-generator?userId=xxx`) so tests do not need to wait for the real 5-minute cron interval
- AI call mocking: use a configurable AI client stub that accepts a `fail_count` parameter; first N calls return 500, subsequent calls return a valid brief JSON
- Clock advancement: use a test-injectable clock (or database timestamp manipulation) for retry and deduplication tests
- Data-groundedness assertion: content assertions should use regex or keyword matching against known module data values (e.g., assert `insight` contains the user's seeded streak number); do not assert exact phrasing as AI output is non-deterministic
- Timezone DST test: run as a dedicated backend integration test with a user whose `timezone` is set to a DST-observing zone; advance the clock through a DST transition and assert that no brief is skipped or doubled
- Framework: backend integration tests (server-side test framework); manual exploratory testing for brief content quality

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

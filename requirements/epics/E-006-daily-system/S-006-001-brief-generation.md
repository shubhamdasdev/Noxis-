# Story: Daily Brief Generation Engine

---

**ID:** S-006-001
**Epic:** E-006
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to prepare a personalized morning brief every day before I open the app**, so that **the first thing I see is a sharp, relevant summary of where I stand and what matters today**.

## Goal

A server-side cron job generates one daily brief per user per day at their configured delivery time. The brief assembles context from the last 7 days of session logs, current module states, and USER.md goals, then produces a focused Noxis-voice brief: one 1-2 sentence insight, up to three actionable items, and an optional curated content card.

## Why / Rationale

The daily brief is the highest-frequency touchpoint for retained users. It is the reason to open the app every morning. Without it, the product lives or dies on whether the user remembers to initiate a conversation. The brief inverts that dynamic — Noxis comes to the user. And because it is grounded in real module data and session history, it is never generic.

## Functional Requirements

- A `DailyBriefGenerator` cron job runs per user once per day at the user's configured `brief_delivery_time` (stored in user settings, default 07:00 local time)
- The job runs in two passes:
  1. **Data assembly:** fetch last 7 days of session logs (S-003-006), fetch current module summary (gym streak, wardrobe item count, food patterns this week, spending this week, active routines), fetch USER.md goals section
  2. **Brief generation:** AI call with the assembled context; produces the brief JSON structure
- **Brief JSON structure:**
  ```json
  {
    "insight": "You've trained 5 of the last 7 days but your food has been mostly ordered. The gym work is real — the nutrition isn't keeping up.",
    "action_items": [
      "Cook at least 2 meals today.",
      "Your routine streak is at 4 days — don't break it tonight.",
      "You've got the navy blazer — wear it this week before you forget you own it."
    ],
    "curated_content": {
      "title": "The Compounding Effect of Small Habits",
      "type": "article",
      "url": null,
      "description": "A short read on why showing up consistently beats intensity every time."
    },
    "generated_at": "2026-03-18T07:00:00Z",
    "context_window_days": 7
  }
  ```
- `action_items` is capped at 3 items; may be fewer if context does not support 3 meaningful items
- `curated_content` is optional; included when the user's current focus area maps to an available piece of curated content (S-006-006); omitted if no match
- The insight must reference real data from the user's module state or session logs — generic statements ("keep pushing!") are explicitly prohibited in the generation prompt
- The brief is stored in the `daily_briefs` table with `status: generated` when the job completes
- If a brief for today already exists (e.g., the job ran twice due to a cron error), the second run is skipped — one brief per user per calendar day
- A brief is only generated if the user has at least one session log in the last 7 days OR has any module data (gym streak > 0, wardrobe items > 0, etc.); brand new users with zero activity receive no brief for their first day

## Prerequisites

- S-003-006 (session logs — required context source)
- S-003-001 (memory storage — USER.md and module context)
- S-005-001 (dashboard grid — module state data)

## Acceptance Criteria

- [ ] Given a user has session logs from the last 7 days and active module data, when the cron job runs at their configured delivery time, then a daily brief is generated and stored with `status: generated` within 60 seconds of the scheduled time
- [ ] Given the brief is generated, when the insight field is inspected, then it references a specific module data point (e.g., a gym streak number, a food pattern, a wardrobe item) — not generic motivational language
- [ ] Given the brief is generated, when the action_items array is inspected, then there are between 1 and 3 items, each referencing a specific actionable step relevant to the user's current context
- [ ] Given the cron job runs twice for the same user on the same day, when the second run executes, then it detects the existing brief and exits without creating a duplicate
- [ ] Given a user has zero session logs and zero module data, when the cron job runs, then no brief is generated for that user on that day
- [ ] Given the brief is generated, when stored, then `generated_at` is within 5 minutes of the user's configured `brief_delivery_time` on the correct local calendar date
- [ ] Given the user's configured delivery time is 06:30, when the brief is generated, then the push notification (S-006-004) is dispatched within the same job run

## Negative Scenarios

- Session log fetch fails for a user → the brief is generated with module data only (no session context); the brief is marked with a metadata flag `session_logs_unavailable: true`; no user-facing impact
- AI generation call fails → the job marks the brief as `status: failed`; a retry runs 15 minutes later; if the second attempt fails, no brief is generated for that day; the user sees nothing (no broken cards)

## Edge Cases

- User changes their configured delivery time mid-day (e.g., from 07:00 to 06:00 after 06:00 has already passed) → the new time takes effect from the next day; today's brief is not regenerated
- User is in a timezone with DST transition on the brief day → the stored `brief_delivery_time` in UTC is adjusted for the correct local time; no brief is skipped or doubled

## Dependencies

- S-003-006 (session logs)
- S-003-001 (memory storage)
- S-005-001 (dashboard grid)

## Technical Requirements

- **`DailyBriefGenerator` cron job (backend):** runs every 5 minutes; queries `users WHERE brief_delivery_time BETWEEN (now_utc - 5min) AND now_utc AND date(last_brief_generated_at, timezone) < today`; processes each eligible user
- **Data assembly:** parallel async fetches — `SessionLogRepository.getLogs(userId, last7Days)`, `ModuleSummaryService.getSummary(userId)`, `UserProfileRepository.getGoals(userId)` — assembled into a context string
- **Generation prompt:** "You are Noxis, a direct, high-standard personal advisor. Given this user's recent activity data, write a daily brief. The insight must reference a specific data point from the user's life — no generic motivation. Action items must be concrete and completable today. Tone: [user's tone mode]. Return JSON with: insight (string), action_items (array of up to 3 strings), curated_content (object or null)." Max tokens: 500.
- **`daily_briefs` table:** `(id UUID, user_id STRING, date DATE, insight TEXT, action_items TEXT[], curated_content JSONB, status ENUM(generated, failed), generated_at TIMESTAMP, viewed_at TIMESTAMP, context_session_logs_count INT, metadata JSONB)`
- **Deduplication check:** `SELECT 1 FROM daily_briefs WHERE user_id = ? AND date = today_in_user_timezone` before generating
- **Activity gate:** `SELECT COUNT(*) FROM session_logs WHERE user_id = ? AND date >= now() - interval '7 days' UNION ALL SELECT COUNT(*) FROM gym_workouts WHERE user_id = ?` — if total = 0, skip generation
- **Retry logic:** `BriefRetryJob` runs at +15min for any `status: failed` records from the current day

## Future Scope

- Brief personalization evolution: as the user accumulates more history, the insight shifts from module-level to pattern-level ("You've been training consistently for 3 months — your baseline is higher now")
- Weekend vs. weekday brief format variation
- Brief quality scoring: allow users to rate their brief (1-5) to improve generation over time
- Team briefs: multiple users sharing a brief context (couples or accountability partners, v4)

## Do Not Do

- In this story, do not build the brief display screen — that is S-006-002
- Do not implement push notification dispatch in this story — that is S-006-004 (though this story triggers it)
- Do not generate a brief for users with zero activity — it must be grounded in real data

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

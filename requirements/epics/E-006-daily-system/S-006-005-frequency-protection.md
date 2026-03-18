# Story: Frequency Protection — Adaptive Brief Format

---

**ID:** S-006-005
**Epic:** E-006
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to back off when I've gone quiet**, so that **I'm not receiving briefs I'm not reading, and when I come back the app feels welcoming rather than like I owe it attention**.

## Goal

When a user hasn't opened the app or their brief for 3+ consecutive days, the brief format shortens to headline-only. After 7 days, push notifications pause. When the user returns, full format and notifications resume automatically. This prevents the "ignored notification spiral" that kills retention.

## Why / Rationale

A user who ignores notifications for 3 days is on the edge of churning. Sending the same full brief every morning to someone who isn't reading it trains them to ignore Noxis entirely. Shortening the format reduces the felt "debt" of unread briefs. Pausing notifications after 7 days respects their absence. When they return, Noxis welcomes them back without guilt — and the full experience resumes naturally.

## Functional Requirements

- The `DailyBriefGenerator` (S-006-001) evaluates the user's engagement state before generating each brief:
  - Check `days_since_last_open`: number of consecutive calendar days since the user's last app open (tracked via `user_sessions.last_opened_at`)
  - Check `days_since_last_brief_viewed`: number of calendar days since the last `daily_briefs.viewed_at` was set

**Engagement tiers:**

| State | Condition | Brief Format | Push Notifications |
|---|---|---|---|
| Active | last open < 3 days ago | Full (insight + up to 3 actions + curated content) | Send |
| Quiet | last open 3–6 days ago | Short (insight only — no action items, no curated content) | Send |
| Dormant | last open 7+ days ago | No brief generated | Paused |
| Returning | user opens app after dormant | Immediate: generate a "welcome back" brief in short format; push resume next day | Resume next day |

- **Short format brief:** contains the `insight` field only; `action_items` is an empty array; `curated_content` is null; a `format: "short"` field indicates the format in the record
- **Dormant state:** no brief is generated for that day; no push is sent; the Daily tab shows the last available brief (which may be from several days ago)
- **Welcome-back brief:** when a dormant user opens the app, a one-time brief is generated on demand (triggered by app open, not the cron job): insight only, tone acknowledges the return without guilt: "You've been gone a few days. No lecture — let's pick back up." Max 2 action items for the welcome-back brief.
- **Push resume:** the day after a returning user first opens the app, push notifications resume at their configured delivery time; the cron job re-evaluates their engagement state
- Engagement state is tracked server-side; the client reports app opens via `POST /api/v1/sessions/open` on each cold launch after auth
- The engagement tier is logged on each `daily_briefs` record as `engagement_tier: "active" | "quiet" | "dormant"`

## Prerequisites

- S-006-001 (daily brief generation — this story modifies the brief generator's behavior)
- S-006-004 (push notification — this story controls whether pushes are sent)

## Acceptance Criteria

- [ ] Given a user has not opened the app for 4 days, when the brief generator runs, then the brief is generated in short format (insight only, no action items, no curated content) and `format: "short"` is set on the record
- [ ] Given a user has not opened the app for 7 days, when the brief generator runs, then no brief is generated for that day and no push notification is dispatched
- [ ] Given a dormant user opens the app, when the open is detected, then a welcome-back brief is generated on demand within 30 seconds and displayed on the Daily tab
- [ ] Given a returning user's welcome-back brief is generated, when the insight is inspected, then it does not contain guilt-inducing language and does not reference the absence duration negatively
- [ ] Given a returning user opens the app today (breaking dormancy), when the following day's cron job runs, then the full brief format resumes and push notifications are sent again
- [ ] Given an active user (opened app yesterday), when the brief generator runs, then the full format brief is generated with insight, up to 3 action items, and optional curated content
- [ ] Given each brief is generated, when the record is stored, then the `engagement_tier` field is populated with the correct tier value

## Negative Scenarios

- App open reporting call (`POST /api/v1/sessions/open`) fails → the app still launches normally; the missed open event means engagement state is not updated for that session; the next successful open report catches up correctly
- User opens the app but immediately closes it (< 2 seconds open time) → the open event is still reported; any open counts as engagement for frequency protection purposes

## Edge Cases

- User is in a timezone far from UTC and opens the app at 00:01 local time (day boundary) → the calendar day is evaluated in the user's local timezone; the open correctly counts for the new local day
- User reinstalls the app → `last_opened_at` is reset to null on the backend when the account is re-authenticated from a fresh install; treated as a returning user; welcome-back brief is generated on first open

## Dependencies

- S-006-001 (daily brief generation)
- S-006-004 (push notification)

## Technical Requirements

- **`user_sessions` table:** add column `last_opened_at TIMESTAMP`; updated by `POST /api/v1/sessions/open` (upsert by user_id)
- **`DailyBriefGenerator` modification:** before generating, call `EngagementService.getEngagementTier(userId)` → returns `EngagementTier` enum; `.dormant` → skip generation; `.quiet` → set `format: "short"`, generate insight only; `.active` → generate full brief
- **`EngagementService.getEngagementTier(userId)`:** `SELECT last_opened_at FROM user_sessions WHERE user_id = ?`; compute `daysSinceOpen = Calendar.current.dateComponents([.day], from: last_opened_at, to: now).day`; return `.active` (< 3), `.quiet` (3-6), `.dormant` (7+)
- **Welcome-back brief generation:** `AppOpenService.handleAppOpen(userId)` — on each cold launch `POST /api/v1/sessions/open`; if the server detects a dormant→returning transition, it triggers `DailyBriefGenerator.generateWelcomeBack(userId)` and returns `{ "welcome_back_brief_ready": true }` in the response; client checks this flag and navigates to Daily tab regardless of time
- **`daily_briefs.format` field:** `ENUM('full', 'short')` — default 'full'; set to 'short' when tier is quiet or welcome-back
- **`daily_briefs.engagement_tier` field:** `ENUM('active', 'quiet', 'dormant', 'welcome_back')` — logged per record
- **Push pause implementation:** `APNSService.send()` checks `EngagementTier` before dispatch; `.dormant` → skip push; all other tiers → send

## Future Scope

- Configurable re-engagement thresholds: user sets their own "back off after N days" preference
- Re-engagement nudge: a single non-brief notification after 10 days of dormancy: "Noxis still has your back — tap to come back when you're ready"
- Analytics: dormancy rate, welcome-back conversion rate, brief format effectiveness comparison

## Do Not Do

- In this story, do not send any re-engagement push notification during dormancy — silence is the strategy
- Do not show any "you haven't opened the app in X days" messaging in the welcome-back brief
- Do not apply frequency protection to any feature other than the daily brief and its associated push notification

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

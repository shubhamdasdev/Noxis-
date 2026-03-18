# Story: Food Module — Meal Log & Nutrition Patterns

---

**ID:** S-005-004
**Epic:** E-005
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a food module that shows what I've been eating and how my patterns look**, so that **Noxis can give me food advice that reflects what I actually eat rather than what I think I eat**.

## Goal

The Food module detail screen shows a meal log, a delivery-vs-cooked ratio for the week, a quick-log button for manual meal entry, and a pattern summary from Noxis. No calorie counting. Qualitative logging only — it is about patterns and quality, not macros.

## Why / Rationale

Food advice without food data is noise. When Noxis knows the user has ordered food 5 of the last 7 days, it has the grounds to call it out. When it knows the user cooked a high-protein breakfast three days in a row, it can acknowledge the win. Without food module data, the AI has to take the user's word for their habits — which is the least accurate data source.

## Functional Requirements

- The Food module screen has three sections:
  1. **Pattern summary card** — a `GlassCard` at the top showing a 2-3 sentence Noxis-generated assessment of this week's food pattern (e.g., "5 of 7 meals this week were ordered. You're spending money on convenience — not necessarily a problem, but worth noticing. Three of those meals included solid protein."); refreshed weekly when the summary job runs
  2. **Delivery vs. cooked ratio** — a simple visual ratio bar for the current week: percentage of meals that were delivered vs. home-cooked; determined by Noxis's assessment of each logged meal (no user tagging required)
  3. **Meal log** — chronological list (most recent first) of logged meals; each row: relative date/time, meal description (e.g., "Grilled chicken + rice"), optional photo thumbnail, and Noxis's one-line quality tag (e.g., "Good protein hit", "Processed — okay occasionally")

- **Quick-log button** — a `GlassButton` at the bottom: "Log a meal"; sheet opens with:
  - Description text field (required, max 200 chars) — e.g., "Eggs and avocado toast"
  - Optional photo picker (same flow as S-004-002 image picker, max 5MB)
  - Meal timing: Breakfast / Lunch / Dinner / Snack (segmented picker)
  - "Save" button

- On save, the meal is written to `POST /api/v1/food/meals`; Noxis generates a quality tag for the meal asynchronously (separate lightweight AI call); the quality tag is updated on the meal record when ready
- Meals can also be auto-synced from chat via S-005-007 (e.g., user sends a photo of food → classified as meal → logged automatically)
- The pattern summary card is refreshed by a weekly server job (runs every Monday morning at 07:00 local time)
- Empty state: "No meals logged yet — log your first meal or send Noxis a food photo in chat"

## Prerequisites

- S-005-001 (dashboard grid — navigation entry point)
- S-005-007 (chat extraction pipeline — auto-sync meals from chat)
- S-003-001 (memory storage — food preferences stored as memories)

## Acceptance Criteria

- [ ] Given the user logs a meal with a description, when the save completes, then the meal appears at the top of the meal log with the description and meal timing
- [ ] Given a meal is saved and the quality-tag AI call completes, when the meal row is next displayed, then the one-line quality tag is visible beneath the description
- [ ] Given the food module has meals logged for the current week, when the delivery/cooked ratio bar is displayed, then it correctly reflects the proportion of delivered vs. home-cooked meals based on Noxis's assessment
- [ ] Given the pattern summary card is present, when the screen renders, then it shows a 2-3 sentence Noxis-generated text assessment — not a generic placeholder
- [ ] Given a meal photo is sent to Noxis in chat and classified as a meal, when the Food module is opened, then that meal appears in the log with `source: chat_sync`
- [ ] Given no meals are logged, when the screen renders, then the empty state message and the quick-log button are the only elements visible (no broken ratio bars or empty cards)
- [ ] Given the user opens the quick-log sheet, when the meal timing picker is displayed, then Breakfast, Lunch, Dinner, and Snack are the only four options

## Negative Scenarios

- Meal quality-tag AI call fails → the meal is still stored; the quality tag is shown as blank; a background retry runs once after 60 seconds; if the second attempt fails, the quality tag remains blank permanently for that meal entry
- `GET /api/v1/food/meals` fails → inline error state in the meal log section with retry button; the pattern summary card and ratio bar still render from cached data if available

## Edge Cases

- User logs a meal with only a photo (no description) → valid; the description is set to an empty string; Noxis generates the quality tag from the photo via vision API
- Week has only 1 logged meal → the delivery/cooked ratio bar renders with that single meal's classification; no "insufficient data" hiding; the pattern summary acknowledges the limited data

## Dependencies

- S-005-001 (dashboard grid)
- S-005-007 (chat extraction pipeline)
- S-003-001 (memory storage)

## Technical Requirements

- **`FoodModuleView.swift`** — `ScrollView` with `PatternSummaryCard`, `DeliveryRatioBar`, `MealLogView`; plus `FoodQuickLogSheet`
- **`Meal` model:** `(id UUID, user_id, description: String, timing: MealTiming, photo_url: String?, quality_tag: String?, is_delivered: Bool?, logged_at: Date, source: LogSource (.manual | .chat_sync))`
- **`MealTiming` enum:** `breakfast | lunch | dinner | snack`
- **Quality tag generation:** `POST /api/v1/food/meals/{id}/generate-tag` — backend runs AI call with prompt: "Given this meal description (and photo if available), write a single short phrase (max 8 words) evaluating the food quality from a protein-first, whole-food perspective. Be direct, not judgmental." Sets `is_delivered: Bool` based on keywords (ordered/delivery/Uber Eats → true; cooked/made/prepared → false; ambiguous → null)
- **Pattern summary job:** `FoodPatternSummarizer` weekly cron; uses last 7 days of `Meal` records; calls AI with prompt focused on delivery ratio, protein quality, and frequency patterns; writes result to `food_summaries` table
- **Delivery ratio:** `SELECT COUNT(*) WHERE is_delivered = true / total` for the current week; null `is_delivered` records are excluded from the ratio but counted in total meal count for the summary
- **Backend routes:** `GET /api/v1/food/summary`, `GET /api/v1/food/meals?limit=30`, `POST /api/v1/food/meals`

## Future Scope

- Photo-based macro estimation (using vision AI to estimate protein/carb/fat from a meal photo) — v3 feature
- Apple Health food log integration
- Noxis weekly food report delivered via the daily brief: "This week in food: ..."
- Streak tracking for "home cooked meals this week"

## Do Not Do

- In this story, do not implement calorie counting, macro tracking, or portion size estimation
- Do not require the user to tag meals as delivered vs. cooked — Noxis infers this
- Do not implement a weekly food report in this story — that is part of the daily brief system (E-006)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

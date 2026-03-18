# Story: Spending Module — Purchase Log & Category Tracking

---

**ID:** S-005-005
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

As a **user**, I want **a spending module that tracks what I'm buying and where my money is going**, so that **Noxis can tell me whether my spending is compounding my life or just numbing it**.

## Goal

The Spending module shows recent purchases (auto-extracted from chat + manual entry), a weekly spend summary by category, and Noxis's qualitative assessment of spend quality. No bank integration — this is conversational and manual spending awareness, not a budgeting app.

## Why / Rationale

Most young men have no structured view of their discretionary spending beyond a bank statement. Noxis's lens is not "save more money" — it is "are your purchases compounding your identity or just providing momentary comfort?" The spending module exists to surface that pattern, not to enforce a budget.

## Functional Requirements

- The Spending module screen has three sections:
  1. **Spend quality assessment** — a `GlassCard` at the top with Noxis's weekly qualitative assessment (e.g., "You spent on 3 pieces of gym kit, a book, and ordered food 4 times. The gear compounds. The food orders are convenience spending — not bad, but watch the pattern."); generated weekly by a background job
  2. **Weekly spend by category** — a simple horizontal bar breakdown showing total spend across categories for the current week: Clothing / Food / Fitness / Entertainment / Other; each bar is labeled with the category name and total amount
  3. **Purchase log** — chronological list of logged purchases; each row: item description, amount, category, date; purchases marked as "compounding" (gym gear, books, courses) get a subtle accent-color dot; those marked "numbing" (impulse fast food, entertainment) get a muted dot; assessment is Noxis-generated

- **Manual add purchase** — a `GlassButton` at the bottom: "Log a purchase"; sheet opens with:
  - Item description (required, max 140 chars)
  - Amount field (numeric, two decimal places, currency symbol from user locale)
  - Category picker: Clothing / Food / Fitness / Entertainment / Other
  - "Save" button

- On save, the purchase is written to `POST /api/v1/spending/purchases`; Noxis generates a compounding vs. numbing tag for the purchase asynchronously
- Purchases auto-synced from chat (S-005-007) appear in the log with `source: chat_sync`
- The spend quality assessment card refreshes weekly (Monday 07:00 local time)
- Empty state: "No purchases tracked yet — add one manually or mention what you've bought in chat"

## Prerequisites

- S-005-001 (dashboard grid — navigation entry point)
- S-005-007 (chat extraction pipeline — auto-sync purchases from chat)
- S-003-001 (memory storage — spending patterns stored as memories)

## Acceptance Criteria

- [ ] Given the user logs a purchase with a description, amount, and category, when the save completes, then the purchase appears in the log and the weekly category bar updates to reflect the new amount
- [ ] Given a purchase is saved, when the compounding/numbing tag AI call completes, then the purchase row shows the appropriate colored dot indicator
- [ ] Given the spend quality assessment card is present, when the screen renders, then it shows a non-generic Noxis assessment that references actual recent purchases by type or category — not a placeholder
- [ ] Given the user has purchases in 3 categories this week, when the weekly breakdown is rendered, then exactly 3 category bars are shown with accurate amounts; empty categories show no bar
- [ ] Given a purchase is auto-synced from chat, when the Spending module opens, then it appears in the log with a `chat_sync` source indicator
- [ ] Given the user has no logged purchases, when the screen renders, then the empty state message is shown as the primary content and the log and breakdown sections are hidden
- [ ] Given the user opens the manual add sheet, when the amount field is displayed, then it accepts numeric input only and shows the user's local currency symbol

## Negative Scenarios

- `POST /api/v1/spending/purchases` fails → the purchase is not stored; the sheet remains open with an error message; the save button re-enables for retry
- The compounding/numbing tag AI call fails → the purchase is stored; the dot indicator is absent; no retry in v1 (tag remains blank)

## Edge Cases

- User logs an amount of 0 → allowed; edge cases like splitting a bill or noting a free item; a $0 entry is valid
- Purchase description contains a merchant name that could be in multiple categories (e.g., "Amazon — could be anything") → defaults to "Other" category; user selects the correct category manually

## Dependencies

- S-005-001 (dashboard grid)
- S-005-007 (chat extraction pipeline)
- S-003-001 (memory storage)

## Technical Requirements

- **`SpendingModuleView.swift`** — `ScrollView` with `SpendQualityCard`, `WeeklyCategoryBreakdown`, `PurchaseLogView`; plus `SpendingAddSheet`
- **`Purchase` model:** `(id UUID, user_id, description: String, amount: Decimal, category: SpendingCategory, spend_quality: SpendQuality?, logged_at: Date, source: LogSource (.manual | .chat_sync))`
- **`SpendingCategory` enum:** `clothing | food | fitness | entertainment | other`
- **`SpendQuality` enum:** `compounding | neutral | numbing`
- **Compounding/numbing tag:** `POST /api/v1/spending/purchases/{id}/generate-tag` — AI prompt: "Classify this purchase as 'compounding' (builds skill, identity, capability), 'neutral', or 'numbing' (comfort, impulse, escapism). Return one of: compounding, neutral, numbing. Purchase: [description] in category [category]."
- **Weekly category breakdown:** `GET /api/v1/spending/summary?week=current` returns `{ categories: [{category, total_amount, purchase_count}], quality_assessment: String }`
- **Spend quality assessment job:** `SpendQualitySummarizer` weekly cron (Monday 07:00 local); uses last 7 days of purchases; AI prompt references actual purchase descriptions and amounts; stores result in `spending_summaries` table
- **Amount field:** `TextField` with `keyboardType: .decimalPad`; formatted with `NumberFormatter` using user's `Locale.current`
- **Backend routes:** `GET /api/v1/spending/summary`, `GET /api/v1/spending/purchases?limit=30`, `POST /api/v1/spending/purchases`

## Future Scope

- Bank/card integration (Plaid or Open Banking API) for automatic transaction import — v3
- Monthly spend trend chart
- Noxis-initiated spending check-in: "You've spent £340 on food this month. That's 60% more than last month. Want to talk about it?"
- Budget target setting per category with soft alerts

## Do Not Do

- In this story, do not integrate with any bank, payment provider, or card network
- Do not implement receipt scanning or OCR in v1
- Do not show a total net worth, savings, or investment view — this is discretionary spend awareness only

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

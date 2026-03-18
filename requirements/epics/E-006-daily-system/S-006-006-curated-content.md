# Story: Curated Content Integration

---

**ID:** S-006-006
**Epic:** E-006
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P3
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **my morning brief to occasionally include a piece of reading or a recommendation relevant to what I'm working on**, so that **Noxis not only reflects my life back at me but also points me toward things worth consuming**.

## Goal

Each daily brief optionally includes one curated content item (article, book recommendation, or short video reference) matched to the user's current focus area. Content comes from a manually curated editorial list organized by life module category. Content refreshes on a weekly rotation per category. The item is shown as a linked card at the bottom of the brief.

## Why / Rationale

A brief that only reflects what the user has already been doing can feel like a mirror. The curated content item is the editorial layer — it points the user forward rather than just summarizing where they are. For a user deep in a gym phase, a link to something about training philosophy or recovery is a signal that Noxis is paying attention at a deeper level than just counting workout days.

## Functional Requirements

- A `curated_content` table stores all content items with: `id UUID`, `title: String`, `type: ContentType (article | book | video_reference)`, `description: String` (2-3 sentences), `url: String?` (null for book recommendations without a URL), `category: LifeModule` (wardrobe | gym | food | spending | routines | general), `week_number: Int` (1-52, which week of the year this item is shown), `is_active: Bool`
- Content is manually populated by the editorial team (not web search, not AI-generated URLs); the table is seeded with at least 2 items per category per week for the first 12 weeks of launch
- The brief generator (S-006-001) includes curated content by:
  1. Determining the user's primary active module (the module with the most activity in the last 7 days from session logs)
  2. Querying `curated_content WHERE category = primaryModule AND week_number = currentWeekNumber AND is_active = true`
  3. If no match for the primary module, fall back to `category = 'general'`
  4. If no general content for this week, `curated_content` is omitted from the brief entirely
  5. If multiple items match for the same week and category, select one deterministically (e.g., lowest `id` first — no randomness; same user sees the same content item all week)
- The curated content item is embedded in the `daily_briefs.curated_content` JSON field (title, type, url, description)
- The content card is shown at the bottom of the brief card (S-006-002); tapping opens `SFSafariViewController` if a URL is present; for books with no URL, tapping opens a bottom sheet with the title, author, and full description
- Content items are reviewed and rotated weekly by the editorial team; `week_number` is fixed per item; a single item can appear in multiple weeks by duplicating the record
- A `curated_content_admin` API endpoint (authenticated, admin role only) allows CRUD operations on the content library without a code deploy

## Prerequisites

- S-006-001 (daily brief generation — curated content is embedded in the brief record)
- S-006-002 (brief screen — the content card UI is displayed here)

## Acceptance Criteria

- [ ] Given the user's primary active module for the week is `gym` and a gym content item exists for the current week number, when the brief is generated, then the `curated_content` field is populated with that item's title, type, description, and url
- [ ] Given no content exists for the user's primary module this week, when the brief generator queries for content, then it falls back to `category: general`; if no general content exists this week, `curated_content` is null
- [ ] Given the same user generates briefs on Tuesday and Thursday of the same week, when both briefs are inspected, then both contain the same `curated_content` item for that week and category (deterministic selection, not random)
- [ ] Given a curated content card is displayed in the brief, when the user taps a card with a URL, then `SFSafariViewController` opens the URL in-app
- [ ] Given a curated content card is a book recommendation with no URL, when the user taps the card, then a bottom sheet appears showing the title, author (from description), and full description text
- [ ] Given the brief generator cannot find any content item for any category this week, when the brief is generated, then `curated_content` is null and the brief card renders without a content section
- [ ] Given an admin creates a new content item via the admin API, when the next brief generation runs for an eligible user, then the new item is eligible for selection

## Negative Scenarios

- `curated_content` table query fails → the brief is generated without the curated content field (null); no brief generation failure; the error is logged
- The URL in a content item is no longer valid (404) → the app still opens the URL in `SFSafariViewController`; the broken link experience is handled by Safari; no pre-validation in v1

## Edge Cases

- Week 53 (some years have 53 ISO weeks) → content items with `week_number = 53` are used; if none exist, fall back to week 52 items
- User has equal activity across two modules in a week → `primaryModule` is determined by the module with the highest `message_count` in session logs; ties broken by the lower enum ordinal (wardrobe < gym < food < spending < routines)

## Dependencies

- S-006-001 (daily brief generation)
- S-006-002 (brief screen)

## Technical Requirements

- **`curated_content` table:** `(id UUID, title VARCHAR(200), type ENUM('article','book','video_reference'), description TEXT, url VARCHAR(500) NULLABLE, category LifeModule, week_number SMALLINT, is_active BOOL DEFAULT true, created_at TIMESTAMP)`
- **Content query:** `SELECT * FROM curated_content WHERE category = ? AND week_number = ? AND is_active = true ORDER BY id ASC LIMIT 1`
- **Primary module determination:** `CuratedContentSelector.primaryModule(userId: String, weekStart: Date)` → reads session logs for the past 7 days; counts `module_activity` occurrences per category; returns the most frequent category; caches result in `user_content_preferences` table per week
- **`SFSafariViewController` presentation:** `ContentCardView` in `S-006-002`; `@State var showSafari = false`; `.sheet(isPresented: $showSafari) { SafariView(url: url) }`
- **Book bottom sheet:** `BookDetailSheet.swift` — `GlassCard` inside a `.sheet`; shows `title`, `description`; no URL needed
- **Admin API:** `GET/POST/PUT/DELETE /api/v1/admin/curated-content` protected by `admin` role JWT claim; used by the editorial team via any REST client (no admin UI in v1)
- **Seed data:** SQL seed file with 52 weeks × 6 categories × 2 items = 624 records minimum; to be populated before launch; provided by the PM/editorial team as a CSV for import

## Future Scope

- AI-assisted content selection: rather than week-based rotation, use semantic matching between the brief insight and available content items for the best-fit recommendation
- User-submitted content recommendations: allow users to suggest articles or books that Noxis might recommend to others
- Content performance tracking: measure click-through rates per content item; surface in admin dashboard; auto-retire low-performing items
- Affiliate links for book recommendations (bookshop.org or Amazon Associates)

## Do Not Do

- In this story, do not implement web search or real-time content fetching — all content is from the curated manual list only
- Do not show more than one content item per brief
- Do not implement user content preferences or "don't show me this type" filtering in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

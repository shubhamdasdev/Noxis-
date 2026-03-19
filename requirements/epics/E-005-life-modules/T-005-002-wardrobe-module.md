# QA Test Plan — Wardrobe Module — Inventory & Outfit Log

---

**Plan ID:** T-005-002
**Story:** S-005-002
**Epic:** E-005
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan covers the Wardrobe module detail screen in full: tab switching between Inventory and Outfits, manual item addition via the Add Item sheet (photo, category, notes), item persistence to `POST /api/v1/wardrobe/items`, item detail view and deletion, auto-synced outfit entries sourced from the chat extraction pipeline (S-005-007), empty states on both tabs, and pull-to-refresh. Error handling for photo upload failure and API fetch failure are also covered.

## Out of Scope

- Chat extraction sync logic (S-005-007) — this plan tests only that auto-synced outfit entries display correctly once they exist in the backend
- Outfit rating or chat pipeline behavior
- Manual outfit entry (not in v1)
- Outfit builder, duplicate detection, wear frequency tracking, archive/donate features (future scope)
- Dashboard grid navigation entry point (tested in T-005-001)

## Prerequisites

- S-005-001 (dashboard grid) is implemented; user can navigate to Wardrobe from the grid
- S-005-007 (chat extraction pipeline) is implemented; `OutfitEntry` records can be pre-seeded in the test database
- S-003-001 (memory storage) is implemented
- Backend routes are deployed: `GET /api/v1/wardrobe/items`, `POST /api/v1/wardrobe/items`, `DELETE /api/v1/wardrobe/items/{id}`, `GET /api/v1/wardrobe/outfits`
- Media upload endpoint `/api/v1/media/upload` is operational in the test environment
- Test user A: has 3 existing wardrobe items and 2 outfit entries pre-seeded
- Test user B: has zero items and zero outfits

---

## Core Test Flow

### TC-005-002-001: Full happy path — both tabs render correctly, add item, view detail, delete item, view outfit entry

**Type:** E2E
**Priority:** P0
**AC Covered:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Dependencies:** S-005-001, S-005-007, S-003-001

**Preconditions:**
- Test user A is authenticated with 3 wardrobe items and 2 outfit entries pre-seeded
- At least one outfit entry has a valid `photo_url`, a `noxis_rating` of "Clean and intentional", and a known `created_at` date
- Device camera and photo library simulator permissions are granted

**Steps:**
1. Navigate to the Wardrobe module from the Modules tab
2. Observe which tab is active on screen load
3. Confirm both "Inventory" and "Outfits" tab labels are visible
4. Inspect the Inventory grid for the 3 pre-seeded items
5. Tap the "Add Item" floating action button (bottom right)
6. In the add item sheet: select a photo from the photo library
7. Select category "Jacket" from the category picker
8. Enter "navy blazer from work event" in the notes field (32 characters)
9. Tap "Save"
10. Confirm the new item appears in the Inventory grid
11. Tap the newly added item card
12. Inspect the detail view
13. Initiate delete from the detail view (swipe up and confirm delete)
14. Confirm the item is removed from the grid
15. Tap the "Outfits" tab
16. Inspect the outfit list for the 2 pre-seeded entries
17. Tap the first outfit row
18. Inspect the expanded detail

**Expected Result:**
- Step 2: Inventory tab is the default active tab
- Step 3: both "Inventory" and "Outfits" tabs are visible in the tab bar at the top
- Step 4: 3 item cards displayed in a 3-column grid; each shows a photo thumbnail (or placeholder), a category label, and a 1-line truncated notes preview
- Step 5: an add item sheet slides up
- Step 6–8: photo picker, category picker, and notes field are all present and interactive
- Step 9: sheet dismisses; the new item appears as the latest card in the grid (4 total items now)
- Step 11: detail view opens showing: full photo, full notes text "navy blazer from work event", date added, and a visible delete affordance
- Step 13: `DELETE /api/v1/wardrobe/items/{id}` is called; the item is removed from the grid immediately (3 items again)
- Step 16: 2 outfit rows visible; each row shows a photo thumbnail, a Noxis rating phrase (e.g., "Clean and intentional"), and a date
- Step 17–18: the row expands in-place to show the full outfit photo and full Noxis judgment text; no navigation to a new screen

**Failure Indicators:**
- Outfits tab is the default active tab on open
- Add item sheet missing any of the three fields (photo, category, notes)
- Newly added item does not appear without a screen refresh
- Delete does not remove the item from the grid
- Outfit rows do not show rating text
- Outfit row expansion navigates to a new screen instead of expanding in-place

---

## Sub Flows

### TC-005-002-002: Empty state on Inventory tab — correct copy shown

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-4
**Dependencies:** S-005-001

**Preconditions:**
- Test user B has zero wardrobe items and zero outfits

**Steps:**
1. Navigate to Wardrobe module as test user B
2. Confirm the Inventory tab is active by default
3. Read the empty state message

**Expected Result:**
- The empty state message reads exactly: "No items yet — send Noxis a photo in chat to get started"
- No item grid or ghost/placeholder cards are shown
- The "Add Item" floating action button is still visible and tappable

**Failure Indicators:**
- Message text differs from the specified copy
- Grid cells or placeholder rows appear instead of the empty state
- Add Item button is hidden in the empty state

---

### TC-005-002-003: Empty state on Outfits tab — correct copy shown

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-4
**Dependencies:** S-005-001

**Preconditions:**
- Test user B has zero outfit entries

**Steps:**
1. Navigate to Wardrobe module as test user B
2. Tap the "Outfits" tab
3. Read the empty state message

**Expected Result:**
- Empty state message reads: "No outfits yet — send Noxis a photo in chat to get started"
- No outfit rows or placeholders shown

**Failure Indicators:**
- Wrong copy (e.g., "No items yet" instead of "No outfits yet")
- Empty state not shown; list appears with no rows and no message

---

### TC-005-002-004: Photo upload fails during Add Item — item saves without photo, error indicator shown

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-005-001

**Preconditions:**
- `/api/v1/media/upload` is configured to return a 500 error (test network stub)
- Test user A is authenticated

**Steps:**
1. Open the Wardrobe module, Inventory tab
2. Tap "Add Item"
3. Select a photo from the photo library
4. Select category "T-Shirt"
5. Leave notes empty
6. Tap "Save"

**Expected Result:**
- The add item sheet displays an error indicator within the photo area
- A "Retry photo" option is visible within the sheet
- The item is still saved to `POST /api/v1/wardrobe/items` without a `photo_url`
- The new item appears in the Inventory grid with a placeholder glass card (no photo thumbnail)
- The sheet does not remain permanently blocked in an error state

**Failure Indicators:**
- The entire save operation fails because the photo upload failed
- No "Retry photo" option is shown
- The item does not appear in the grid at all
- The sheet freezes or becomes unresponsive

---

### TC-005-002-005: GET /api/v1/wardrobe/items fails — inline error with retry, navigation remains functional

**Type:** Negative
**Priority:** P1
**AC Covered:** None (FR / Negative Scenario)
**Dependencies:** S-005-001

**Preconditions:**
- `GET /api/v1/wardrobe/items` returns a 500 error (network stub)
- Test user A is authenticated

**Steps:**
1. Navigate to Wardrobe module
2. Wait for the fetch to fail
3. Inspect the Inventory tab
4. Tap the "Outfits" tab
5. Tap the back button to return to the Modules grid

**Expected Result:**
- Step 3: an inline error state appears in the grid area (e.g., "Couldn't load your wardrobe — pull to refresh" or equivalent) with a visible retry affordance
- No blocking modal or full-screen error overlay
- Step 4: switching to the Outfits tab still works
- Step 5: back navigation to the Modules grid works normally

**Failure Indicators:**
- A full-screen error blocks all interaction
- Tab switching is disabled
- Back navigation fails

---

### TC-005-002-006: Pull-to-refresh on Inventory and Outfits tabs

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-7
**Dependencies:** S-005-001

**Preconditions:**
- Test user A has existing data on both tabs
- A new wardrobe item is added directly to the database between the initial load and the refresh step

**Steps:**
1. Open Wardrobe module, Inventory tab
2. Note the current item count
3. Pull down to trigger pull-to-refresh on the Inventory tab
4. Observe the grid after refresh
5. Switch to the Outfits tab
6. Pull down to trigger pull-to-refresh on the Outfits tab
7. Observe the list after refresh

**Expected Result:**
- Step 4: the Inventory grid reflects any new data added to the database; new item appears if one was seeded
- Step 7: the Outfits list similarly refreshes and reflects any newly available outfit entries

**Failure Indicators:**
- Pull-to-refresh gesture has no effect
- New data does not appear after refresh
- Pull-to-refresh on one tab affects the other tab's state unexpectedly

---

### TC-005-002-007: Auto-synced outfit entry displays correctly from S-005-007

**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-3
**Dependencies:** S-005-001, S-005-007

**Preconditions:**
- An `OutfitEntry` record has been created by the chat extraction pipeline with: `photo_url` pointing to a valid image, `noxis_rating` = "Works for the occasion", `created_at` = today's date

**Steps:**
1. Open the Wardrobe module
2. Tap the "Outfits" tab
3. Locate the auto-synced outfit entry
4. Confirm the row content
5. Tap the row to expand

**Expected Result:**
- The row shows: a photo thumbnail, the rating text "Works for the occasion", and today's date
- On expand: the full outfit photo is shown at a larger size; the full Noxis judgment text (from `noxis_judgment_full`) is visible below the photo

**Failure Indicators:**
- Rating text is missing or shows raw JSON
- Photo thumbnail does not load (broken image instead of placeholder)
- Expand does not show the full judgment text

---

### TC-005-002-008: Large inventory — 200+ items, LazyVGrid handles efficiently

**Type:** Edge Case
**Priority:** P2
**AC Covered:** None (FR / Edge Case)
**Dependencies:** S-005-001

**Preconditions:**
- Test user has 210 wardrobe items pre-seeded
- `GET /api/v1/wardrobe/items` returns all 210 records (no pagination in v1)

**Steps:**
1. Open the Wardrobe module
2. Scroll slowly through the inventory grid from top to bottom
3. Measure approximate frame rate / smoothness (visual inspection)
4. Scroll back to the top

**Expected Result:**
- All items are accessible by scrolling
- Items load progressively as the user scrolls (lazy loading behavior of LazyVGrid)
- No crash, no freeze, no app termination
- Scrolling back to top is smooth

**Failure Indicators:**
- App crashes when loading 200+ items
- Visible jank/freeze when scrolling through large inventory
- Memory warning triggers visible stutter

---

## Automation Notes

- Selector strategy: accessibility identifiers on `WardrobeItemCard` keyed by item ID; "Add Item" button identified by `accessibilityLabel = "Add Item"`; tabs identified by `accessibilityLabel = "Inventory"` and `accessibilityLabel = "Outfits"`
- Photo picker: use `UIImagePickerController` mock in unit tests; in XCUITest, use the simulator photo library seeded with test assets
- Network stubs: use URLProtocol-based stubs to simulate 500 errors for specific endpoints without affecting other requests
- Delete flow: XCUITest can simulate swipe gesture on the detail view; confirm deletion via assertion on item count
- Flakiness risk: photo picker interactions in XCUITest can be flaky — use a dedicated test image in the simulator library and assert using accessibility rather than visual snapshot
- Framework: XCTest (unit for ViewModel logic) + XCUITest (E2E flows)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

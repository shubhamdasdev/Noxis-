# Story: Wardrobe Module — Inventory & Outfit Log

---

**ID:** S-005-002
**Epic:** E-005
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **a dedicated wardrobe screen that tracks what I own and what outfits Noxis has rated**, so that **I can build on my style over time and remember what actually works**.

## Goal

The Wardrobe module detail screen shows two sections: an inventory grid of logged clothing items (photo + category + notes) and an outfit log of past Noxis-rated outfits. A floating Add Item button allows manual entry. Items can also be auto-synced from chat via S-005-007.

## Why / Rationale

Style advice without inventory context is generic. When Noxis knows what you own, it can tell you whether what you're about to wear makes sense with your existing pieces, flag when something in your wardrobe is underused, or recognize a recurring fit pattern. The wardrobe inventory is what turns image classification from one-off feedback into ongoing style intelligence.

## Functional Requirements

- The Wardrobe screen has two tabs at the top: "Inventory" and "Outfits"

**Inventory tab:**
- Displays a 3-column grid of wardrobe item cards; each card shows: item photo thumbnail (or a placeholder glass card if no photo), category label (T-Shirt / Pants / Shoes / Jacket / Accessory / Other), and a brief notes line (truncated to 1 line)
- "Add Item" floating action button (bottom right); tapping opens an add item sheet with: photo picker (from library or camera), category picker (segmented or dropdown), notes text field (optional, max 140 chars)
- On save, the item is written to `POST /api/v1/wardrobe/items` and appended to the grid
- Each item card is tappable: opens a detail view showing the full photo, full notes, date added, and a swipe-up to delete

**Outfits tab:**
- Displays a vertical list of past outfit entries; each row shows: outfit photo thumbnail, Noxis's rating (a short phrase, e.g., "Clean and intentional" or "Works for the occasion"), and the date
- Outfit entries are created automatically by S-005-007 when a chat image is classified as `outfit` and Noxis provides a judgment
- Outfit rows are not manually addable in v1
- Tapping an outfit row expands it to show the full photo and full Noxis judgment text

**Both tabs:**
- Empty state: "No [items / outfits] yet — send Noxis a photo in chat to get started"
- Pull-to-refresh on both tabs

## Prerequisites

- S-005-001 (dashboard grid — navigation entry point)
- S-005-007 (chat extraction pipeline — auto-sync of items and outfits from chat)
- S-003-001 (memory storage — wardrobe preferences stored as memories)

## Acceptance Criteria

- [ ] Given the user opens the Wardrobe module, when the screen renders, then Inventory and Outfits tabs are visible and the Inventory tab is the default active tab
- [ ] Given the user taps "Add Item", when the sheet opens, then a photo picker, category selector, and notes field are present; tapping "Save" creates the item and it appears in the grid immediately
- [ ] Given an outfit is auto-synced from chat (S-005-007), when the user opens the Outfits tab, then the outfit appears with its photo, Noxis's rating text, and the date
- [ ] Given the Inventory grid has no items, when the screen renders, then the empty state is shown: "No items yet — send Noxis a photo in chat to get started"
- [ ] Given the user taps an item card, when the detail view opens, then the full photo, full notes, date added, and a delete option are all visible
- [ ] Given the user deletes an item from its detail view, when the delete is confirmed, then the item is removed from the grid and the `DELETE /api/v1/wardrobe/items/{id}` call is made
- [ ] Given the user pulls down on either tab, when the gesture completes, then the data is re-fetched and the grid / list updates

## Negative Scenarios

- Photo upload fails when adding an item → the item is saved without a photo (photo field is optional); an error indicator shows in the add item sheet with a "Retry photo" option
- `GET /api/v1/wardrobe/items` fails → an inline error state appears in the grid area with a retry button; the tab bar and navigation remain functional

## Edge Cases

- User adds the same item twice (e.g., same T-shirt photo) → no duplicate detection in v1; both entries are stored separately; deduplication is a future scope feature
- Inventory has 200+ items → the `LazyVGrid` handles this efficiently; no pagination in v1; items load as the user scrolls

## Dependencies

- S-005-001 (dashboard grid)
- S-005-007 (chat extraction pipeline)
- S-003-001 (memory storage)

## Technical Requirements

- **`WardrobeModuleView.swift`** — top-level view with `TabView` (segmented style); two child views: `WardrobeInventoryView` and `WardrobeOutfitsView`
- **`WardrobeItem` model:** `(id UUID, user_id, photo_url: String?, category: WardrobeCategory, notes: String?, created_at, source: ItemSource (.manual | .chat_sync))`
- **`OutfitEntry` model:** `(id UUID, user_id, photo_url: String, noxis_rating: String, noxis_judgment_full: String, chat_message_id: UUID, created_at)`
- **Add item sheet:** `WardrobeAddItemSheet.swift` — `@State var selectedImage: UIImage?`, `@State var category: WardrobeCategory`, `@State var notes: String`; calls `WardrobeViewModel.addItem()`
- **Backend routes:** `GET /api/v1/wardrobe/items`, `POST /api/v1/wardrobe/items`, `DELETE /api/v1/wardrobe/items/{id}`, `GET /api/v1/wardrobe/outfits`
- **Photo storage:** images uploaded to the same media endpoint as chat images (`/api/v1/media/upload`); returned URL stored on the item record
- **`WardrobeCategory` enum:** `tshirt | pants | shoes | jacket | accessory | other`
- **Outfit auto-entry:** `OutfitEntry` records are written by S-005-007 — this story only reads and displays them

## Future Scope

- Outfit builder: combine items from inventory into a proposed outfit and ask Noxis to rate it
- Duplicate detection: flag when the same or very similar item is added twice
- Wear frequency tracking: mark items as "worn today" to build a usage heatmap
- Archive items that haven't been worn in 90 days with a "donate?" prompt from Noxis

## Do Not Do

- In this story, do not build the chat extraction sync logic — that is S-005-007
- Do not implement outfit rating within this story — Noxis rates outfits via the chat pipeline; this screen only displays existing outfit entries
- Do not add manual outfit entry in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

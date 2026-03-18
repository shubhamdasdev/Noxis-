# Story: Image Classification & Module-Aware Judgment

---

**ID:** S-004-003
**Epic:** E-004
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P1
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to automatically understand what I've photographed and respond with the right kind of judgment**, so that **I don't have to explain whether I'm asking about my outfit, my meal, or my body — Noxis figures it out and responds in the right register**.

## Goal

When a photo arrives in the chat pipeline, a classification pass runs before the response is generated. The image is categorized (outfit / meal / body / environment / other) and the appropriate judgment framework from SOUL.md is applied. The final response reflects the category-specific criteria: outfits get style and occasion assessment, meals get protein-first nutritional judgment, body photos get fitness progress assessment.

## Why / Rationale

Without classification, every photo triggers a generic response. The entire visual judgment system depends on routing the image to the correct evaluation lens. A user who sends a plate of food should not get a style critique. The classification step is what makes Noxis feel intelligent rather than reactive.

## Functional Requirements

- When an incoming message contains an image, a classification step runs as part of the orchestration pipeline (S-004-005) before the AI response is generated
- Classification uses the AI provider's vision capability to determine the primary category of the image: `outfit`, `meal`, `body`, `environment`, or `other`
- Classification is a lightweight call (low token budget) that returns a JSON object: `{ "category": "outfit", "confidence": 0.94, "sub_tags": ["casual", "streetwear"] }`
- The category result is used to load the correct module context into the system prompt for the main response call:
  - `outfit` → wardrobe judgment lens: fit quality, color coordination, occasion appropriateness, styling recommendations
  - `meal` → food judgment lens: protein presence and quality, whole food vs. processed ratio, meal timing if known from context
  - `body` → fitness judgment lens: visible muscle development, posture, overall physical presentation — delivered with the brother tone regardless of selected tone mode
  - `environment` → contextual note only; Noxis may comment briefly but does not judge environments
  - `other` → no specific lens; Noxis responds to the image naturally as part of the conversation
- Confidence below 0.60 → Noxis asks the user what the photo is before applying any judgment lens
- The category is stored on the message record and used to sync data to the appropriate module (S-005-007)
- If the image contains recognizable people other than a solo subject, Noxis acknowledges the photo but does not judge other people — only the user's own presence in the frame

## Prerequisites

- S-004-002 (image input — image must be uploaded and available as a URL before classification can run)
- S-002-003 (module routing — classification result feeds into the module router)
- S-002-004 (judgment framework — category-specific lenses must be defined in SOUL.md)

## Acceptance Criteria

- [ ] Given the user sends a photo of an outfit, when the classification runs, then the response applies the wardrobe judgment lens — commenting on fit, color, and occasion appropriateness per the SOUL.md framework
- [ ] Given the user sends a photo of a meal, when the classification runs, then the response applies the food judgment lens — leading with protein assessment and overall food quality
- [ ] Given the user sends a body photo, when the classification runs, then the response applies the fitness judgment lens in brother tone regardless of the user's configured tone mode
- [ ] Given the classification confidence is below 0.60, when the response is generated, then Noxis asks the user to clarify what the photo shows before giving judgment
- [ ] Given the classification completes, when the message record is stored, then the `image_category` field is populated with the returned category string
- [ ] Given the image contains multiple visible people, when the response is generated, then Noxis only references the user's presence and does not pass judgment on other individuals in the frame
- [ ] Given the image is classified as `other`, when the response is generated, then Noxis responds to the image naturally without forcing a judgment framework onto it

## Negative Scenarios

- AI vision call for classification fails or times out → fall through to the `other` category; proceed with a general response; log the classification failure for monitoring; do not expose the failure to the user
- Image is blurry, dark, or too small to interpret → vision model returns low confidence; Noxis responds with "This one's a bit hard to read — can you send a clearer shot?"

## Edge Cases

- User sends a photo of text (a screenshot, a label, a menu) → classified as `other`; Noxis reads and responds to the text content naturally
- User sends a photo of food inside a gym bag alongside gym gear → the primary category is determined by the most prominent subject; if ambiguous between two categories both above 0.60, the first classification result wins

## Dependencies

- S-004-002 (image input — provides uploaded image URL)
- S-002-003 (module routing)
- S-002-004 (judgment framework)

## Technical Requirements

- **Classification call:** separate lightweight AI provider call before the main response call; model: `gpt-4o-mini` or equivalent vision-capable model; max tokens: 100; response format: JSON via function calling or structured output
- **Classification prompt:** "Analyze this image and return a JSON object with: category (one of: outfit, meal, body, environment, other), confidence (0.0-1.0), sub_tags (array of up to 3 descriptive strings). Be concise."
- **Module context injection:** a `buildModuleContext(category: ImageCategory) -> String` function returns the appropriate judgment lens text to inject into the main system prompt
- **Message model extension:** `ChatMessage` gains `imageCategory: ImageCategory?` enum field
- **Body photo privacy:** classification prompt includes instruction: "If the image contains a person, only assess the primary subject — do not comment on secondary individuals"
- **Timeout:** classification call has a 10-second hard timeout; failure triggers fallback to `other` category
- **Backend endpoint:** `POST /api/v1/classify` accepts `{ media_url: String }` returns classification JSON; client sends this before initiating the main chat completion

## Future Scope

- Multi-label classification (an image can belong to two categories simultaneously, e.g. body + outfit)
- User-correctable classification: "Actually this is a meal, not that" → user can tap to override the category
- Automatic tagging of classified images into module galleries without any extra user action

## Do Not Do

- In this story, do not implement the dashboard sync logic — that is S-005-007
- Do not build the module detail screens — those are E-005 stories
- Do not implement real-time object detection or on-device ML — use the AI provider's vision API only in v1

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

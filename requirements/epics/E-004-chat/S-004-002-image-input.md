# Story: Image Input — Camera Button & Photo Send

---

**ID:** S-004-002
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

As a **user**, I want **to send photos directly in the chat input bar**, so that **I can get Noxis's judgment on what I'm wearing, eating, or doing without having to describe it in words**.

## Goal

The camera icon in the chat input bar opens either the photo library or the camera. The selected image is compressed, attached as a visual message bubble, and sent alongside any typed text. The pipeline hands the image to the AI provider for vision-based classification and response.

## Why / Rationale

A significant portion of Noxis's value comes from photo-based judgment: outfit rating, meal quality assessment, body progress. Without photo input in chat, these features require the user to manually describe what they're looking at — which defeats the purpose. This is the input mechanism that unlocks the entire visual judgment layer.

## Functional Requirements

- The camera icon in the chat input bar (SF Symbol `photo`) opens an action sheet with two options: "Choose from Library" and "Take Photo"
- "Choose from Library" presents `PHPickerViewController` with `selectionLimit: 1` and `filter: .images`
- "Take Photo" presents `UIImagePickerController` with `sourceType: .camera`; if camera permission is denied, show a settings-deep-link alert
- Selected or captured image is displayed as a preview thumbnail in the input bar (above or inline) before sending — user can remove it by tapping the X on the thumbnail
- When the user taps Send with an image attached, the image is compressed to JPEG at 0.7 quality; if the result exceeds 10MB, the compression quality is lowered iteratively (0.5, 0.3) until under 10MB; if still over 10MB at 0.3, the image is rejected with a clear error message
- The sent image appears in the chat as a full-width (bounded to 260pt wide) image card inside a `GlassCard` frame, right-aligned, with a subtle rounded corner
- If text was also typed in the input field, the text and image are sent as a combined message — the image card appears above the text bubble
- A spinner overlay appears on the image card while it is uploading to the backend storage; on success, the spinner is replaced by the image; on failure, a retry button overlays the card
- Images are uploaded to the backend as multipart form data; the backend returns an image URL that is stored with the message record
- After the image is sent, the Noxis response pipeline (S-004-005) receives both the image URL and any accompanying text

## Prerequisites

- S-007-001 (bottom tab bar — disambiguates from the center camera tab which is a different entry point)
- S-007-003 (design system — GlassCard for image bubble framing)

## Acceptance Criteria

- [ ] Given the user taps the camera icon in the input bar, when the action sheet opens, then "Choose from Library" and "Take Photo" are the only two options shown
- [ ] Given the user selects a photo from the library, when the image is chosen, then a thumbnail preview appears in the input bar with an X dismiss button before the message is sent
- [ ] Given the user taps Send with an image attached, when the message is submitted, then the image appears in the chat list as a right-aligned glass-framed card with a spinner overlay while uploading
- [ ] Given an image is larger than 10MB after initial compression, when the compression loop runs, then quality is reduced iteratively and only rejected with an error message if still over 10MB at minimum quality
- [ ] Given camera permission has been denied by the user, when they tap "Take Photo", then an alert appears explaining the permission requirement with a "Go to Settings" button that deep-links to the iOS Settings app
- [ ] Given the image upload fails, when the error occurs, then a retry button appears overlaid on the image card and the pipeline does not proceed until a successful upload completes or the user cancels
- [ ] Given the user typed text alongside an image, when the message is sent, then the image card and text bubble both appear in the chat in the correct order (image above text)

## Negative Scenarios

- User selects a non-image file type (e.g. a video via the library picker) → the picker configuration filters to images only; non-image content is never selectable
- Upload times out (>30 seconds) → the spinner is replaced by a timeout error indicator with "Retry" and "Cancel" options; cancel removes the unsent image card from the list

## Edge Cases

- User selects a very large RAW or HEIC photo (40MB+) → HEIC is converted to JPEG by `UIImage(data:)` before compression; if the initial JPEG conversion itself exceeds 10MB, the compression loop runs as normal
- User rapidly taps the camera button multiple times → the picker is presented once; subsequent taps are ignored until the picker is dismissed

## Dependencies

- S-007-001 (tab bar — clarifies which camera icon this is)
- S-007-003 (design system — GlassCard image bubble)

## Technical Requirements

- **`ImagePickerCoordinator.swift`** — `UIViewControllerRepresentable` wrapping `PHPickerViewController` (library) and `UIImagePickerController` (camera); delegate forwards `UIImage` back to the view model
- **Image compression:** `UIImage.jpegData(compressionQuality:)` loop with qualities `[0.7, 0.5, 0.3]`; size check after each pass
- **Upload:** `URLSession` multipart POST to `/api/v1/media/upload`; returns `{ url: String, media_id: String }`; stored on `ChatMessage.mediaUrl`
- **`PHPickerConfiguration`:** `configuration.filter = .images`, `configuration.selectionLimit = 1`, `configuration.preferredAssetRepresentationMode = .current`
- **Camera permission:** `AVCaptureDevice.requestAccess(for: .video)` before presenting picker; handle `.denied` and `.restricted` states
- **Message model extension:** `ChatMessage` gains optional `mediaUrl: String?` and `mediaStatus: MediaStatus (.uploading | .uploaded | .failed)`
- **Thumbnail preview state:** local `@State var pendingImage: UIImage?` in `ChatInputBar`; cleared on send or dismiss

## Future Scope

- Multi-image selection (select up to 4 photos in one message) for outfit comparisons
- Video support (short clips, max 15 seconds) for form-check in the gym module
- Drag-and-drop image attachment from other iOS apps

## Do Not Do

- In this story, do not implement the image classification logic — that is S-004-003
- Do not implement the camera tab bar entry point — that is S-007-001
- Do not apply any AI vision call in this story — only handle capture, compression, upload, and display

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

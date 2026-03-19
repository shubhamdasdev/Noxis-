# QA Test Plan — Glass + Black Design System — Theme, Typography, Components

---

**Plan ID:** T-007-003
**Story:** S-007-003
**Epic:** E-007
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the complete visual design system implementation for Noxis, including all color tokens, typography definitions, glass panel components, spacing scale, and interaction states. Testing confirms that every UI surface uses tokens from `DesignSystem.swift` with no hardcoded hex values, that the app is permanently locked to dark/black mode, that glass blur panels render correctly, that tap animations fire at 0.96 scale, and that loading shimmer states appear in place of any spinner. Coverage spans all acceptance criteria plus the light-mode negative scenario and the device/screen edge cases.

## Out of Scope

- Design token export to Figma or Pencil (Future Scope)
- Themed accent color variations (Future Scope)
- Building any specific screen content — this plan covers the component library and tokens only (Do Not Do)
- Light mode support — Noxis is always black (Do Not Do)

## Prerequisites

- `DesignSystem.swift` is committed with all color, typography, and spacing tokens defined
- All eight core components (`GlassCard`, `GlassButton`, `GlassInput`, `GlassTabBar`, `NoxisAvatar`, `MessageBubble`, `StreakBadge`, `ModuleCard`) are implemented
- A component gallery or test harness screen is available to exercise all components in isolation
- App can be run on a physical iPhone (or high-fidelity simulator) to observe blur effects
- `Info.plist` has `UIUserInterfaceStyle = Dark`

---

## Core Test Flow

### TC-007-003-001: Full design system render — all tokens, components, and states in a single pass
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006
**Dependencies:** None

**Preconditions:**
- App is installed on a device (or simulator) with system appearance set to Light Mode to confirm override
- A component gallery screen is available showing all eight core components plus loading and interaction states
- Source code is accessible for token inspection

**Steps:**
1. Launch the app with the device system appearance set to Light Mode
2. Verify the app background renders as pure black (#000000) — no white or grey background at any level of the UI
3. Navigate to the component gallery screen
4. Observe each `GlassCard` component — confirm frosted blur effect is visible and a subtle white stroke border (rgba(255,255,255,0.12)) is present
5. Observe all text elements on screen — confirm SF Pro typeface is used at correct sizes: Display 28pt Bold, Headline 20pt Semibold, Body 16pt Regular, Caption 13pt Regular, Mono 14pt Regular
6. Tap a `GlassButton` — confirm a 0.96 scale spring animation plays with a momentary brightness reduction; no abrupt jump
7. Tap a `GlassCard` — confirm the same 0.96 scale spring animation fires
8. Trigger a loading state on any glass component — confirm shimmer animation plays across the glass surface; confirm no spinner, activity indicator, or loading wheel appears
9. Open `DesignSystem.swift` in the source tree — confirm all color tokens (`background`, `surface`, `glass`, `glassStroke`, `primary`, `secondary`, `muted`, `accent`, `success`, `destructive`) are defined; open any one component file (e.g. `GlassCard.swift`) — confirm it references token names, not hardcoded hex strings

**Expected Result:**
- Background is #000000 on every screen regardless of system appearance setting
- Glass panels show frosted blur with rgba(255,255,255,0.12) stroke border and no shadow
- All tap interactions produce a visible 0.96 scale spring animation
- Loading states show shimmer only — zero spinners anywhere in the app
- All text matches SF Pro at the defined point sizes and weights
- No hardcoded hex values appear in any component `.swift` file; all references are to named tokens in `DesignSystem.swift`

**Failure Indicators:**
- Any white, grey, or off-black background visible at any point
- Glass panels lacking blur or lacking a visible border stroke
- Tap produces no animation, or animation is a fade/opacity change rather than a scale spring
- A spinner, `ProgressView`, or `ActivityIndicator` appears anywhere in a loading state
- Text renders in a non-SF Pro font or at incorrect sizes (e.g. Body at 14pt instead of 16pt)
- Hardcoded hex string (e.g. `"#F0E6D3"` or `Color(hex: "...")`) found in any component file

---

## Sub Flows

### TC-007-003-002: Device set to Light Mode — app stays black
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-001
**Dependencies:** None

**Preconditions:**
- Device system appearance is set to Light Mode in iOS Settings

**Steps:**
1. Launch the app
2. Observe the launch screen and every primary screen
3. Toggle system appearance between Light and Dark while the app is backgrounded, then foreground the app

**Expected Result:**
- App renders pure black (#000000) background at all times regardless of system appearance toggle
- No component switches to a light background, white surface, or grey fill at any point

**Failure Indicators:**
- Any screen or component adopts a light background when system is in Light Mode
- Toggling system appearance changes any visual element in the app

### TC-007-003-003: Older device (non-ProMotion) — animations play at 60fps
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-003
**Dependencies:** None

**Preconditions:**
- App is running on a device or simulator without ProMotion (e.g. iPhone SE, iPhone 14 standard)

**Steps:**
1. Open the component gallery
2. Tap multiple buttons and cards in rapid succession
3. Trigger a loading shimmer state
4. Observe animation smoothness

**Expected Result:**
- Scale spring animations on tap play smoothly at 60fps — no dropped frames or jank visible to the naked eye
- Shimmer animation on glass surface plays continuously without stutter
- No frame-rate-adaptive behavior (no animation skipping or simplification compared to ProMotion device)

**Failure Indicators:**
- Animations visibly stutter or skip frames on a 60fps device
- Shimmer pauses or flickers on older hardware

### TC-007-003-004: Small screen (iPhone SE) — spacing scale compresses without clipping
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-001, AC-002, AC-005
**Dependencies:** None

**Preconditions:**
- App is running on an iPhone SE (3rd gen) simulator or device (375×667 logical points)

**Steps:**
1. Open the component gallery showing all eight components
2. Scroll through the full gallery
3. Verify no component is horizontally clipped or cut off by the screen edge
4. Verify text does not fall below the 12pt dynamic type floor

**Expected Result:**
- All components are fully visible within the screen width
- Spacing between components compresses proportionally — content does not overflow horizontally
- No text renders smaller than 12pt on the smallest device

**Failure Indicators:**
- Any component is horizontally clipped or partially off-screen
- Text is visually illegible due to size falling below 12pt
- Content requires horizontal scrolling where none is intended

### TC-007-003-005: Token reference audit — no hardcoded values in component files
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-006
**Dependencies:** None

**Preconditions:**
- Full codebase is available for static inspection
- `DesignSystem.swift` is the single source of truth for tokens

**Steps:**
1. Search all component `.swift` files for hardcoded hex strings (pattern: `#[0-9A-Fa-f]{3,8}` or `Color(red:` with literal float values matching any palette color)
2. Search for hardcoded font size literals (e.g. `.font(.system(size: 28))`) that bypass the `DesignSystem` typography definitions
3. Search for hardcoded spacing literals (e.g. `.padding(17)`) that are not on the 4pt grid

**Expected Result:**
- Zero hex color strings found in component files outside of `DesignSystem.swift`
- Zero hardcoded font size values found in component files — all reference the typography constants defined in `DesignSystem.swift`
- Spacing values used are exclusively from the defined scale: 4, 8, 12, 16, 24, 32, 48pt

**Failure Indicators:**
- Any hardcoded color, font size, or off-grid spacing value found in a component file

---

## Automation Notes

- **Static analysis:** A lint rule or custom script can grep component files for hex strings and raw `Color(red:green:blue:)` literals; run this in CI as a build step
- **Snapshot testing:** Use `swift-snapshot-testing` to capture reference renders of each component in dark mode; diff against baseline on each PR to catch visual regressions
- **Animation testing:** Not automatable via XCTest UI assertions — tap animations and shimmer must be verified by human observation on device; flag as manual-only in the test matrix
- **Flakiness risk:** Blur effect rendering in the simulator is sometimes inconsistent — always validate glass panel appearance on a physical device before marking AC-002 passed
- **Token audit:** Automate with a CI script using `grep -r '#[0-9A-Fa-f]' Sources/Components/` excluding `DesignSystem.swift` — fail the build if matches are found

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

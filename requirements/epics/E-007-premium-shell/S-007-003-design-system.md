# Story: Glass + Black Design System — Theme, Typography, Components

---

**ID:** S-007-003
**Epic:** E-007
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **the entire app to feel visually cohesive, premium, and private**, so that **Noxis looks and feels like a high-end tool built for a specific purpose — not a generic productivity app**.

## Goal

A complete design system is implemented covering colors, typography, glass components, spacing, and interaction states. Every UI element in the app uses tokens from this system — no one-off styles. The black + glass aesthetic is consistent from splash screen to deepest settings view.

## Why / Rationale

The design is part of the product identity. A man who cares about how he presents himself will not trust lifestyle advice from an app that looks cheap. The premium visual language must be established as the foundation before any feature screen is built.

## Functional Requirements

**Color palette:**
- `background`: #000000 (pure black)
- `surface`: #0A0A0A (slightly elevated surface)
- `glass`: UIBlurEffect dark + 8% white overlay (frosted glass panels)
- `glassStroke`: rgba(255,255,255,0.12) (glass border)
- `primary`: #FFFFFF (primary text, active icons)
- `secondary`: rgba(255,255,255,0.60) (secondary text, inactive icons)
- `muted`: rgba(255,255,255,0.30) (placeholder text, disabled states)
- `accent`: #F0E6D3 (warm off-white — used sparingly for highlights, CTAs)
- `success`: #4CAF50 (streaks, confirmations)
- `destructive`: #FF3B30 (iOS red — errors, delete)

**Typography:**
- **Display**: SF Pro Display — Bold, 28pt (screen titles)
- **Headline**: SF Pro Display — Semibold, 20pt (section headers)
- **Body**: SF Pro Text — Regular, 16pt (chat messages, module content)
- **Caption**: SF Pro Text — Regular, 13pt (timestamps, metadata)
- **Mono**: SF Mono — Regular, 14pt (data values, stats)
- All text uses dynamic type with a minimum floor (no text below 12pt)

**Glass panel component:**
- Background: `UIVisualEffectView(effect: UIBlurEffect(style: .systemUltraThinMaterialDark))`
- Overlay: 8% white fill on top of blur
- Border: 1pt stroke, `glassStroke` color
- Corner radius: 16pt (cards), 12pt (inputs), 24pt (large panels)
- Shadow: none — glass panels float via blur, not shadow

**Core components defined:**
- `GlassCard` — the primary surface container used across all screens
- `GlassButton` — primary CTA with glass background + accent border
- `GlassInput` — text input field with glass background
- `GlassTabBar` — bottom tab bar (used by S-007-001)
- `NoxisAvatar` — Noxis identity mark used in chat and loading states
- `MessageBubble` — user and Noxis chat bubbles (user: glass card right-aligned; Noxis: no background, left-aligned)
- `StreakBadge` — pill badge for habit streaks
- `ModuleCard` — card used in the dashboard grid

**Spacing scale:** 4pt base grid — 4, 8, 12, 16, 24, 32, 48pt

**Interaction states:**
- Tap: 0.96 scale spring animation + momentary brightness reduction
- Swipe: spring physics, rubber-band at edges
- Loading: shimmer animation on glass background (no spinner)

## Prerequisites

- None — this is the first story to implement

## Acceptance Criteria

- [ ] Given any screen in the app, when rendered, then the background is pure black (#000000) with no white or grey backgrounds
- [ ] Given a glass panel is rendered, when displayed, then it shows a frosted blur effect with a subtle white stroke border
- [ ] Given a user taps any button or card, when tapped, then a 0.96 scale spring animation plays
- [ ] Given a loading state, when displayed, then a shimmer animation plays on the glass surface — no spinning indicators
- [ ] Given any text in the app, when rendered, then it uses SF Pro typeface at the defined sizes and weights
- [ ] Given the design system tokens file exists, when any component is built, then it references tokens — no hardcoded hex values in component files

## Negative Scenarios

- Device in light mode → app ignores system appearance and always renders in dark/black mode (Noxis is always dark)

## Edge Cases

- Older devices without ProMotion display → animations still play at 60fps; no adaptive frame rate needed in v1
- Small screen (iPhone SE) → spacing scale compresses gracefully; no content clipping

## Dependencies

- None

## Technical Requirements

- **`DesignSystem.swift`** — single file defining all color tokens, typography styles, and spacing constants as SwiftUI `Color`, `Font`, and `CGFloat` values
- **`GlassCard.swift`, `GlassButton.swift`** etc. — one file per core component; each accepts standard modifiers
- **No third-party UI libraries** — pure SwiftUI + UIKit where needed
- **Dark mode locked** — `UIUserInterfaceStyle.dark` forced in `Info.plist`

## Future Scope

- Design token export to Figma/Pencil for handoff
- Themed accent color variations (user-selectable in v3)

## Do Not Do

- In this story, do not build any screen content — only the component library and design tokens
- Do not support light mode — Noxis is always black

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

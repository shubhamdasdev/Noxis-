# Architecture Task: iOS Project Scaffold + Design System Foundation

---

**ID:** AT-001-001
**Release:** R-001
**Stage:** Backlog
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Extends:** None
**Depends On:** None
**Version:** 1.0

---

## What

Creates the Xcode project from scratch and establishes the `DesignSystem` foundation that every iOS coding agent in R-001 will import. Without this artifact, agents working on S-007-001 (tab bar), S-004-001 (chat), and every other iOS screen would each invent their own token values and component wrappers — producing irreconcilable inconsistencies. This AT produces a buildable, empty-screen iOS app with all design tokens, typography helpers, and base glass components in place so story agents can reference them by name from day one.

## Artifact Location

```
Noxis/
  Noxis.xcodeproj/
  Noxis/
    App/
      NoxisApp.swift              ← @main entry point, RootView placeholder
      AppRouter.swift             ← determineInitialTab() stub (returns .chat)
    DesignSystem/
      DesignSystem.swift          ← all color/typography/spacing tokens
      Components/
        GlassCard.swift
        GlassButton.swift
        GlassInput.swift
        EmptyState.swift
        ShimmerCard.swift
        Toast.swift
        BottomTabBar.swift        ← 5-tab shell with placeholder destinations
```

## Steps

1. `[Agent]` iOS 17+ Xcode project exists, compiles cleanly, and runs on iPhone simulator — blank black screen with `RootView` as placeholder (e.g. `pm new ios-project Noxis --min-ios 17`)

2. `[Agent]` `DesignSystem.swift` defines all tokens from `design.md` Visual Language section as Swift constants — every color (`Color.noxisBackground`, `Color.noxisAccent`, `Color.noxisGlass`, etc.), every typography style (`Font.noxisDisplay`, `Font.noxisHeadline`, `Font.noxisBody`, `Font.noxisCaption`), and every spacing/radius value

3. `[Agent]` `GlassCard.swift` — SwiftUI `ViewModifier` that applies glass surface fill, glass border, and 16pt corner radius; exposes `.glassCard()`, `.glassCardElevated()`, and `.glassCardSelected(accentGlow: Bool)` variants as defined in `design.md`

4. `[Agent]` `GlassButton.swift` — SwiftUI `View` accepting `label: String`, `style: GlassButtonStyle` (`.fullWidth` / `.compact`), `isLoading: Bool`, `isDisabled: Bool`; applies exact token values from design.md; disabled state uses 0.4 opacity

5. `[Agent]` `GlassInput.swift` — SwiftUI `View` wrapping `TextField`/`SecureField`; accepts `placeholder: String`, `isSecure: Bool`; applies glass fill, glass border, accent focus border; focus state managed with `@FocusState`

6. `[Agent]` `EmptyState.swift`, `ShimmerCard.swift`, `Toast.swift` — implement each component per design.md specification; `ShimmerCard` uses animated gradient overlay on a loop; `Toast` auto-dismisses after 3s with slide-up/fade-in animation

7. `[Agent]` `BottomTabBar.swift` — custom 5-tab bar (Chat · Modules · Camera · Daily · Profile) matching design.md specification exactly: `UIBlurEffect(style: .systemUltraThinMaterialDark)` background, center Camera button (56pt Accent circle, elevated 8pt, `camera.fill` icon), active/inactive tab states; each tab destination is a placeholder `Text` view — feature content is added by story agents

8. `[Agent]` `NoxisApp.swift` wires `RootView` with `BottomTabBar`; app builds and runs on iPhone 17 simulator with no warnings and no errors; tab bar visible, tapping tabs switches between placeholder screens, center camera button triggers `AVCaptureDevice.requestAccess` (permission check only at this stage)

## Acceptance Criteria

- [ ] `xcodebuild -scheme Noxis -destination 'platform=iOS Simulator,name=iPhone 17' build` exits 0 with zero errors and zero warnings
- [ ] All color tokens in `DesignSystem.swift` exactly match the hex values in `design.md` Visual Language section — no approximations
- [ ] All typography styles in `DesignSystem.swift` use system SF Pro at the exact point sizes and weights specified in `design.md`
- [ ] `.glassCard()`, `.glassCardElevated()`, `.glassCardSelected(accentGlow:)` modifiers exist and produce visually distinct surfaces when applied to a test view
- [ ] `GlassButton` renders correctly in `.fullWidth` and `.compact` styles; disabled state reduces opacity to 0.4; loading state shows `ProgressView` spinner
- [ ] `GlassInput` standard variant shows glass border on idle and accent border on focus; secure variant shows eye-toggle icon
- [ ] `BottomTabBar` renders exactly 5 items with center camera button elevated 8pt above bar baseline; tapping non-camera tabs switches active tab; tapping camera button requests camera permission
- [ ] `ShimmerCard` animates a gradient shimmer on a continuous loop when placed in a SwiftUI view
- [ ] App background is pure `#000000` on all simulator screenshots — no gray or off-white system backgrounds visible

## Depends On

None — this is the first task in R-001.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

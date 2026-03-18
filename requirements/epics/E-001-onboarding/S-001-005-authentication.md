# Story: Authentication (Email / Apple Sign-In)

---

**ID:** S-001-005
**Epic:** E-001
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **guest**, I want **to create an account or sign in with email or Apple**, so that **my memories, conversation history, and settings are saved and restored across sessions and devices**.

## Goal

Authentication is presented after tone selection and quick profile — not before. The user experiences the product personality first, then creates an account to save their progress. Email/password and Apple Sign-In are both supported. Session persists until the user signs out.

## Why / Rationale

Asking users to create an account before they've seen the product creates drop-off. By placing auth after the first two onboarding screens, the user has already selected their tone and answered quick profile questions — they have skin in the game and a reason to create an account to save it.

## Functional Requirements

**Placement in flow:**
- Auth screen appears after Quick Profile (S-001-002), before Conversational Onboarding (S-001-003)
- Heading: "Save your setup" (Headline)
- Subtext: "Create an account so Noxis can remember everything." (Body, secondary)

**Sign-in options:**
1. **Apple Sign-In** — native Apple button; uses `ASAuthorizationAppleIDProvider`; returns name (first time only) + email + identity token
2. **Email / Password** — two fields: Email, Password (min 8 chars); "Create account" CTA
3. **Existing account** — "Already have an account? Sign in" text link below → shows email + password fields with "Sign in" CTA

**Account creation (email):**
- Submits email + password to backend → creates account → returns JWT
- No email verification in v1 — account is active immediately
- On success: persist JWT locally, flush `OnboardingState` to user profile via API, proceed to Conversational Onboarding

**Apple Sign-In:**
- On success: persist JWT locally, flush `OnboardingState` to user profile, proceed to Conversational Onboarding
- If Apple credential already exists (returning user) → sign in, load existing profile, skip to Chat

**Sign in (existing account):**
- Email + password → validate → return JWT → restore existing user profile → navigate to Chat (skip onboarding)

**Session persistence:**
- JWT stored in iOS Keychain
- On app launch, valid JWT → skip auth and onboarding entirely
- Expired/invalid JWT → clear Keychain, route to Welcome screen

## Prerequisites

- S-001-002 (Quick Profile screen — auth follows this)
- S-007-003 (Design system)

## Acceptance Criteria

- [ ] Given a new user completes Quick Profile and taps Continue, when the auth screen renders, then Apple Sign-In and email/password options are both visible
- [ ] Given a user signs in with Apple, when authentication succeeds, then a JWT is stored in Keychain and the user proceeds to Conversational Onboarding
- [ ] Given a user creates an email account, when the form is submitted with valid credentials, then an account is created, JWT stored, and they proceed to Conversational Onboarding
- [ ] Given an existing user opens the app with a valid JWT in Keychain, when the app launches, then they skip auth and onboarding and go directly to Chat or Daily (per S-007-004)
- [ ] Given a user enters an already-registered email, when submitted, then an inline error appears: "An account with this email already exists. Sign in instead."
- [ ] Given a user enters a password under 8 characters, when they try to submit, then the button remains disabled with inline validation error

## Negative Scenarios

- Network failure during auth → error toast: "Connection failed. Please try again." — form data preserved
- Apple Sign-In cancelled by user → return to auth screen with no error shown

## Edge Cases

- User's Apple ID email is hidden (private relay) → accept the relay email; store it as their account email
- JWT expires mid-session → silently refresh token in background; if refresh fails, sign the user out and route to Welcome with message: "Your session expired. Please sign in again."

## Dependencies

- S-007-003

## Technical Requirements

- **`AuthView.swift`** — SwiftUI view with Apple + email options
- **`AuthManager.swift`** — handles JWT storage (Keychain), token refresh, and session state
- **Backend API: POST /auth/register** — email + password → creates user, returns JWT
- **Backend API: POST /auth/login** — email + password → validates, returns JWT
- **Backend API: POST /auth/apple** — Apple identity token → validates with Apple, creates/retrieves user, returns JWT
- **Keychain wrapper** — use `KeychainAccess` or equivalent for secure JWT storage

## Future Scope

- Google Sign-In (v2)
- Biometric re-authentication (Face ID) for app re-open after background (v2)

## Do Not Do

- In this story, do not implement email verification — account is active immediately on registration
- Do not show the auth screen before Quick Profile — placement in the flow is deliberate

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

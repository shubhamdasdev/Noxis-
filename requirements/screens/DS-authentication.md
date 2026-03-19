# Design Spec — Screen: Authentication

**Version:** 1.0
**Stage:** Draft
**Flow:** UF-001
**Screen:** Screen: Authentication
**Stories:** S-001-005
**Modes:** sign-up, sign-in
**Components:** GlassInput, GlassButton, AppleSignInButton, Toast
**Updated:** 2026-03-18

---

## Screen: Authentication

**Story context:** User creates an account (or signs in) to save the tone and profile they just set up. Two modes share the same screen: sign-up (default, new users) and sign-in (existing users via the text link).

### Layout & Hierarchy

1. Primary: Header + subtext — top of screen, 16pt horizontal padding
2. Secondary: Apple Sign-In button — below header; highest visual weight of the two auth options
3. Tertiary: "or" divider — between Apple and email options
4. Quaternary: Email + Password fields + primary CTA — email/password path
5. Quinary: Mode-switch link — bottom, toggles between sign-up and sign-in

### Components

| Element | Component | Notes |
|---------|-----------|-------|
| Screen header | Text (Headline, white) | |
| Subtext | Text (Body, Text secondary) | |
| Apple Sign-In button | AppleSignInButton | Native `ASAuthorizationAppleIDButton(type: .default, style: .black)` |
| "or" divider | Text (Caption, Text secondary) with `Divider()` lines left and right | Horizontal rule + centered "or" label |
| Email field | GlassInput (standard) | Keyboard: `.emailAddress`; autocapitalization: `.none` |
| Password field | GlassInput (secure) | Eye-toggle reveals/hides; min 8 chars enforced client-side |
| Primary CTA | GlassButton (full-width) | Label changes per mode (see Copy table) |
| Mode-switch link | Text (Caption, Accent color) | Tappable text; no border |
| Inline field error | Text (Caption, Destructive) | Appears below relevant field; no modal |

### Copy

| Element | Sign-up mode | Sign-in mode |
|---------|-------------|-------------|
| Screen header | Save your setup | Welcome back |
| Subtext | Create an account so Noxis can remember everything. | Sign in to continue. |
| Email placeholder | Email | Email |
| Password placeholder | Password (min 8 characters) | Password |
| Primary CTA | Create account | Sign in |
| Mode-switch link | Already have an account? Sign in | Don't have an account? Create one |
| Duplicate email error | An account with this email already exists. Sign in instead. | — |
| Password too short | Password must be at least 8 characters. | — |
| Network error Toast | Connection failed. Please try again. | Connection failed. Please try again. |

### States

- **Default (sign-up):** Header "Save your setup"; Apple button + email/password fields + "Create account" CTA active; mode-switch link shows "Already have an account? Sign in"
- **Default (sign-in):** Header "Welcome back"; Apple button + email/password + "Sign in" CTA; mode-switch shows "Don't have an account? Create one"
- **Disabled CTA:** "Create account" / "Sign in" button is disabled when email is empty or password is < 8 chars
- **Loading:** CTA shows GlassButton loading state (spinner replaces label); fields disabled during request
- **Field error:** Inline error text (Caption, Destructive) appears below the relevant field; field border upgrades to Destructive color
- **Network error:** Toast (error variant) dismisses after 3s; form data preserved exactly as entered

### References

None — standard component library behavior applies.

### Interaction Notes

- Tapping the mode-switch link toggles between sign-up and sign-in modes in-place; no navigation push; form fields clear on mode switch
- Apple Sign-In triggers `.fullScreenCover` presenting the native Apple auth sheet (managed by `ASAuthorizationController`)
- On Apple Sign-In cancel: overlay dismissed, auth screen restored, no error shown
- Password field: eye-toggle icon is `eye` / `eye.slash` (SF Symbols); tap reveals/hides password text; 44pt hit area

---

## For Coding Agents

Read this file alongside `S-001-005-authentication.md` and `design.md` before implementing any screen in this story:

1. **Components** — use exactly the components listed in each screen's Components table; do not substitute
2. **Copy** — use the exact copy in the Copy table; do not paraphrase button labels or error messages
3. **States** — implement all four states (default, loading, error, field error) for every screen that lists them
4. **References** — the reference notes describe what to adopt, not what to copy wholesale

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

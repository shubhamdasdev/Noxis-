# QA Test Plan — Authentication (Email / Apple Sign-In)

---

**Plan ID:** T-001-005
**Story:** S-001-005
**Epic:** E-001
**Stage:** Draft
**Created:** 2026-03-18
**Updated:** 2026-03-18

---

## Scope

This plan validates the authentication screen and all associated flows: Apple Sign-In (new user and returning user), email/password account creation, existing account sign-in, JWT storage in Keychain, session persistence across cold launches, duplicate email inline error, password length inline validation, network failure handling, Apple Sign-In cancellation, Apple private relay email handling, and mid-session JWT expiry with silent refresh and fallback sign-out. The auth screen must appear after Quick Profile and before Conversational Onboarding.

## Out of Scope

- Email verification — account is active immediately, no verification email (Do Not Do)
- Showing the auth screen before Quick Profile — placement is deliberate and tested in the flow (Do Not Do)
- Google Sign-In (Future Scope)
- Face ID biometric re-authentication (Future Scope)

## Prerequisites

- S-001-002 (Quick Profile screen is complete; auth screen appears after it)
- S-007-003 (Design system implemented)
- Backend APIs are running: `POST /auth/register`, `POST /auth/login`, `POST /auth/apple`
- An Apple Developer account with Sign In with Apple capability configured
- A test Apple ID and a test email account are available
- Keychain can be cleared between test runs (via `Security.framework` or test helper)
- A registered test email account exists in the backend for the "existing account" test cases

---

## Core Test Flow

### TC-001-005-001: Full auth flow — new user email signup, JWT persisted, proceeds to conversational onboarding
**Type:** E2E
**Priority:** P0
**AC Covered:** AC-001, AC-002, AC-003, AC-004, AC-005, AC-006
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- User has completed Quick Profile (any selections) and tapped Continue
- The auth screen appears as the next navigation step
- No Keychain JWT is present
- A unique test email address is ready (not previously registered)

**Steps:**
1. Observe the auth screen — confirm heading "Save your setup" in Headline style, subtext "Create an account so Noxis can remember everything." in Body secondary, Apple Sign-In button visible, email and password fields visible, and "Already have an account? Sign in" text link visible
2. Tap the email field — enter the unique test email address
3. Tap the password field — enter a 7-character password (e.g. "Pass123")
4. Observe the form — confirm the "Create account" CTA remains disabled (or submit is prevented) and an inline validation error appears indicating the password is too short (minimum 8 characters)
5. Clear the password field and enter an 8-character password ("Pass1234")
6. Confirm the "Create account" CTA becomes active
7. Tap "Create account"
8. Confirm the backend call `POST /auth/register` is made with the correct email and password
9. Confirm a JWT is received and stored in iOS Keychain (verify via test helper or Keychain inspection)
10. Confirm `OnboardingState` (tone, quick profile selections) is flushed to the user profile via API
11. Confirm the app navigates to Conversational Onboarding
12. Force-quit and cold-launch the app — confirm the app skips auth and onboarding entirely and navigates to Chat or Daily per the S-007-004 routing logic (valid JWT found in Keychain)
13. Confirm the auth screen is not shown on this second launch

**Expected Result:**
- Auth screen layout matches the spec: heading, subtext, Apple button, email/password fields, existing-account link
- Password under 8 characters: "Create account" CTA is disabled and inline error indicates minimum length requirement
- Password of 8+ characters: CTA becomes active; form submits correctly
- On success: JWT is stored in Keychain, `OnboardingState` is flushed, app navigates to Conversational Onboarding
- On subsequent cold launch with valid JWT: auth and onboarding are both skipped; user lands in Chat or Daily

**Failure Indicators:**
- Auth screen does not appear after Quick Profile (appears at wrong point in the flow, or not at all)
- Password shorter than 8 characters is accepted without an inline error or CTA disable
- JWT is not stored in Keychain after successful registration
- `OnboardingState` is not flushed to the user profile
- App does not navigate to Conversational Onboarding after registration
- Valid JWT on second launch does not bypass auth — user is sent to auth or onboarding again

### TC-001-005-002: Apple Sign-In — new user, JWT persisted, proceeds to conversational onboarding
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- No existing account for the Apple ID being used
- Keychain is clear of any Noxis JWT

**Steps:**
1. On the auth screen, tap the native Apple Sign-In button
2. Complete the Apple authentication flow (provide Apple ID credentials if prompted)
3. Confirm the Apple credential (identity token + email) is sent to `POST /auth/apple`
4. Confirm a JWT is received and stored in Keychain
5. Confirm `OnboardingState` is flushed to the user profile
6. Confirm the app navigates to Conversational Onboarding

**Expected Result:**
- Apple Sign-In launches the native Apple authentication sheet
- On success: JWT stored in Keychain, `OnboardingState` flushed, navigation to Conversational Onboarding

**Failure Indicators:**
- Apple authentication sheet does not appear
- JWT is not stored after Apple auth success
- App navigates to the wrong destination after Apple Sign-In (e.g. Chat instead of Conversational Onboarding for a new user)

### TC-001-005-003: Apple Sign-In — returning user, loads existing profile, skips to Chat
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- An Apple ID credential already exists in the backend (previously registered)
- Keychain is clear of any Noxis JWT (simulating a fresh install on a new device)

**Steps:**
1. Tap the Apple Sign-In button
2. Complete Apple authentication with the already-registered Apple ID
3. Observe routing after sign-in

**Expected Result:**
- `POST /auth/apple` recognises the existing credential and returns the existing user's JWT
- Existing user profile is loaded
- App navigates directly to Chat (skipping Conversational Onboarding entirely)

**Failure Indicators:**
- Returning Apple user is routed to Conversational Onboarding instead of Chat
- A duplicate account is created for the existing Apple ID

### TC-001-005-004: Existing email sign-in — restores profile, navigates to Chat
**Type:** Happy Path
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- A registered test account with email and password exists in the backend
- Keychain is clear

**Steps:**
1. On the auth screen, tap "Already have an account? Sign in"
2. Confirm the form switches to sign-in mode (email + password fields, "Sign in" CTA)
3. Enter the registered email and correct password
4. Tap "Sign in"
5. Confirm `POST /auth/login` is called with the correct credentials
6. Confirm JWT is stored in Keychain and the app navigates to Chat (not Conversational Onboarding)

**Expected Result:**
- The "Already have an account? Sign in" link reveals or navigates to a sign-in form
- Successful sign-in returns a JWT, stores it in Keychain, and routes the user to Chat — onboarding is not shown
- Existing user profile is restored

**Failure Indicators:**
- "Already have an account? Sign in" link does nothing or navigates to the wrong screen
- Successful sign-in routes to Conversational Onboarding instead of Chat
- JWT is not stored after sign-in

### TC-001-005-005: Duplicate email registration — inline error shown
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-005
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- A registered account with email `test@example.com` already exists in the backend

**Steps:**
1. On the auth screen (Create account mode), enter `test@example.com` and a valid password ("Pass1234")
2. Tap "Create account"
3. Observe the response

**Expected Result:**
- An inline error appears: "An account with this email already exists. Sign in instead."
- The app does not navigate away from the auth screen
- No second account is created in the backend

**Failure Indicators:**
- A generic error or no error is shown
- The error uses different copy than specified ("An account with this email already exists. Sign in instead.")
- A duplicate account is created for the same email

### TC-001-005-006: Password under 8 characters — CTA disabled with inline validation error
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-006
**Dependencies:** S-007-003

**Preconditions:**
- User is on the auth screen in Create account mode

**Steps:**
1. Enter any email address
2. Enter a 7-character password ("Pass123")
3. Observe the "Create account" CTA state and any validation feedback

**Expected Result:**
- "Create account" CTA is disabled (cannot be tapped)
- An inline validation error appears adjacent to the password field indicating the 8-character minimum requirement

**Failure Indicators:**
- CTA is active with a sub-8-character password
- No inline validation error appears
- The form submits and `POST /auth/register` is called with a short password

### TC-001-005-007: Network failure during auth — error toast shown, form data preserved
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-003
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- Device has no network connectivity (Airplane Mode) or auth endpoint is mocked to return a network error

**Steps:**
1. Fill in a valid email and valid 8-character password
2. Tap "Create account"
3. Observe the UI response

**Expected Result:**
- An error toast appears: "Connection failed. Please try again."
- The email and password fields retain their values — the user does not have to re-enter credentials
- The app remains on the auth screen; no navigation occurs

**Failure Indicators:**
- App crashes on network failure
- Form fields are cleared after the network error
- No toast or error message appears
- Toast text differs from "Connection failed. Please try again."

### TC-001-005-008: Apple Sign-In cancelled — returns to auth screen with no error
**Type:** Negative
**Priority:** P1
**AC Covered:** AC-002
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- User is on the auth screen

**Steps:**
1. Tap the Apple Sign-In button
2. Cancel the Apple authentication sheet (tap Cancel or dismiss the sheet)
3. Observe the app state

**Expected Result:**
- The app returns to the auth screen
- No error message, alert, or toast is shown — the cancellation is silent
- The auth screen email and password fields (if previously filled) retain their values

**Failure Indicators:**
- An error message appears after cancelling Apple Sign-In
- The app navigates away from the auth screen
- The app crashes after the Apple sheet is cancelled

### TC-001-005-009: Apple private relay email — accepted and stored as account email
**Type:** Edge Case
**Priority:** P2
**AC Covered:** AC-002
**Dependencies:** S-001-002, S-007-003

**Preconditions:**
- Apple ID has "Hide My Email" enabled (private relay email will be provided, e.g. `xyz123@privaterelay.appleid.com`)

**Steps:**
1. Sign in with Apple using an Apple ID configured with Hide My Email
2. Observe the account created in the backend

**Expected Result:**
- The private relay email is accepted by `POST /auth/apple`
- The relay email is stored as the user's account email
- Authentication succeeds and JWT is returned normally
- No error about invalid email format is thrown

**Failure Indicators:**
- Backend rejects the private relay email format
- An error occurs during Apple Sign-In with Hide My Email enabled
- The relay email is stored incorrectly or causes a unique-constraint error

### TC-001-005-010: JWT expires mid-session — silent refresh, fallback sign-out on refresh failure
**Type:** Edge Case
**Priority:** P1
**AC Covered:** AC-004
**Dependencies:** S-007-003

**Preconditions:**
- User is authenticated and actively using the app
- JWT is configured to expire (mock a short-lived token for testing)

**Steps:**
1. Simulate JWT expiry while the user is on the Chat screen (inject an expired token into `AuthManager`)
2. Trigger any API call (send a chat message)
3. Confirm `AuthManager` attempts a silent token refresh in the background — the user sees no sign-out prompt at this stage
4. Mock the refresh endpoint to return success — confirm the new JWT is stored and the original API call succeeds

**Steps (refresh failure branch):**
5. Reset: inject an expired token; mock the refresh endpoint to return a failure
6. Trigger any API call
7. Confirm `AuthManager` clears the Keychain and signs the user out
8. Confirm the user is routed to the Welcome screen
9. Confirm a message appears: "Your session expired. Please sign in again."

**Expected Result:**
- On expiry: a silent background refresh is attempted; no UI disruption if successful
- On refresh success: new JWT stored, in-flight request retried, user remains on Chat screen
- On refresh failure: Keychain cleared, user routed to Welcome screen with the specified message

**Failure Indicators:**
- User is signed out immediately on JWT expiry without attempting a refresh
- Silent refresh fails to store the new JWT
- On refresh failure, no message appears or a different message appears than "Your session expired. Please sign in again."
- App crashes on JWT expiry or refresh failure

---

## Automation Notes

- **Keychain isolation:** Each test case must clear the Keychain at setup and teardown using a `KeychainTestHelper` — never share Keychain state between test runs
- **Apple Sign-In:** Full Apple Sign-In flow requires a physical device with a real Apple ID; use a backend mock (`POST /auth/apple` accepting a test identity token) for CI; the full UI flow must be covered in a manual device run
- **Inline validation:** Assert `XCUIElement.isEnabled` on the "Create account" button after entering a short password; assert `staticText` or `label` of the inline error element
- **Network failure simulation:** Use `URLProtocol` stubbing or a test-mode network layer that can return errors on demand — do not use Airplane Mode for automated tests as it affects the full device
- **JWT expiry:** Inject a mock `AuthManager` with a configurable token expiry and refresh outcome; test the refresh logic in unit tests rather than end-to-end to avoid timer-based flakiness
- **Flakiness risk:** Apple Sign-In requires a real Apple ID in manual testing — the flow depends on Apple's servers and can be slow; do not include the full UI test in a short-timeout CI suite; isolate backend-facing assertions via API mocks

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |

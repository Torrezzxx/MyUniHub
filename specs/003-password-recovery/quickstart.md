# Quickstart: Recuperação de Senha

**Phase**: 1 (Design & Contracts)
**Created**: 2026-09-16
**Feature**: `specs/003-password-recovery/spec.md`

## Purpose

This guide describes how to validate that the password recovery feature works end-to-end. It is a validation checklist for manual and scripted browser testing.

## Prerequisites

- `index.html` implemented with the password recovery modal.
- A Firebase Auth project configured with e-mail/password sign-in.
- A valid e-mail address registered in Firebase Auth (for the success flow).
- A modern browser with JavaScript and ES modules enabled.

## Setup

1. Open `index.html` in a browser (preferably via a local web server to avoid CORS issues with Firebase).
2. Ensure you are on the login screen (`authScreen`).
3. Optionally, open the browser DevTools Network tab and disable cache.

## Test scenarios

### Scenario 1: Link opens the recovery modal

- **Given** the user is on the login screen.
- **When** the user clicks "Esqueci minha senha".
- **Then** a modal with a glassmorphism style appears.
- **And** the e-mail input is focused.
- **And** the modal can be closed by pressing Escape, clicking the backdrop or clicking the close button.

### Scenario 2: Empty e-mail validation

- **Given** the modal is open.
- **When** the user leaves the e-mail field empty and clicks submit.
- **Then** the modal does not call Firebase.
- **And** a visible error message appears.

### Scenario 3: Invalid e-mail format validation

- **Given** the modal is open.
- **When** the user enters `marcio@` and clicks submit.
- **Then** the modal does not call Firebase.
- **And** a visible error message about invalid format appears.

### Scenario 4: Successful recovery request

- **Given** the modal is open and a valid e-mail is entered.
- **And** there is no active cooldown.
- **When** the user clicks submit.
- **Then** the submit button is disabled during the request.
- **And** a loading indicator is visible while the request is in flight.
- **And** after completion, a success message is displayed ("e-mail enviado, confira sua caixa de entrada").
- **And** the submit button remains disabled for 45 seconds.
- **And** the modal can be closed after reading the message.

### Scenario 5: Error feedback

- **Given** the modal is open with a valid e-mail format.
- **When** the submit button is clicked and Firebase returns an error.
- **Then** a friendly error message is displayed.
- **And** the raw Firebase error message is not shown to the user.

### Scenario 6: Cooldown enforcement

- **Given** a request has just completed.
- **When** the user tries to submit again immediately (via DevTools or rapid clicks).
- **Then** the submit button is disabled.
- **And** a cooldown message is shown.
- **And** after 45 seconds, the submit button becomes enabled again.
- **And** reloading the page within the cooldown window still blocks submission.

### Scenario 7: Responsive and themed layout

- **Given** the user is on the login screen.
- **When** the user opens the modal in light theme.
- **Then** the modal uses the ocean/violet/warm palette.
- **And** when switching to dark theme, the modal remains readable and consistent.
- **And** on a narrow viewport, the modal remains usable without horizontal scrolling.

## Expected outcomes

- All scenarios above pass without JavaScript errors in the console.
- The user can complete a password reset request in under 3 interactions.
- Feedback (success or error) appears within 3 seconds under normal network conditions.
- No new Firestore collections, Cloud Functions or backend services are required.

## See also

- `contracts/` for UI and Firebase Auth integration contracts.
- `data-model.md` for states, entities and cooldown rules.
- `research.md` for implementation decisions and rationale.

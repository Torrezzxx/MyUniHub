# Data Model: Recuperação de Senha

**Phase**: 1 (Design & Contracts)
**Created**: 2026-09-16
**Feature**: `specs/003-password-recovery/spec.md`

## Overview

This model describes the data and state involved in the password recovery flow. The state is ephemeral on the client side; no Firestore collections or server-side records are created by this feature.

## Entities

### PasswordResetRequest

A single attempt to request a password reset e-mail. This entity exists only in memory and in `localStorage` for cooldown tracking.

| Field | Type | Description |
|-------|------|-------------|
| `email` | string | The e-mail address entered by the user, trimmed. |
| `status` | enum | Current state of the request. |
| `lastRequestAt` | number \| null | Epoch milliseconds of the last successful send attempt. Used for cooldown. |
| `cooldownUntil` | number \| null | Epoch milliseconds until which further requests are blocked. |
| `errorCode` | string \| null | Firebase Auth error code, if any. |
| `errorMessage` | string \| null | User-facing error message, if any. |

### Entity validation rules

- `email` MUST be non-empty after trimming.
- `email` MUST pass standard e-mail format validation before sending to Firebase.
- `status` MUST be one of: `idle`, `open`, `validating`, `sending`, `success`, `error`, `closed`.
- `lastRequestAt` is updated only after a send attempt that reaches Firebase (even if Firebase later returns an error that is treated as user-facing error; cooldown applies after a successful send or after reaching Firebase).
- `cooldownUntil` is set to `lastRequestAt + 45000` (45 seconds).

## State transitions

```text
closed ──open──> validating ──sending──> success
                         |
                         +--sending──> error
```

| From | To | Trigger |
|------|----|---------|
| closed | open | User clicks "Esqueci minha senha" or opens modal programmatically. |
| open | validating | User submits the form; field-level validation runs. |
| validating | sending | E-mail format is valid; cooldown is inactive; `sendPasswordResetEmail` is called. |
| sending | success | `sendPasswordResetEmail` resolves without throwing. |
| sending | error | `sendPasswordResetEmail` throws (invalid-email, too-many-requests, network, or other). |
| success | closed | User closes the modal after reading the success message. |
| error | closed | User closes the modal after reading the error message. |
| Any | closed | User cancels, presses Escape, clicks backdrop, or clicks close button. |

## Cooldown rules

- After a request reaches Firebase (success or error), the client starts a cooldown period of 45 seconds.
- During cooldown:
  - Submit button MUST be disabled.
  - New send attempts MUST be blocked with a message like "Aguarde alguns instantes antes de tentar novamente."
  - The cooldown timestamp is persisted in `localStorage` so it survives page reloads within the 45-second window.
- After cooldown expires, the form returns to `idle`/`open` state.

## Business constraints

- No e-mail confirmation about account existence is shown; all valid e-mails receive the same success or generic error path.
- Firebase Auth error messages are never shown raw; they are mapped to friendly Portuguese messages.
- No server-side rate limiting or audit log is required for this feature.
- The modal must be usable in both light and dark themes.

## Data volume assumptions

- One modal field (e-mail) per request.
- Ephemeral state per browser session; local persistence limited to cooldown timestamp.
- No large datasets, lists or tables.

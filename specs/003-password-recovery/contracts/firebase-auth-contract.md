# Firebase Auth Contract: Password Reset

**Phase**: 1 (Design & Contracts)
**Created**: 2026-09-16
**Feature**: `specs/003-password-recovery/spec.md`

## Purpose

Defines the integration contract between the MyUniHub client and Firebase Auth for the password reset feature.

## Prerequisites

- Firebase Auth v11.0.1 is already imported and initialized in the client.
- The app module has access to the `auth` instance.
- `sendPasswordResetEmail` MUST be imported from the same `firebase/auth` entry point as the existing Auth functions.

## Import contract

```javascript
import { getAuth, sendPasswordResetEmail } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js";
```

The `auth` instance is available as the module-scoped constant `auth` already used by login, register and logout flows.

## Function contract

### `sendPasswordResetEmail(auth, email)`

- **Input**: `email` (string) trimmed e-mail address.
- **Output**: resolves when Firebase Auth has accepted the request or rejects with a Firebase Auth error.
- **Side effects**: sends an e-mail from the project's configured Action Email templates.

## Behavior expectations

- For a valid e-mail format that passes client-side validation, the client MUST call `sendPasswordResetEmail`.
- If the e-mail format is invalid, the client MUST NOT call Firebase and MUST display a client-side validation error.
- If a 45-second cooldown is active, the client MUST NOT call Firebase and MUST display a cooldown message.
- Any Firebase Auth error MUST be caught and mapped to a friendly user-facing message.

## Error mapping

| Firebase error code | User-facing message |
|---------------------|---------------------|
| `auth/invalid-email` | "E-mail inválido. Verifique o endereço informado." |
| `auth/too-many-requests` | "Muitas tentativas. Aguarde alguns instantes e tente novamente." |
| `auth/network-request-failed` | "Não foi possível conectar. Verifique sua internet." |
| All other errors | "Não foi possível enviar o e-mail. Tente novamente mais tarde." |

## Security constraints

- Client MUST NOT reveal whether the e-mail is registered in Firebase Auth.
- Client MUST NOT forward raw Firebase error messages to the user.
- Client MUST NOT log sensitive request data to audit logs (per clarification).

## Notes

- This contract relies on the existing Firebase project configuration in `index.html`.
- No Firestore collections, Cloud Functions or additional backend services are required.

# Research: Recuperação de Senha

**Phase**: 0 (Research)
**Created**: 2026-09-16
**Feature**: `specs/003-password-recovery/spec.md`

## Overview

This document consolidates research for implementing password recovery in the MyUniHub client-side application. Decisions are documented with rationale and alternatives considered.

---

## Decision 1: Use `sendPasswordResetEmail` from the existing Firebase Auth SDK

**Decision**: Import `sendPasswordResetEmail` from `firebase/auth` inside the existing `<script type="module">` and call it directly from the client.

**Rationale**:
- Firebase Auth is already initialized and used for login/register/logout in the same module.
- No backend, Cloud Functions or custom server is required by the specification.
- This is the standard Firebase mechanism for email-based password reset.
- It integrates with the existing user record and respects project authentication settings.

**Alternatives considered**:
- Custom backend endpoint to call Firebase Admin SDK — rejected because the spec forbids backend and Cloud Functions for this feature.
- `fetchSignInMethodsForEmail` pre-check to confirm the e-mail exists — rejected because it can enable user enumeration and the spec already requests a generic success/error UX without confirming account existence.
- Manual SMTP mail sender — rejected because it requires backend infrastructure and secrets management.

---

## Decision 2: Generic success message for all valid e-mails (security by default)

**Decision**: Always show the same success message ("e-mail enviado, confira sua caixa de entrada") when the request passes client-side validation, regardless of whether the e-mail is registered.

**Rationale**:
- Prevents user enumeration.
- Matches current Firebase email enumeration protection behavior for newer projects.
- Keeps UX simple and secure without extra API calls.
- Aligns with the spec requirement of showing success/error without exposing account status.

**Alternatives considered**:
- Different messages for registered vs. unregistered e-mail — rejected for security reasons (enumeration attack).
- Explicit "e-mail não encontrado" for unknown accounts — rejected because it reveals account existence and was superseded by the clarification that error should be generic when the request fails.

---

## Decision 3: Client-side cooldown of 45 seconds between requests

**Decision**: Store the timestamp of the last successful send attempt in `localStorage`; block any new send attempt for 45 seconds, disable the submit button, and show remaining time information if feasible.

**Rationale**:
- Specified by the user in the clarification session.
- Cheap and sufficient for a client-side app without backend.
- `localStorage` allows the cooldown to survive page reloads during the 45-second window.
- Also disables the submit button as a visual cue.

**Alternatives considered**:
- In-memory only cooldown — rejected because a page reload would reset it, leading to accidental duplicate requests.
- Backend rate limiting — rejected because no backend is allowed.
- 60 seconds — rejected because 45 seconds was explicitly chosen in clarification.

---

## Decision 4: Firebase Auth error mapping and user-facing messages

**Decision**: Map Firebase Auth error codes to friendly Portuguese messages without exposing internal details.

| Error code | User-facing message |
|------------|---------------------|
| `auth/invalid-email` | "E-mail inválido. Verifique o endereço informado." |
| `auth/too-many-requests` | "Muitas tentativas. Aguarde alguns instantes e tente novamente." |
| `auth/network-request-failed` | "Não foi possível conectar. Verifique sua internet." |
| Any other error | "Não foi possível enviar o e-mail. Tente novamente mais tarde." |

**Rationale**:
- `auth/invalid-email` is thrown when the e-mail format is malformed (client-side validation should also catch this before calling Firebase).
- `auth/too-many-requests` is Firebase server-side rate limiting; client cooldown should prevent it, but the mapping still protects the user from a confusing technical message.
- Network errors are common on mobile and should be reported clearly.
- Generic messages avoid leaking internal service details.

**Alternatives considered**:
- Showing raw error `message` to the user — rejected because messages are English, technical and may expose implementation details.
- Always showing "Erro desconhecido" — rejected because it hurts usability for recoverable failures like network issues.

---

## Decision 5: Modal UI follows the existing modal pattern

**Decision**: Use the same modal conventions already established in the app:
- Backdrop: `fixed inset-0 bg-black/55 backdrop-blur-md`.
- Panel: white/dark slate, `rounded-3xl`, `shadow-[0_24px_64px_rgba(0,0,0,.22)]`, `border border-white/60`.
- Top accent: `h-1 rounded-t-3xl` gradient `linear-gradient(90deg,#2E5A9E,#7c5cbf)`.
- Header: title and subtitle, close button.
- Footer: Cancel and primary action buttons.
- Glassmorphism and ocean/violet/warm palette.

**Rationale**:
- Maintains design consistency across the app.
- Reuses existing CSS classes and conventions.
- Reduces implementation risk by copying a known-working pattern.

**Alternatives considered**:
- Inline form below password field — rejected because spec asks for a modal and because it would require more layout changes on the login card.
- Custom visual style — rejected to preserve brand consistency.

---

## Decision 6: Accessibility basics for the recovery modal

**Decision**:
- Move focus into the e-mail input when opening the modal.
- Return focus to the "Esqueci minha senha" link when closing.
- Use explicit labels and ARIA attributes on form fields and buttons.
- Close the modal on `Escape`.

**Rationale**:
- Spec requires focus management and ARIA/labels.
- Basic keyboard and screen-reader support without building a full focus trap.

**Alternatives considered**:
- Full focus trap — rejected because overkill for a single-field modal; basic focus management is sufficient per spec.
- No accessibility handling — rejected because spec explicitly requires it.

---

## Decision 7: No audit logging of password reset requests

**Decision**: Do not log password reset events to Firestore, console or external monitoring, per clarification.

**Rationale**:
- User explicitly chose no dedicated audit logging for this version.
- Keeps the feature scope minimal and avoids storing audit data in the client or backend.

**Alternatives considered**:
- Console logging for debugging — rejected because the clarification says no dedicated audit logging; console logs may still be used for development but are out of production scope.
- Logging to Firestore — rejected due to scope, storage and audit-data sensitivity.

---

## Sources

- [Firebase Auth - Password sign-in docs](https://firebase.google.com/docs/auth/web/password-auth)
- Web search: "Firebase Auth sendPasswordResetEmail error codes" (research on user enumeration protection and error codes)

# UI Contract: Password Reset Modal

**Phase**: 1 (Design & Contracts)
**Created**: 2026-09-16
**Feature**: `specs/003-password-recovery/spec.md`

## Purpose

Defines the user-visible behavior and appearance expected from the password recovery modal. Implementations must satisfy all requirements below.

## Modal appearance

- Modal MUST use the existing app modal pattern:
  - Backdrop: `fixed inset-0 bg-black/55 backdrop-blur-md`.
  - Panel: `bg-white dark:bg-slate-800 rounded-3xl shadow-[0_24px_64px_rgba(0,0,0,.22)] border border-white/60 dark:border-slate-700`.
  - Top accent: gradient `linear-gradient(90deg,#2E5A9E,#7c5cbf)` as a thin rounded bar at the top.
- Modal MUST follow glassmorphism and use the ocean/violet/warm palette.
- Modal MUST be usable in both light and dark themes.
- Modal MUST be responsive and usable on narrow viewports without horizontal scrolling.

## Modal content

- Header with a title such as "Recuperar Senha" and a subtitle explaining the purpose.
- A single e-mail input with:
  - Visible label.
  - Placeholder text like "seu@email.com".
  - Accessible name / ARIA label.
  - Focus glow style consistent with `input-glow`.
- Cancel/Close button with a clear label (✕ or "Cancelar").
- Submit button with label like "Enviar E-mail de Recuperação".
- Feedback area for success and error messages.
- Loading indicator during Firebase request.

## Modal behavior

- Opens when user clicks "Esqueci minha senha" on the login screen.
- Closes on:
  - Clicking the close button.
  - Clicking the backdrop (outside modal panel).
  - Pressing Escape.
  - Calling `closePasswordResetModal()`.
- Focus MUST move to the e-mail input when the modal opens.
- Focus MUST return to the "Esqueci minha senha" link when the modal closes.
- All fields and buttons MUST have accessible labels.
- Submit button MUST be disabled while a request is in flight.
- Submit button MUST be disabled for 45 seconds after a completed request (cooldown).

## Messages

- Success message: "e-mail enviado, confira sua caixa de entrada".
- Empty e-mail error: a clear message that the e-mail is required.
- Invalid e-mail error: a clear message that the e-mail format is invalid.
- Cooldown message: a clear message that the user must wait 45 seconds.
- Firebase error: friendly Portuguese message mapped from error code, not raw error text.

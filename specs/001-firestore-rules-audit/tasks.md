---
description: "Task list — Firestore Security Rules (MyUniHub)"
---

# Tasks: Firestore Security Rules — MyUniHub

**Input**: Design documents from `/specs/001-firestore-rules-audit/`

**Prerequisites**:
- `plan.md` (required) — tech stack, structure
- `spec.md` (required) — user stories P1, P2, P3
- `research.md` — 9 design decisions (D1–D9)
- `data-model.md` — entity `users/{uid}` and field validations
- `contracts/firestore-rules-contract.md` — external contract of the rules
- `quickstart.md` — validation scenarios (emulator + Playground + deploy)

**Branch**: `001-firestore-rules-audit`

**Tests**: O spec não solicita TDD explícito; o caminho de validação recomendado é via (a) emulador Firestore local, (b) Rules Playground no console, e (c) validação manual no app de produção após deploy. Tasks de teste com `@firebase/rules-unit-testing` são marcadas como **OPTIONAL** e vivem na fase de Polish.

**Organization**: Tasks are grouped by user story (US1, US2, US3, US4) to enable independent implementation and validation. Como US1, US2 e US3 vivem no **mesmo arquivo** (`firestore.rules`), tarefas em fases diferentes compartilham o mesmo path — o paralelismo intra-arquivo é baixo, e a aplicação do marcador `[P]` é limitada.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

Esta feature tem uma estrutura plana (ver `plan.md`):

```text
MyUniHub/
├── firestore.rules             # NOVO
├── firebase.json               # NOVO
├── .firebaserc                 # NOVO
├── index.html                  # modificação mínima (bootstrapUid)
└── specs/001-firestore-rules-audit/  # já existe
```

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Criar a infraestrutura mínima (arquivo de Rules + config Firebase) necessária para deployar.

- [ ] T001 [P] Create `firestore.rules` at `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` with header `rules_version = '2';` and section markers for `isSignedIn`/`isOwner` helpers, `match /users/{uid}`, and catch-all `match /{document=**}` (placeholders to be filled in later phases).
- [ ] T002 [P] Create `firebase.json` at `C:\Users\Marcio\Desktop\MyUniHub\firebase.json` with `{ "firestore": { "rules": "firestore.rules" } }` only — no hosting, no other services.
- [ ] T003 [P] Create `.firebaserc` at `C:\Users\Marcio\Desktop\MyUniHub\.firebaserc` with `{ "projects": { "default": "myunihub-4bb1f" } }`.
- [ ] T004 Run `firebase login` (verify existing session) and `firebase projects:list` from `C:\Users\Marcio\Desktop\MyUniHub\`; confirm `myunihub-4bb1f` is listed and accessible.

**Checkpoint**: Arquivos básicos existem; `firebase deploy --only firestore:rules --dry-run` deve passar (mesmo com placeholder de Rules).

---

## Phase 2: Foundational — Helpers & Catch-All (Blocking Prerequisites)

**Purpose**: Funções helper e catch-all fail-closed que todas as User Stories reutilizam. Sem isso, nenhuma das stories funciona.

- [ ] T005 [US1] Implement `function isSignedIn() { return request.auth != null; }` and `function isOwner(uid) { return isSignedIn() && request.auth.uid == uid; }` in `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules`. (Logical mapping: US1 uses isOwner; US2 uses isSignedIn.)
- [ ] T006 [US1] Add `match /{document=**} { allow read, write: if false; }` at the end of `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` to enforce fail-closed (FR-009).
- [ ] T007 [US1] Add `match /users/{uid} { allow read, write: if isOwner(uid); }` skeleton in `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` (FR-002/FR-003). Refinements de validação granular (FR-006, FR-007b) virão nas fases US4.

**Checkpoint**: Com estas três tasks, US1 e US2 já estão **funcionalmente** completas (sem validação de shape). Validar via `firebase emulators:start --only firestore` + cenários 1, 2, 3 do `quickstart.md` §2.1.

---

## Phase 3: User Story 1 — Isolamento por usuário (Priority: P1) 🎯 MVP

**Goal**: Cada usuário autenticado só lê/escreve `users/{seu_uid}`. Já coberto estruturalmente por T005 + T007; esta fase valida e documenta.

**Independent Test**: Ver `quickstart.md` §1.3 cenários A/B e §2.1 cenários 1, 2, 4 — `getDoc`/`setDoc` no próprio uid permite, em outro uid nega.

### Implementation for User Story 1

- [ ] T008 [US1] Add explanatory comment above `match /users/{uid}` in `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` describing: path protected, who can access, why (multi-tenant isolation). Satisfies FR-010 and SC-006.
- [ ] T009 [US1] Manually validate US1 via Rules Playground (https://console.firebase.google.com/project/myunihub-4bb1f/firestore/rules) executing scenarios 1, 2, 4, 9 of `quickstart.md` §2.1. Record results in a comment block at the bottom of the file or in a separate `validation-log.md` (optional).

**Checkpoint**: US1 fully validated. Bloqueios cruzados demonstrados.

---

## Phase 4: User Story 2 — Bloqueio de acesso não autenticado (Priority: P1)

**Goal**: Toda requisição com `request.auth == null` é negada. Já coberto por T005/T006/T007 (qualquer match que use `isOwner(uid)` falha quando não há auth; catch-all nega paths fora). Esta fase valida.

**Independent Test**: Ver `quickstart.md` §1.3 cenário B e §2.1 cenários 3, 9.

### Implementation for User Story 2

- [ ] T010 [US2] Add a comment above the catch-all `match /{document=**}` in `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` stating that this also blocks unauthenticated access (FR-001).
- [ ] T011 [US2] Manually validate US2 via Rules Playground executing scenario 3 of `quickstart.md` §2.1. Confirm `request.auth` simulation toggle works.

**Checkpoint**: US2 fully validated.

---

## Phase 5: User Story 3 — Compatibilidade com a conta demo (Priority: P2)

**Goal**: A conta `demo@myunihub.app` continua funcionando normalmente, sem regras condicionais especiais. Esta fase é puramente **validação** — não há código novo nas Rules.

**Independent Test**: Ver `quickstart.md` §3.4 passo 3.

### Validation for User Story 3

- [ ] T012 [US3] Run the demo flow end-to-end against the emulator: click "Explorar em Modo Demo" → add a `deadline` → reload page → deadline persists. If running against production rules, run after the deploy of Phase 8. Document result.

**Checkpoint**: US3 validated — the demo account uses generic rules and behaves identically to a regular user.

---

## Phase 6: User Story 4 — Integridade do `uid` interno (Priority: P3)

**Goal**: Em `books` e `weekPlans`, cada item deve ter `uid == request.auth.uid` (ou ausente, tratado como legado). Atende FR-006 com a abordagem híbrida da Clarification Q1, escopo Q4.

**Independent Test**: Ver `quickstart.md` §2.1 cenários 5 e 6.

### Implementation for User Story 4

- [ ] T013 [US4] Add helper functions to `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules`:
      ```
      function hasValidUidOrLegacy(item) {
        return !('uid' in item) || item.uid == request.auth.uid;
      }
      function isValidBooksField(books) {
        return books == null || (books.size() <= 200 && books.all(hasValidUidOrLegacy, []));
      }
      function isValidWeekPlansField(plans) {
        return plans == null || (plans.size() <= 200 && plans.all(hasValidUidOrLegacy, []));
      }
      ```
      (Also implements the array-size portion of FR-007b for these two fields.)
- [ ] T014 [US4] Refine the `match /users/{uid}` body in `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` to:
      ```
      allow read: if isOwner(uid);
      allow write: if isOwner(uid)
        && isValidBooksField(request.resource.data.books)
        && isValidWeekPlansField(request.resource.data.weekPlans);
      ```
      Read is intentionally not gated by shape validations (read of an oversized doc is not a write-side abuse).
- [ ] T015 [US4] Manually validate US4 via Rules Playground executing scenarios 5, 6, 7, 8 of `quickstart.md` §2.1.

**Checkpoint**: US4 fully validated. Items com uid divergente são rejeitados; sem uid (legado) e uid igual são aceitos.

---

## Phase 7: Client-side — Migração silenciosa de `uid` em items legados (Q1 follow-up)

**Purpose**: Atender a parte client-side da Clarification Q1 (migração silenciosa one-shot para retroagir `uid` em `books` e `weekPlans` legados). Sem isso, os usuários com livros antigos continuariam no estado "sem uid" indefinidamente — funciona com as Rules (porque FR-006 aceita ausência), mas o "ruído" legado persiste.

**Independent Test**: Verificar que um usuário com livros legados vê `uid` preenchido após o primeiro reload pós-deploy.

### Implementation

- [ ] T016 [P] In `C:\Users\Marcio\Desktop\MyUniHub\index.html` add a `bootstrapUid()` function that:
      1. Checks a `localStorage` flag `hub_uid_bootstrapped_v1`; if set, returns early.
      2. Iterates `D.books` and `D.weekPlans`; for each item without a `uid` field, sets `item.uid = uid` (the module-scoped Firebase uid).
      3. If any item was modified, calls `persistAll()`.
      4. Sets the flag to `true`.
      Function must be idempotent and safe to call from `onAuthStateChanged` after `startListener()`.
- [ ] T017 [P] Wire `bootstrapUid()` into the `onAuthStateChanged` callback in `C:\Users\Marcio\Desktop\MyUniHub\index.html` (around line 1965): after `startListener()` is invoked, call `bootstrapUid()`.

**Checkpoint**: Migração silenciosa implementada. Pode ser validada com um usuário que tenha items legados (apaga manualmente o campo `uid` no console e observa o preenchimento no próximo reload).

---

## Phase 8: Foundational-client — Validação de shape FR-007b (profile)

**Purpose**: Implementar a parte de FR-007b que **não** foi coberta na US4 (validação de strings de `profile`). Esta fase não tem user story específica (é defesa em profundidade do spec).

**Independent Test**: Ver `quickstart.md` §2.1 cenário 7.

### Implementation

- [ ] T018 Add `function isValidProfileField(p)` to `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` enforcing:
      - `p == null || (p.name.size() <= 80 && p.course.size() <= 120 && p.institution.size() <= 120 && p.goalMinutes >= 0 && p.goalMinutes <= 1440)`
- [ ] T019 Add `isValidProfileField(request.resource.data.profile)` to the `allow write` condition in `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` (the same block refined in T014).
- [ ] T020 [P] Add a comment to `C:\Users\Marcio\Desktop\MyUniHub\firestore.rules` justifying why FR-007 was removed (1 MiB native limit; no `request.resource.size()` in Firestore Rules) and FR-007b is the defense-in-depth replacement.

**Checkpoint**: FR-007b completa. Limites de profile são respeitados.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Validação final, deploy, e garantia de que o spec foi atendido de ponta a ponta.

- [ ] T021 [P] **OPTIONAL** — Add `@firebase/rules-unit-testing` to a `tests/firestore/rules.spec.js` file in `C:\Users\Marcio\Desktop\MyUniHub\tests\firestore\` covering the 6 scenarios listed in `quickstart.md` §1.4. This is **out of the strict scope** of this feature but recommended.
- [ ] T022 [P] Verify all FRs (FR-001 through FR-010, excluding the removed FR-007) are covered by at least one task above. Update `specs/001-firestore-rules-audit/checklists/requirements.md` if necessary.
- [ ] T023 Run `firebase deploy --only firestore:rules --project myunihub-4bb1f` from `C:\Users\Marcio\Desktop\MyUniHub\`. Capture the console URL of the deployed rules in the commit message.
- [ ] T024 Validate the deployed rules against the production app:
      1. Open the production URL → register a fresh user → finish setup → add a subject, a deadline, and a book → reload → all persist (SC-004).
      2. Open a second incognito window → register a different user → confirm cross-uid read is denied via DevTools (SC-001).
      3. In the second window, sign out → in DevTools confirm any unauthenticated read is denied (SC-002).
      4. Open the production URL in a third incognito window → click "Explorar em Modo Demo" → add a deadline → reload → it persists (SC-003).
- [ ] T025 [P] Update `C:\Users\Marcio\Desktop\MyUniHub\README.md` (currently only `# MyUniHub`) with a one-paragraph note: "This project uses Firestore Security Rules defined in `firestore.rules`; deploy with `firebase deploy --only firestore:rules`." (Currently the README is a single line.)
- [ ] T026 Commit the changes on branch `001-firestore-rules-audit` with a message like `feat(firestore-rules): add security rules with owner-isolation, anonymous-deny, and legacy-uid migration`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies. Can start immediately.
- **Foundational (Phase 2)**: Depends on T001 (the `firestore.rules` file must exist). Blocks US1+.
- **User Stories (Phases 3–6)**: All depend on Phase 2 completion. Within the same `firestore.rules` file, so strictly sequential per file (but logically independent — the rules are layered).
- **Client-side migration (Phase 7)**: Depends on Phase 2 (Rules must exist and be validated before the client can rely on them). Can run in parallel with Phases 3–6 since the `index.html` is a different file.
- **Shape validation (Phase 8)**: Depends on Phase 6 (US4 — same match block).
- **Polish (Phase 9)**: Depends on all prior phases.

### User Story Dependencies

- **US1 (P1)**: Depends on Phase 2 only.
- **US2 (P1)**: Depends on Phase 2 only (validated by US1's structure).
- **US3 (P2)**: Depends on US1+US2 (it's a validation, not code).
- **US4 (P3)**: Depends on Phase 2; refines the `allow write` block of US1.

Within each user story, there are no sub-tasks needing ordering — each story is a small number of edits to the same file.

### Within Each User Story

- Helpers (T005/T013) before match bodies (T007/T014).
- Match body before validation (T009/T015/T019).
- Client-side `bootstrapUid()` is independent of any specific user story; it just needs the Rules deployed first (so FR-006's lenient behavior protects the first run).

### Parallel Opportunities

- T001, T002, T003 can run in parallel (different new files).
- T016, T017 can run in parallel with T013–T015 (different files: `index.html` vs `firestore.rules`).
- T018, T020 can run in parallel with T016/T017.
- T021, T022, T025 can run in parallel within Phase 9.

---

## Parallel Examples

### Phase 1 — all setup files in parallel

```bash
# Three different new files; can be authored independently:
Task: "Create firestore.rules with rules_version = '2' header"
Task: "Create firebase.json with firestore.rules path"
Task: "Create .firebaserc with default project alias"
```

### Phase 7 + Phase 8 — `index.html` work in parallel with `firestore.rules` refinements

```bash
Task: "Add bootstrapUid() to index.html"             # T016
Task: "Wire bootstrapUid() into onAuthStateChanged"   # T017
Task: "Add isValidProfileField() helper to firestore.rules"   # T018
Task: "Add FR-007 removal comment to firestore.rules"        # T020
```

---

## Implementation Strategy

### MVP First (US1 + US2 only)

A minimum viable security posture is achieved with **just T001 + T005 + T006 + T007** (about 20 lines of Rules). At that point:

- Cross-uid reads/writes are denied.
- Anonymous access is denied.
- The demo flow works (because it's a regular user).

That's enough to ship safely to production. Everything else (US3 validation, US4, FR-007b shape checks, client-side migration, comments) is incremental hardening.

### Recommended Sequencing

1. **T001–T007** (Setup + Foundational). At this point: `firebase deploy --only firestore:rules` works and provides the MVP.
2. **T008–T011** (US1 + US2). Comments and validation via Playground.
3. **T012** (US3 demo validation, manual).
4. **T013–T015** (US4 uid integrity).
5. **T016–T017** (client-side migration).
6. **T018–T020** (FR-007b shape).
7. **T021–T026** (Polish: tests optional, deploy, README, commit).

### Incremental Delivery

After step 1, you can deploy and ship. Steps 2–7 can be PRs or commits within the same branch, each deployable independently.

---

## Notes

- The `firestore.rules` file is the single point of contention — most tasks touch it. Tasks marked `[P]` are limited to **different files** (e.g., `index.html`, `firebase.json`, `README.md`).
- US3 is intentionally a **validation** task, not an implementation task — the demo account is generic by design (Clarification Q5).
- Tasks that touch `index.html` (T016, T017) require re-testing the demo flow after each change.
- The catch-all `match /{document=**}` (T006) is what enforces FR-009 (fail-closed). It MUST be the last match in the file.
- For the smoke tests in the Rules Playground, the emulator project alias is the same as production (`myunihub-4bb1f`), but the Playground does not write to production unless you explicitly hit **Publish** — read the docs before doing so.
- `goalMinutes` in profile is bounded to `[0, 1440]` (24 hours max daily study). Confirm this upper bound matches product intent before deploy; if 1440 is too restrictive, update T018.

---

## Total Task Count: 26

| Phase | Task Count | Notes |
|---|---|---|
| Phase 1: Setup | 4 | T001–T004 |
| Phase 2: Foundational | 3 | T005–T007 |
| Phase 3: US1 (P1) | 2 | T008–T009 |
| Phase 4: US2 (P1) | 2 | T010–T011 |
| Phase 5: US3 (P2) | 1 | T012 |
| Phase 6: US4 (P3) | 3 | T013–T015 |
| Phase 7: Client migration | 2 | T016–T017 |
| Phase 8: Shape validation | 3 | T018–T020 |
| Phase 9: Polish | 6 | T021–T026 |
| **Total** | **26** | |

### Parallel Opportunities (concrete)

- T001, T002, T003 (3 new files) — parallel.
- T016, T017 (both `index.html`, but T016 defines function, T017 calls it — **sequential within file**).
- T018, T020 (both `firestore.rules`, but at different locations — can be authored in any order, but the file write is sequential).
- T021, T022, T025 (3 different files) — parallel.

### Independent Test per Story (summary)

- **US1**: `quickstart.md` §2.1 cenários 1, 2, 4.
- **US2**: `quickstart.md` §2.1 cenários 3, 9.
- **US3**: `quickstart.md` §3.4 passo 3.
- **US4**: `quickstart.md` §2.1 cenários 5, 6, 7, 8.

### MVP Scope (recommended)

**T001, T005, T006, T007** — that's 4 tasks and about 20 lines of `firestore.rules`. Deploy with `firebase deploy --only firestore:rules` and you have a production-safe baseline. Everything else is incremental hardening.

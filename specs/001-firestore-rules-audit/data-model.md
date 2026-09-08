# Data Model: Firestore Document `users/{uid}`

**Date**: 2026-08-31 | **Branch**: `001-firestore-rules-audit`

Este documento descreve a estrutura canônica do documento Firestore do MyUniHub e o mapeamento entre cada campo e as validações que as `firestore.rules` aplicarão. É a **fonte de verdade** para os limites da FR-007b e para os tipos referenciados em `request.resource.data` no contrato de Rules.

## Visão Geral

- **Coleção única**: `users`
- **Path do documento**: `users/{uid}` onde `{uid}` é o `request.auth.uid` do Firebase Auth
- **Não há sub-coleções** (escopo desta feature).
- **Não há outros documentos** acessados pelo cliente atual (verificado no spec, linha 165).

## Entidade: Documento `users/{uid}`

| Campo | Tipo | Obrigatório? | Validação nas Rules (FR-007b) | Origem no client |
|---|---|---|---|---|
| `profile` | `map` | sim | Veja `Profile` abaixo | `index.html:1943` (criação), `1989` (setup) |
| `subjects` | `list<map>` | sim | `size() <= 30` | `index.html:2221` (push) |
| `deadlines` | `list<map>` | sim | `size() <= 200` | `index.html:2303` |
| `weekPlans` | `list<map>` | sim | `size() <= 200` + validação `hasValidUidOrLegacy` por item (FR-006) | `index.html:2688`, `2754` |
| `books` | `list<map>` | sim | `size() <= 200` + validação `hasValidUidOrLegacy` por item (FR-006) | `index.html:3704` |
| `checklist` | `map` | sim | `items.size() <= 50` | `index.html:1874` |
| `studyHistory` | `map<string, number>` | sim | sem limite por chave; total do doc respeita 1 MiB nativo | `index.html:1851` |
| `timer` | `map` | sim | sem limite específico (objeto pequeno) | `index.html:1854` |
| `onboardingComplete` | `bool` | opcional | sem validação | `index.html:1948` |

### Sub-entidade: `profile` (map)

| Campo | Tipo | Validação FR-007b |
|---|---|---|
| `name` | string | `size() <= 80` |
| `course` | string | `size() <= 120` |
| `institution` | string | `size() <= 120` |
| `semester` | string (1–2 dígitos) | `size() <= 4` (tolerância: "10", "11", "12") |
| `goalMinutes` | number | `>= 0 && <= 1440` (24 h) |
| `semStart` | string (YYYY-MM-DD) | `size() == 10` se presente |
| `semEnd` | string (YYYY-MM-DD) | `size() == 10` se presente |

### Sub-entidade: item de `books` (map)

| Campo | Tipo | Validação |
|---|---|---|
| `id` | number | sem validação específica |
| `title` | string | `size() <= 200` |
| `author` | string | `size() <= 200` |
| `totalPages` | number | `>= 0` |
| `currentPage` | number | `>= 0 && <= totalPages` |
| `deadline` | string (YYYY-MM-DD) | sem validação específica |
| `deadlineMs` | number | sem validação específica |
| `pagesPerDay` | number | `>= 0` |
| `done` | bool | sem validação |
| `createdAt` | string (ISO) | sem validação |
| `uid` | string (opcional) | **validação FR-006**: `!('uid' in item) || item.uid == request.auth.uid` |

### Sub-entidade: item de `weekPlans` (map)

| Campo | Tipo | Validação |
|---|---|---|
| `id` | number | sem validação |
| `subj` | string | `size() <= 120` |
| `topic` | string | `size() <= 200` |
| `date` | string (YYYY-MM-DD) | `size() == 10` se presente |
| `dateMs` | number | sem validação |
| `prio` | string (Baixa/Média/Alta) | sem validação específica (texto livre) |
| `estTime` | string | sem validação específica |
| `status` | string | sem validação específica |
| `done` | bool | sem validação |
| `notes` | string | `size() <= 2000` |
| `resources` | list | sem validação específica |
| `createdAt` | string (ISO) | sem validação |
| `uid` | string (opcional) | **validação FR-006**: idem books |

### Sub-entidade: item de `subjects` (map)

Sem validação de `uid` interno (Q4: escopo da FR-006 cobre só books/weekPlans). Validações opcionais podem ser adicionadas em feature futura.

### Sub-entidade: item de `deadlines` (map)

Sem validação de `uid` interno (Q4). Validações opcionais podem ser adicionadas em feature futura.

## Relacionamentos

- `users/{uid}` é **standalone** — não referencia outros documentos.
- Nenhuma relação pai-filho (sem sub-coleções).
- `uid` interno em `books[i]` e `weekPlans[i]` é uma **redundância defensiva** do `request.auth.uid` (FR-006).

## Transições de Estado

- **Criação**: `createUserWithEmailAndPassword` → `setDoc(users/{uid}, { profile, subjects:[], ..., onboardingComplete: false })` (`index.html:1942-1949`).
- **Update**: `setDoc(..., {merge: true})` ou `updateDoc` via `persist()` (`index.html:1843-1846`).
- **Delete**: `deleteDoc(users/{uid})` ao "apagar dados" (`index.html:2539`).
- **Migração silenciosa** (D8): após o deploy, o client percorre `books` e `weekPlans` uma vez e preenche `uid` em items sem o campo.

## Limites globais (nativos, não-via-Rules)

- **Tamanho total do doc**: 1.048.576 bytes (1 MiB) — aplicado pelo Firestore, **não** pelas Rules.
- **Profundidade de path**: 100 segmentos — não é problema aqui (path tem 2 segmentos: `users/{uid}`).
- **Operações por segundo por documento**: 1 write/s sustentado (pode ser aumentado com burst). As Rules não mitigam isso.

## Validações que **NÃO** entram nesta feature

- Tipo estrito por campo (e.g., `profile.goalMinutes` é **number**): Firestore Rules validam tipo implicitamente na escrita via SDK, mas as Rules não vão além. Se um cliente malicioso mandar `goalMinutes: "abc"`, o Firestore vai rejeitar antes das Rules serem consultadas.
- Validação de formato (regex em `email`, `date`): o client já valida; as Rules podem ser estendidas no futuro.
- Verificação cruzada entre campos (e.g., `semEnd > semStart`): fora de escopo. Se necessário, adicionar em feature de "validação de domínio".

## Rastreabilidade Spec → Data Model

| FR | Mapeamento |
|---|---|
| FR-001 | `request.auth != null` em todos os `allow` |
| FR-002 | `match /users/{uid}` + `isOwner(uid)` |
| FR-003 | `isOwner(uid)` falha para outro uid → deny |
| FR-004 | Sem `allow list` em nenhum match → list denied |
| FR-005 | Sem tratamento especial para demo; rules genéricas |
| FR-006 | `hasValidUidOrLegacy(books[i])` e `(weekPlans[i])` |
| FR-007 | removida; substituída por FR-007b |
| FR-007b | `*.size() <= N` em cada campo conforme tabela acima |
| FR-008 | `firebase deploy --only firestore:rules` (ver quickstart) |
| FR-009 | `match /{document=**}` final com `allow read, write: if false` |
| FR-010 | comentários em cada bloco |

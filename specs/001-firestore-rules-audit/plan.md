# Implementation Plan: Firestore Security Rules — MyUniHub

**Branch**: `001-firestore-rules-audit` | **Date**: 2026-08-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-firestore-rules-audit/spec.md`

**User-provided implementation guidance**:
- Padrão Firestore Security Rules v2 (`rules_version = '2';`).
- Arquivo `firestore.rules` na **raiz do projeto**.
- Sem Storage nem subcoleções — apenas o documento `users/{uid}`.
- Deploy via `firebase deploy --only firestore:rules`.

## Summary

Auditar e implementar Firestore Security Rules para o MyUniHub, cobrindo o modelo de dados existente (documento único `users/{uid}` com 9 campos), garantindo (1) isolamento por usuário, (2) bloqueio de acesso anônimo, (3) compatibilidade com a conta demo sem exceções especiais. A entrega é um único arquivo `firestore.rules` na raiz + ajustes mínimos no `index.html` para migração silenciosa de `uid` em livros e planos semanais, e configuração de deploy via `firebase.json` mínimo.

## Technical Context

- **Language/Version**: Firestore Security Rules Language v2 (`rules_version = '2';`). Sem código de aplicação a escrever em linguagem tradicional para esta feature — a "linguagem" é o DSL das Rules.
- **Primary Dependencies**:
  - Firebase CLI v15.22.3 (já instalado, verificado).
  - Node.js v22.15.0 (já instalado, verificado).
  - Projeto Firebase `myunihub-4bb1f` (credenciais já presentes em `index.html:1794-1800`).
  - `firebase-tools` (vem com o `firebase` CLI).
- **Storage**: Google Cloud Firestore (modo nativo, não Datastore). Plano Spark ou Blaze — compatível com ambos. Documento único `users/{uid}` (1 MiB de limite nativo).
- **Testing**:
  - **Emulador local**: `firebase emulators:start --only firestore` + `firestore.rules` carregado automaticamente quando `firebase.json` aponta para ele.
  - **Rules Playground** no console Firebase (testes pontuais).
  - **Test runner oficial**: `@firebase/rules-unit-testing` (opcional, recomendado para CI).
  - **Validação E2E no app**: fluxo demo + fluxo de usuário comum (manual, conforme SC-003 e SC-004).
- **Target Platform**: Web (SPA estática servida provavelmente em Firebase Hosting). As Rules rodam no servidor do Firestore, independentemente do cliente.
- **Project Type**: Web app estática client-side com backend gerenciado (Firebase). Esta feature é puramente **back-end-as-code** — não há UI nem API nova.
- **Performance Goals**: As Rules devem avaliar em **< 50 ms p95** para o caminho de leitura/escrita típico (uma única `get`/`setDoc` em `users/{uid}`). O Firestore cobra ~1 unidade de leitura/escrita por eval de Rule + 1 pela operação; Rules simples (sem chamadas recursivas, sem `exists()` em chains profundos) ficam bem abaixo desse limite.
- **Constraints**:
  - Rules não podem chamar serviços externos; só podem ler `request.auth`, `request.resource.data`, `resource.data`, e funções declaradas no arquivo.
  - Sem `request.resource.size()` em Firestore (apenas em Storage). Para tamanho total do doc, dependemos do limite nativo de 1.048.576 bytes/doc.
  - Sem `request.time`-based staleness (decisão Q2: concorrência fica fora de escopo).
- **Scale/Scope**:
  - ~1 spec; ~1 arquivo de Rules; ~0-1 arquivo de configuração Firebase.
  - 1 collection (`users`); 1 documento por usuário; 9 campos tipados.
  - Sem impacto em banda/custo além do que já existe (Rules são avaliadas em toda request; o custo por request é desprezível para este padrão simples).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

A constitution do projeto (`.specify/memory/constitution.md`) está atualmente com o template padrão, sem princípios preenchidos. Portanto:

- **Sem gates de constitution aplicáveis**: nenhum princípio da constitution é violado ou exige justificativa.
- **Veredito**: ✅ **PASS** (pass-through — constitution vazia não impõe constraints).

Esta seção será re-avaliada após Phase 1. Como a feature é back-end-as-code sem alteração de UI, sem escolha de framework, sem linguagem de aplicação, é improvável que gates adicionais venham a ser aplicáveis.

## Project Structure

### Documentation (this feature)

```text
specs/001-firestore-rules-audit/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (gerado abaixo)
├── data-model.md        # Phase 1 output (gerado abaixo)
├── quickstart.md        # Phase 1 output (gerado abaixo)
├── contracts/           # Phase 1 output (gerado abaixo)
│   └── firestore-rules-contract.md
├── spec.md              # Especificação (existente)
├── checklists/
│   └── requirements.md  # Checklist de qualidade (existente, validado)
└── tasks.md             # Phase 2 output (NÃO criado por este comando)
```

### Source Code (repository root)

```text
MyUniHub/
├── index.html                  # App (existente, com 1 modificação mínima)
├── firestore.rules             # NOVO — Firestore Security Rules
├── firebase.json               # NOVO — Configuração mínima para deploy
├── .firebaserc                 # NOVO — Alias do projeto (pode ser opcional)
├── README.md                   # Existente
├── .gitignore                  # Existente
└── .specify/, .claude/         # Existente
```

Estrutura plana, sem `src/` ou `tests/` separados: a feature é um único arquivo de Rules + um arquivo de config Firebase + uma pequena modificação no `index.html`. Não justifica estrutura adicional.

**Estrutura adicional opcional** (recomendada para CI/testes futuros, fora do escopo desta entrega):

```text
tests/
└── firestore/
    ├── rules.spec.js           # Test runner com @firebase/rules-unit-testing
    └── fixtures/               # Documentos de exemplo
```

**Structure Decision**: estrutura plana escolhida. Sem `tests/` neste PR; testes podem ser adicionados em PR separado se desejado.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sem violações de constitution a justificar. Tabela vazia (omitida).

---

## Phase 0: Research

Artefato gerado: [`research.md`](./research.md).

Decisões-chave consolidadas (ver research.md para detalhes completos):

| # | Decisão | Rationale |
|---|---|---|
| D1 | Usar `rules_version = '2';` (sintaxe moderna com `match`, `allow`, `function`) | Pedido explícito do usuário; v1 é legacy. |
| D2 | Função `isOwner()` + `isSignedIn()` para reduzir duplicação | Padrão recomendado pela documentação oficial. |
| D3 | `match /users/{uid}` com `allow read, write: if isOwner(uid)` na raiz, e regras granulares em submatch | Estrutura canônica; permite evoluir para subcoleções no futuro. |
| D4 | Para FR-006, helper `hasValidUidOrLegacy(item)` que aceita ausência de `uid` ou `uid == request.auth.uid` | Resolve Q1 e Q4 (híbrido + escopo books/weekPlans). |
| D5 | Para FR-007b, validações granulares via `request.resource.data.profile.name.size() <= 80` etc. | Resolve Q3; usa `string.size()` e `list.size()` (operadores válidos em Firestore Rules). |
| D6 | Comentários `// ── path: ...   // por quê: ...` em cada bloco | Atende FR-010 e SC-006. |
| D7 | `firebase.json` mínimo com `firestore: { rules: "firestore.rules" }` | Necessário para `firebase deploy --only firestore:rules`. |
| D8 | Migração silenciosa one-shot no `index.html`: `bootstrapUid()` que preenche `uid` em items de `books` e `weekPlans` se ausente, executado no primeiro `persistAll` após o deploy | Resolve Q1 sem quebrar dados legados. |

## Phase 1: Design & Contracts

Artefatos gerados:
- [`data-model.md`](./data-model.md) — modelo de dados do documento `users/{uid}` com mapeamento para validações nas Rules.
- [`contracts/firestore-rules-contract.md`](./contracts/firestore-rules-contract.md) — contrato das Rules: paths cobertos, operações permitidas/negadas, exemplos de request.
- [`quickstart.md`](./quickstart.md) — guia de validação: como testar localmente com emulador e como deployar.

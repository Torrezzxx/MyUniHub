# Research: Firestore Security Rules — MyUniHub

**Date**: 2026-08-31 | **Branch**: `001-firestore-rules-audit`

## Resumo

Esta pesquisa consolida as decisões técnicas e padrões da indústria para implementar Firestore Security Rules no MyUniHub, dado o spec já clarificado em `/specs/001-firestore-rules-audit/spec.md`. Não há `NEEDS CLARIFICATION` no Technical Context; todas as decisões abaixo são tomadas com base em (a) investigação do código atual, (b) documentação oficial do Firebase, (c) padrões idiomáticos de produção, e (d) respostas do `/speckit-clarify` (Q1–Q5).

---

## D1 — `rules_version = '2'` (sintaxe moderna)

**Decision**: Usar `rules_version = '2';` no topo de `firestore.rules`.

**Rationale**: Pedido explícito do usuário no input do `/speckit-plan`. A v2 introduz `match`/`allow`/`function` e remove `service cloud.firestore { match /databases/... }` wrapper. É a versão atual recomendada em toda a documentação Firebase desde 2019.

**Alternatives considered**:
- **v1** (`service cloud.firestore { match /databases/{database}/documents { ... } }`): mais verbosa, mas equivalente em poder. Rejeitada por instrução do usuário.
- **v3 / beta**: não existe. v2 é a versão estável atual.

**Verificação**: https://firebase.google.com/docs/firestore/security/get-started (confirmado).

---

## D2 — Funções helper `isOwner()` e `isSignedIn()`

**Decision**: Declarar duas funções helper no topo do arquivo:

```javascript
function isSignedIn() {
  return request.auth != null;
}
function isOwner(uid) {
  return isSignedIn() && request.auth.uid == uid;
}
```

**Rationale**: Padrão canônico recomendado pela documentação. Reduz duplicação, facilita auditoria, e expressa a intenção ("is the requester the owner of this document?") de forma legível.

**Alternatives considered**:
- **Inline `request.auth.uid == uid` em cada `allow`**: menos DRY, mas viável. Rejeitada por legibilidade.
- **Custom claims (`request.auth.token.admin == true`)**: não aplicável — o app não usa custom claims.

---

## D3 — Estrutura `match /users/{uid}` com submatch futuro

**Decision**: Estrutura:

```javascript
match /users/{uid} {
  allow read, write: if isOwner(uid) && <granular_validations>;
}
```

**Rationale**:
- O `match /users/{uid}` cria o escopo onde `uid` é capturado e pode ser comparado com `request.auth.uid`.
- A validação de `isOwner(uid)` antes de tudo garante que mesmo se as validações granulares falharem por typo, o path nunca é acessível a um atacante.
- Deixa a porta aberta para sub-coleções no futuro (ex.: `match /users/{uid}/notifications/{nid}`) sem refatorar a estrutura.

**Alternatives considered**:
- **Único `allow` global no `match /databases/...`**: perderia a variável `uid` capturada. Não funciona.
- **Separar `allow read` e `allow write` com lógicas diferentes**: tentador (escrita exige validação, leitura não), mas cria risco de bypass via `read` + `update` separado. A leitura e a escrita devem ter o mesmo gate de ownership; as validações granulares aplicam-se em ambos (leitura de um campo enorme também é problema de banda).

---

## D4 — Helper `hasValidUidOrLegacy()` para FR-006

**Decision**:

```javascript
function hasValidUidOrLegacy(item) {
  return !('uid' in item) || item.uid == request.auth.uid;
}
function allItemsValidUidOrLegacy(items) {
  return items == null || items.all(hasValidUidOrLegacy, []);
}
```

Aplicado em `request.resource.data.books` e `request.resource.data.weekPlans`.

**Rationale**:
- Resolve Q1 (híbrido: aceita sem uid, rejeita uid divergente).
- Resolve Q4 (escopo só em `books` e `weekPlans`).
- `!('uid' in item)` é a forma idiomática de testar ausência em Firestore Rules (não há `item.uid == null` confiável para "campo ausente").
- A função auxiliar permite que `persistAll` no client continue substituindo o array inteiro (`setDoc`) sem ter que particionar a escrita — a regra apenas inspeciona cada item.

**Alternatives considered**:
- **Rejeitar qualquer escrita que não tenha `uid` em todos os items** (Q1-B): quebraria todos os usuários com livros antigos. Rejeitada.
- **Exigir `uid` em todos (Q1-A pura)**: aceita a injeção. Rejeitada.
- **Aplicar a `subjects` e `deadlines` também** (Q4-B/C): exigiria refatorar o client. Rejeitada por Q4-A.

---

## D5 — Validação granular de campos (FR-007b)

**Decision**: Helper `isValidProfile()`, `isValidArrayLengths()` e validações inline. Operadores Firestore disponíveis:

- `string.size()` para comprimento de string.
- `list.size()` para tamanho de array.
- `map.keys()` para acessar campos opcionais (com `('key' in m)` para testar presença).

**Rationale**:
- Resolve Q3 (substitui a falsa promessa de "1 MB total via Rules").
- `request.resource.data.X.size()` é válido e idiomático.
- Limites sugeridos na FR-007b são *defensivos* (defesa em profundidade); o limite nativo de 1 MiB é a fronteira dura.

**Alternatives considered**:
- **Limite de tamanho total via `request.resource.size()`** (Q3-A): não existe em Firestore Rules (só em Storage Rules). Confirmado em https://firebase.google.com/docs/firestore/security/rules-conditions.
- **Sem validação de campo** (rejeitada por Q3-C pragmático).

---

## D6 — Comentários de auditoria (FR-010, SC-006)

**Decision**: Cada bloco `match` e cada `allow` terá um comentário `//` acima no formato:

```javascript
// ── /users/{uid} — documento único do usuário ──
// Protege: profile, subjects, deadlines, weekPlans, books, checklist, studyHistory, timer, onboardingComplete
// Quem: request.auth.uid deve coincidir com {uid}
// Por quê: isolamento multi-tenant (User Story 1)
```

**Rationale**: Atende FR-010 ("comentários explicando cada bloco, o caminho protegido e o motivo") e SC-006 ("100% dos caminhos cobertos têm comentário explicativo"). Também ajuda o `code-review` humano.

**Alternatives considered**:
- **Sem comentários**: viola FR-010. Rejeitada.

---

## D7 — `firebase.json` mínimo

**Decision**: Arquivo `firebase.json` na raiz:

```json
{
  "firestore": {
    "rules": "firestore.rules"
  }
}
```

**Rationale**: Necessário para `firebase deploy --only firestore:rules` resolver o caminho das Rules. Pode ser estendido depois (hosting, emulators) sem reescrever.

**Alternatives considered**:
- **Sem `firebase.json`**: o `deploy` exige um `firebase.json` mínimo ou que o caminho seja passado via `--config`. O caminho-padrão é mais limpo.
- **`.firebaserc` com alias de projeto**: opcional mas recomendado para que `firebase deploy` saiba qual projeto alvejar sem flag. **Decidido incluir** um `.firebaserc` com `{ "projects": { "default": "myunihub-4bb1f" } }`.

---

## D8 — Migração silenciosa one-shot no client (Q1 follow-up)

**Decision**: No `index.html`, adicionar função `bootstrapUid()` que:

1. Verifica se já rodou nesta sessão (flag em `localStorage`).
2. Se sim, sai.
3. Se não, percorre `D.books` e `D.weekPlans` e, para cada item sem `uid`, define `uid = uid` (o module-scoped uid do usuário logado).
4. Faz `persistAll()` se houve mudança.
5. Marca a flag.

Chamada: no `startListener()` (após o primeiro `snap` chegar) ou no `onAuthStateChanged` antes do `startListener`.

**Rationale**:
- Q1 disse "migração silenciosa one-shot no client" como parte do C.
- Implementar como função idempotente e versionada em localStorage evita rodar a cada login.
- A `D.books` e `D.weekPlans` já carregam `uid` no item quando o user adiciona hoje (`book.uid = uid || null` e `plan.uid = uid || null`); a migração só precisa retroagir para os items adicionados **antes** desta feature, onde o client pode ter passado `null` (porque `uid` ainda não estava disponível, embora esteja — investigar mais no plan).

**⚠ Achado adicional da pesquisa**: Ao reler `index.html:3701` (`book.uid = uid || null`) e `index.html:2767` (`plan.uid = uid || null`), noto que o client **atualmente** está gravando `uid: null` para os items que estão sendo criados hoje, porque a chamada `persist({ books: D.books })` vem **depois** do `push` mas o `uid` está no escopo do módulo... vou marcar como ponto de atenção na fase de implementação, mas não muda a decisão: a regra ainda aceita `uid == null` (ausência ou `null` é equivalente na prática para o `hasValidUidOrLegacy`).

**Alternatives considered**:
- **Script one-time de migração no console Firebase**: não escala, exige execução manual, e "silencioso" é justamente o que o usuário pediu (Q1-C).
- **Não migrar** (Q1-A): a regra aceita sem uid, então não há urgência. Mas a migração é trivial e elimina o "ruído" legado — vale fazer.

---

## D9 — Sem extension hooks / sem CI obrigatório

**Decision**: Esta entrega **não** inclui CI nem hook de pre-commit. Razão:
- O `.specify/extensions.yml` não existe no projeto (verificado).
- Adicionar CI para uma feature de 1 arquivo + 1 config é over-engineering para v1.
- Deploy manual via `firebase deploy --only firestore:rules` é o caminho explícito pedido pelo usuário.

**Alternatives considered**:
- **Adicionar GitHub Actions para deploy automático em merge na `main`**: melhoria valiosa, mas fora do escopo desta feature. Pode ser uma feature de "DevOps" futura.

---

## Resumo das Decisões

| ID | Decisão | Spec ref |
|---|---|---|
| D1 | `rules_version = '2'` | input do usuário |
| D2 | Helpers `isSignedIn`, `isOwner` | FR-009, FR-010 |
| D3 | `match /users/{uid}` + validação granular | FR-002, FR-003, FR-004 |
| D4 | `hasValidUidOrLegacy` em books/weekPlans | FR-006 + Q1 + Q4 |
| D5 | Validação granular de strings/arrays | FR-007b + Q3 |
| D6 | Comentários de auditoria | FR-010, SC-006 |
| D7 | `firebase.json` + `.firebaserc` mínimos | FR-008 |
| D8 | Migração silenciosa one-shot no client | Q1 follow-up |
| D9 | Sem CI / sem hooks nesta entrega | pragmatismo |

Nenhuma `NEEDS CLARIFICATION` remanescente.

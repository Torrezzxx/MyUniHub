# Contract: Firestore Security Rules — MyUniHub

**Date**: 2026-08-31 | **Branch**: `001-firestore-rules-audit`

Este documento é o **contrato externo** das `firestore.rules`: define, do ponto de vista do cliente, o que é permitido e o que é negado em cada operação. É a referência para testes E2E e para o `quickstart.md`.

## Princípio Geral (Fail-Closed)

Qualquer path não explicitamente permitido em uma regra `allow ... if <condição verdadeira>` é **negado por padrão**. Isso é garantido pela regra catch-all `match /{document=**}` com `allow read, write: if false`.

## Paths Cobertos

| Path | Operação | Permitido para | Condição |
|---|---|---|---|
| `/users/{uid}` | `read` (get, list) | request.auth.uid == `{uid}` | `isOwner(uid) && isValidDocumentShape()` |
| `/users/{uid}` | `write` (create, update, delete) | request.auth.uid == `{uid}` | `isOwner(uid) && isValidDocumentShape()` |
| `/{document=**}` (qualquer outro) | qualquer | ninguém | sempre `false` |

> **Nota sobre listagem**: embora o `match /users/{uid}` use um curinga `{uid}`, **listar a coleção `users`** (queries tipo `getDocs(collection(db,'users'))`) é **negado** porque o SDK do Firestore trata `list` como uma operação que precisa de `allow list` explícita no `match` correspondente (ou em um `match` ancestral). Como esta regra **não** declara `allow list`, `getDocs(collection(db,'users'))` falha. `getDoc(doc(db,'users',X))` continua funcionando (é um `get`, não um `list`).

## Funções Helper (contrato público das Rules)

| Função | Assinatura | Semântica |
|---|---|---|
| `isSignedIn()` | `() -> bool` | `request.auth != null` |
| `isOwner(uid)` | `(uid: string) -> bool` | `isSignedIn() && request.auth.uid == uid` |
| `hasValidUidOrLegacy(item)` | `(item: map) -> bool` | Aceita item se o campo `uid` está **ausente** ou se é **igual** a `request.auth.uid`. Rejeita se está **presente e divergente**. |
| `isValidBooksField()` | `(books: list) -> bool` | `books == null \|\| (books.size() <= 200 && books.all(hasValidUidOrLegacy, []))` |
| `isValidWeekPlansField()` | `(plans: list) -> bool` | `plans == null \|\| (plans.size() <= 200 && plans.all(hasValidUidOrLegacy, []))` |
| `isValidProfileField()` | `(p: map) -> bool` | `p == null \|\| (p.name.size() <= 80 && p.course.size() <= 120 && p.institution.size() <= 120 && p.goalMinutes >= 0 && p.goalMinutes <= 1440)` |
| `isValidDocumentShape()` | `() -> bool` | Conjunção de todas as validações granulares (FR-007b) sobre `request.resource.data` |

> As funções helper são internas ao arquivo de Rules e não são expostas ao cliente; o "contrato público" é o resultado de `allow`.

## Operações e Comportamento Esperado

### Read (`getDoc`)

```javascript
// Permitido:
await getDoc(doc(db, 'users', request.auth.uid));   // ✅

// Negado:
await getDoc(doc(db, 'users', 'outro-uid'));        // ❌ permission-denied
await getDocs(collection(db, 'users'));            // ❌ permission-denied (list)
await getDoc(doc(db, 'qualquer-coisa', 'x'));      // ❌ permission-denied (catch-all)
```

### Write (`setDoc` / `updateDoc` / `deleteDoc`)

```javascript
// Permitido (escopo e shape válidos):
await setDoc(doc(db, 'users', request.auth.uid), { profile: { name: 'Ana' }, subjects: [] });
await updateDoc(doc(db, 'users', request.auth.uid), { 'profile.name': 'Beatriz' });
await deleteDoc(doc(db, 'users', request.auth.uid));

// Negado por path:
await setDoc(doc(db, 'users', 'outro-uid'), { foo: 'bar' });   // ❌

// Negado por validação de shape:
await setDoc(doc(db, 'users', request.auth.uid), {
  books: [{ id: 1, title: 'OK', uid: 'outro-uid' }]            // ❌ FR-006
});
await setDoc(doc(db, 'users', request.auth.uid), {
  profile: { name: 'x'.repeat(100) }                            // ❌ FR-007b (name > 80)
});
await setDoc(doc(db, 'users', request.auth.uid), {
  books: new Array(201).fill({ id: 1, title: 'x' })             // ❌ FR-007b (size > 200)
});

// Negado por ausência de auth:
firebase.auth().signOut();
await setDoc(doc(db, 'users', 'algum-uid'), { foo: 'bar' });   // ❌ FR-001
```

### Realtime (`onSnapshot`)

`onSnapshot` é equivalente a um `read` em termos de permissões. As Rules se aplicam na primeira leitura e em cada re-fetch automático.

```javascript
// ✅ permitido
onSnapshot(doc(db, 'users', request.auth.uid), cb);

// ❌ negado
onSnapshot(doc(db, 'users', 'outro-uid'), cb);  // permission-denied
```

## Casos da Conta Demo

A conta `demo@myunihub.app` é tratada como usuário comum. Seu `uid` (gerado pelo Firebase Auth no primeiro login) é o `request.auth.uid` para as Rules — nada mais.

```javascript
// Login demo
await signInWithEmailAndPassword(auth, 'demo@myunihub.app', 'Demo@2025!');
const demoUid = auth.currentUser.uid;

// ✅ demo lê/escreve seu próprio doc
await getDoc(doc(db, 'users', demoUid));

// ❌ demo tenta ler outro usuário
await getDoc(doc(db, 'users', 'uid-de-outro'));  // permission-denied
```

## Mensagens de Erro Esperadas

O Firestore retorna códigos estáveis para o cliente:

| Causa | Código do erro | `message` típico |
|---|---|---|
| `request.auth == null` | `permission-denied` (functions) / `unauthenticated` (REST) | "Missing or insufficient permissions" |
| `request.auth.uid != uid` | `permission-denied` | "Missing or insufficient permissions" |
| Item com `uid` divergente | `permission-denied` | "Missing or insufficient permissions" |
| Validação de shape falhou | `permission-denied` | "Missing or insufficient permissions" |
| `goalMinutes > 1440` etc. | `permission-denied` | "Missing or insufficient permissions" |
| Operação em path não permitido (catch-all) | `permission-denied` | "Missing or insufficient permissions" |

> **Importante**: o Firestore **não** detalha *por que* a permissão foi negada (anti-enumeração). O cliente só vê "permission-denied". Por isso, mensagens de UI devem ser genéricas ("Não foi possível salvar", "Tente novamente").

## Compatibilidade com o Cliente Atual

Operações que o `index.html` faz atualmente (ver `index.html:1841-1860`, `1942-1956`, `1989-1992`, `2539`, `3471-3472`, `3511-3512`) e que **devem** continuar funcionando após o deploy:

| Operação no client | Compatível? |
|---|---|
| `getDoc(doc(db,'users',uid))` em `onAuthStateChanged` | ✅ |
| `setDoc(doc(db,'users',uid), {profile, subjects:[], ...})` no registro | ✅ |
| `setDoc(doc(db,'users',uid), {profile, ...}, {merge:true})` no setup | ✅ |
| `updateDoc(doc(db,'users',uid), {onboardingComplete: true})` | ✅ |
| `updateDoc(doc(db,'users',uid), partial)` via `persist()` | ✅ (com validação granular) |
| `deleteDoc(doc(db,'users',uid))` em "apagar dados" | ✅ |
| `onSnapshot(doc(db,'users',uid), ...)` no `startListener` | ✅ |

## Limites Conhecidos (Edge Cases Cobertos)

- **Reautenticação**: tokens expirados geram `auth/user-token-expired` **antes** de chegar às Rules.
- **Coleção vazia**: `getDoc(doc(db,'users','inexistente'))` autenticado retorna "not found" (não permission-denied), preservando o comportamento atual.
- **Documento escrito parcialmente** (`updateDoc` com 1 campo): Rules validam apenas `request.resource.data` (estado pós-escrita), não os campos não tocados.

## Fora do Escopo do Contrato

- `request.time`-based staleness (Q2): last-write-wins permanece.
- Custom claims (e.g., roles de admin): não existem no app.
- Operações em outras coleções: nenhuma outra coleção é acessada.
- Storage: não há Storage configurado.
- Callable Functions: não há nenhuma exposta.

# Quickstart: Validating Firestore Security Rules — MyUniHub

**Date**: 2026-08-31 | **Branch**: `001-firestore-rules-audit`

Guia de validação **executável** das `firestore.rules`. Cobre três caminhos: (1) emulador local, (2) Rules Playground no console, (3) deploy em produção. Após o deploy, valide o app real manualmente com a checklist do final.

## Pré-requisitos

- Firebase CLI 13+ (verificado: v15.22.3 instalado em `C:\Users\Marcio\AppData\Roaming\npm\firebase`).
- Node.js 18+ (verificado: v22.15.0).
- Java JRE 11+ (necessário para o emulador Firestore).
- Acesso ao projeto Firebase `myunihub-4bb1f` (Owner ou Editor em IAM, ou Firebase Admin).
- Login CLI: `firebase login` (uma vez por máquina; use a conta que tem acesso ao projeto).

## 0. Login e sanity check

```powershell
firebase login                                       # se ainda não logado
firebase projects:list                               # confirma que myunihub-4bb1f aparece
node --version                                       # >= 18
firebase --version                                   # >= 13
```

## 1. Teste Local com Emulador (recomendado para desenvolvimento)

### 1.1 Subir o emulador

Na raiz do projeto:

```powershell
firebase emulators:start --only firestore --project myunihub-4bb1f
```

Saída esperada: o emulador inicia em `127.0.0.1:8080` (Firestore) e `127.0.0.1:4000` (UI do emulator suite). A UI permite navegar pelos documentos.

### 1.2 Apontar o app para o emulador

Para validar **sem mexer no app de produção**, crie um arquivo de teste separado (NÃO comitar) ou use uma flag de ambiente. Em `index.html`, **temporariamente** troque a chamada `getFirestore()` para detectar `localhost`:

> ⚠ Apenas para validação local. Reverta antes do commit.

```javascript
// Antes de getFirestore(app), adicione (apenas em dev):
// import { connectFirestoreEmulator } from ".../firebase-firestore.js";
// connectFirestoreEmulator(db, '127.0.0.1', 8080);
```

Alternativa mais limpa: criar `index.test.html` que importa o mesmo módulo mas com a flag de emulador.

### 1.3 Validar manualmente no app (emulador)

1. Abra `http://localhost:5000` (ou o host configurado) com o app apontando para o emulador.
2. **Cenário A — isolamento**: crie usuário A, adicione uma matéria. Crie usuário B em janela anônima. Tente no console do navegador: `getDoc(doc(db,'users','uid-de-A'))` logado como B → erro `permission-denied`.
3. **Cenário B — anônimo**: deslogado, no console: `getDoc(doc(db,'users','qualquer-uid'))` → erro `permission-denied`.
4. **Cenário C — demo**: clique em "Explorar em Modo Demo" → adicione um prazo → recarregue → o prazo persiste.
5. **Cenário D — integridade de uid**: logado como A, no console: `setDoc(doc(db,'users',A.uid), { books: [{id:1, title:'X', uid:'outro-uid'}] }, {merge:true})` → erro `permission-denied`.
6. **Cenário E — limite de campo**: logado, no console: `setDoc(doc(db,'users',A.uid), { profile: { name: 'X'.repeat(200) } }, {merge:true})` → erro `permission-denied`.

### 1.4 Test runner oficial (opcional, recomendado para CI futuro)

```powershell
npm init -y
npm i --save-dev @firebase/rules-unit-testing mocha
```

Criar `tests/firestore/rules.spec.js`:

```javascript
const { initializeTestEnvironment, assertFails, assertSucceeds } =
  require('@firebase/rules-unit-testing');
const { doc, getDoc, setDoc } = require('firebase/firestore');

const env = await initializeTestEnvironment({
  projectId: 'demo-myunihub',
  firestore: { rules: require('fs').readFileSync('firestore.rules','utf8') }
});

it('bloqueia leitura anônima', async () => {
  const ctx = env.unauthenticatedContext();
  await assertFails(getDoc(doc(ctx.firestore(), 'users', 'qualquer')));
});

it('permite leitura do próprio doc', async () => {
  const ctx = env.authenticatedContext('uid-A');
  await assertSucceeds(getDoc(doc(ctx.firestore(), 'users', 'uid-A')));
});

it('bloqueia leitura cruzada', async () => {
  const ctx = env.authenticatedContext('uid-B');
  await assertFails(getDoc(doc(ctx.firestore(), 'users', 'uid-A')));
});

it('rejeita uid divergente em books', async () => {
  const ctx = env.authenticatedContext('uid-A');
  await assertFails(setDoc(doc(ctx.firestore(),'users','uid-A'),
    { books: [{ id:1, title:'X', uid:'outro-uid' }] }));
});

it('aceita uid ausente em books (legado)', async () => {
  const ctx = env.authenticatedContext('uid-A');
  await assertSucceeds(setDoc(doc(ctx.firestore(),'users','uid-A'),
    { books: [{ id:1, title:'X' }] }));
});

it('rejeita name > 80', async () => {
  const ctx = env.authenticatedContext('uid-A');
  await assertFails(setDoc(doc(ctx.firestore(),'users','uid-A'),
    { profile: { name: 'X'.repeat(81) } }));
});

await env.cleanup();
```

Rodar com `node --test tests/firestore/rules.spec.js` (Node 18+ tem test runner nativo) ou `npx mocha tests/firestore`.

> Este arquivo de teste **não** é parte da entrega desta feature — é recomendação para PR futuro.

## 2. Rules Playground (testes pontuais no console)

Para validar regras **diretamente no console Firebase**, sem rodar emulador local:

1. Abra https://console.firebase.google.com/project/myunihub-4bb1f/firestore/rules.
2. Clique em **Rules Playground** (ou "Playground" no menu lateral).
3. Para cada cenário, preencha:
   - **Location**: `users/{uid}` (substitua `{uid}` por um uid de teste real do projeto).
   - **Method**: `get`, `create`, `update` ou `delete`.
   - **Auth**: `Authenticated` (com uid `A` ou `B` para isolamento) ou `Unauthenticated`.
   - **Document data**: JSON que simula `request.resource.data`.
4. Clique em **Run** e verifique se a regra **Allows** ou **Denies**.

### 2.1 Cenários a executar no Playground

| # | Path | Method | Auth uid | Expected |
|---|---|---|---|---|
| 1 | `users/A` | get | A | Allows |
| 2 | `users/A` | get | B | Denies |
| 3 | `users/A` | get | (unauth) | Denies |
| 4 | `users/A` | update | A (dados ok) | Allows |
| 5 | `users/A` | update | A (`books: [{uid:'X'}]` com X != A) | Denies |
| 6 | `users/A` | update | A (`books: [{id:1}]` sem uid) | Allows (legado) |
| 7 | `users/A` | update | A (`profile.name` > 80) | Denies |
| 8 | `users/A` | update | A (`subjects` length > 30) | Denies |
| 9 | `qualquer-coisa/X` | get | A | Denies (catch-all) |
| 10 | `users` (collection) | list | A | Denies |

## 3. Deploy em Produção

### 3.1 Pré-deploy checklist

- [ ] `firestore.rules` revisado e commitado na branch.
- [ ] `firebase.json` revisado.
- [ ] `.firebaserc` presente e com o alias `default: myunihub-4bb1f`.
- [ ] Pelo menos um cenário do Playground (item 2) executado com sucesso.
- [ ] Pelo menos um cenário do emulador (item 1) executado com sucesso.
- [ ] Migração silenciosa do client pronta (verificar `bootstrapUid` no `index.html`).

### 3.2 Comando de deploy

```powershell
firebase deploy --only firestore:rules
```

Saída esperada: `✔ Deploy complete!`. O console Firebase vai mostrar a nova versão em **Rules → Activity** (timestamp + autor).

### 3.3 Rollback (se necessário)

No console Firebase → Firestore → Rules → **Activity** (ou **Versions**), selecione a versão anterior e clique em **Restore**. O rollback é imediato. Alternativa CLI:

```powershell
# (a partir de 2024 o rollback programático está disponível; senão use o console)
```

### 3.4 Validação pós-deploy no app de produção

1. Abra o app em `https://myunihub-4bb1f.web.app` (ou a URL de produção configurada).
2. **Cenário usuário comum** (SC-004): registre-se → setup → adicione matéria, prazo, livro → logout → login → tudo persiste.
3. **Cenário demo** (SC-003): clique "Explorar em Modo Demo" → adicione um prazo → recarregue → prazo persiste.
4. **Cenário isolamento** (SC-001): abra duas janelas anônimas, cadastre A e B, e confirme via DevTools que `getDoc(doc(db,'users',A.uid))` logado como B falha.
5. **Cenário anônimo** (SC-002): deslogado, no DevTools, `getDoc(doc(db,'users','algum-uid'))` falha.

## 4. Critérios de Done (mapping com Success Criteria)

| SC | Como verificar |
|---|---|
| SC-001 (0% cross-uid) | Cenário 2 do Playground + passo 4 da validação pós-deploy |
| SC-002 (0% anônimo) | Cenário 3 do Playground + passo 5 da validação pós-deploy |
| SC-003 (demo OK) | Passo 3 da validação pós-deploy |
| SC-004 (comum OK) | Passo 2 da validação pós-deploy |
| SC-005 (deploy reversível) | `firebase deploy` + botão Restore no console |
| SC-006 (legibilidade) | Inspeção visual de `firestore.rules` — cada bloco comentado |

## 5. Troubleshooting

| Sintoma | Causa provável | Solução |
|---|---|---|
| `firebase deploy` falha com "auth" | Não logado ou sem permissão | `firebase login`; verifique IAM |
| Emulador não sobe | Java não instalado | Instale JRE 11+ |
| App retorna `permission-denied` em escrita legítima | Validação de shape (FR-007b) rejeitou | Verifique no console: `firestore.googleapis.com/.../users/{uid}` mostra o que foi rejeitado? Em caso afirmativo, ajuste a regra ou o client |
| Lista de `users` falha (esperado) | Sem `allow list` (correto) | Comportamento desejado — SC-001 |
| Migração silenciosa não roda | Flag `localStorage` já marcada de uma sessão anterior | Limpe `localStorage` em DevTools ou aguarde — é one-shot |
| Rollback não disponível via CLI | Limitação do Firebase CLI | Use o console (Activity → Restore) |

## 6. Próximos passos (fora do escopo desta feature)

- Adicionar `@firebase/rules-unit-testing` ao CI (GitHub Actions, etc.).
- Adicionar App Check (mencionado no spec, Q5) para proteger a apiKey contra abuso.
- Migrar a operação de `setDoc` que substitui arrays inteiros para `updateDoc` com merge — reduzir risco de perda em concorrência multi-dispositivo (Q2, feature futura).

# Feature Specification: GitGuardian Remediation — MyUniHub

**Feature Branch**: `002-gitguardian-remediation`

**Created**: 2026-08-31

**Status**: Draft

**Input**: Remediação completa dos 4 findings (1 HIGH, 3 MEDIUM) do relatório GitGuardian `gitguard-report-Torrezzxx-MyUniHub-yz1dv57o.md` (scan `cmt8unj6201oz10d7yz1dv57o`, executado em 2026-08-25), consolidado com o deploy das Firestore Security Rules como mitigação de defesa em profundidade.

## Contexto e Findings

O relatório GitGuardian identificou **4 findings** no branch `main` no commit `55e07b4df121608959f770881da904a888ee6b17`:

| # | Severidade | Scanner | Categoria | Resumo |
|---|---|---|---|---|
| F1 | 🔴 **HIGH** | GITLEAKS | Secret leak | GCP API key (a do Firebase) detectada em `index.html` |
| F2 | 🟡 MEDIUM | SEMGREP | SAST | `<script>` sem Subresource Integrity (SRI) — Tag 1 |
| F3 | 🟡 MEDIUM | SEMGREP | SAST | `<script>` sem SRI — Tag 2 |
| F4 | 🟡 MEDIUM | SEMGREP | SAST | `<script>` sem SRI — Tag 3 |

Investigação no `index.html` confirmou a localização exata:

- **F1**: `index.html:1795` — `apiKey: "AIzaSyDhKMwHNZTNjXp_SZ72lmil0_NOKFzXeMs"` (Firebase project `myunihub-4bb1f`).
- **F2, F3, F4**: tags `<script src="https://...">` em:
  - linha 23: `https://cdn.tailwindcss.com`
  - linha 71: `https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js`
  - linhas 1784, 1788, 1791: imports ESM do Firebase (`firebase-app.js`, `firebase-auth.js`, `firebase-firestore.js`) — embora o Semgrep possa não contar cada import individualmente, **também não têm SRI** e devem ser tratados.

Observação de segurança: a presença da `apiKey` em código client-side do Firebase é **esperada e documentada** (a segurança depende das Firestore Rules). O risco real é a **ausência de restrições** na key (HTTP referrers, APIs permitidas) que permite abuso por atacantes. **Restrições são o remédio crítico**; a "remoção" da key do código não é viável em apps Firebase client-side.

## User Scenarios & Testing

### User Story 1 — Restrição e rotação da API key exposta (Priority: P1)

A GCP API key do Firebase exposta em `index.html:1795` é tratada como credencial comprometida até prova em contrário. O time aplica **restrições de segurança** (HTTP referrers + APIs permitidas) e, se houver indício de abuso, **rotaciona a key** gerando uma nova e atualizando o `index.html` com a nova. O app continua funcionando normalmente com a nova key, sem expor a antiga.

**Why this priority**: É o único finding HIGH e o vetor de ataque mais crítico. Sem restrições, qualquer site mal-intencionado pode usar a key para consumir a cota do projeto Firebase ou acessar serviços GCP habilitados. Rotação é defesa em profundidade caso a key já tenha sido indexada por bots (GitGuardian, scanners de credential stuffing, etc.).

**Independent Test**: Após aplicar restrições, testar que requisições vindas de um referrer **fora** da allowlist retornam `403 PERMISSION_DENIED` (ou erro equivalente). Após rotação (se aplicável), testar que o app continua funcionando (login demo + persistência) com a nova key.

**Acceptance Scenarios**:
1. **Given** a key atual sem restrições no GCP, **When** alguém faz uma chamada REST à API do Firebase a partir de um domínio não autorizado (ex.: `https://evil.com`), **Then** a chamada é negada com erro `403`.
2. **Given** o projeto Firebase tem a key atual, **When** é aplicada restrição de HTTP referrer para `*.web.app` e `*.firebaseapp.com` e localhost, **Then** chamadas vindas desses domínios continuam funcionando normalmente.
3. **Given** uma key nova foi gerada, **When** ela é colocada em `index.html` no lugar da antiga, **Then** o app continua funcionando (registro, login, persistência).
4. **Given** a key antiga foi comprometida, **When** a rotação é concluída, **Then** a key antiga é revogada no GCP e requisições com a key antiga retornam `400 API_KEY_INVALID`.

---

### User Story 2 — Subresource Integrity (SRI) em todos os scripts externos (Priority: P1)

Cada `<script src="https://...">` em `index.html` passa a carregar com atributo `integrity="sha384-..."` e `crossorigin="anonymous"`, garantindo que o navegador recuse o script se o conteúdo do CDN for alterado. O app continua funcionando normalmente porque os hashes são calculados sobre o conteúdo real atual.

**Why this priority**: 3 dos 4 findings do GitGuardian são MEDIUM sobre SRI. Mitigação barata (atributo em uma tag) e protege contra comprometimento de CDN (ataque à cadeia de suprimentos). A gravidade é MEDIUM e não HIGH, mas é "free win" e alinhada com a postura de segurança.

**Independent Test**: Após adicionar SRI, abrir o app e verificar que:
- Todos os scripts carregam (DevTools → Network → status 200).
- Se alguém trocar o conteúdo de `chart.umd.min.js` no CDN, o navegador **bloqueia** o script e o app mostra erro no console.

**Acceptance Scenarios**:
1. **Given** o `<script>` do Tailwind em `index.html:23`, **When** recebe o atributo `integrity="sha384-..."` com o hash correto do conteúdo atual, **Then** o script é executado normalmente e o styling do app é aplicado.
2. **Given** o `<script>` do Chart.js em `index.html:71`, **When** recebe `integrity` e `crossorigin`, **Then** Chart.js executa e o dashboard de progresso renderiza.
3. **Given** os 3 imports ESM do Firebase em `index.html:1784, 1788, 1791`, **When** recebem `integrity` e `crossorigin`, **Then** o app inicializa normalmente (login, persistência).
4. **Given** um atacante consegue alterar o conteúdo de `chart.umd.min.js` no CDN, **When** o navegador tenta carregá-lo, **Then** o navegador **bloqueia** a execução por falha no `integrity check` e exibe erro no console.

---

### User Story 3 — Deploy das Firestore Security Rules como mitigação de defesa em profundidade (Priority: P1)

Como parte desta remediação, as Firestore Security Rules desenvolvidas na feature `001-firestore-rules-audit` são deployadas em produção, garantindo que mesmo que a apiKey seja abusada, os dados dos usuários permaneçam isolados por uid e inacessíveis a anônimos.

**Why this priority**: O GitGuardian reportou o vazamento da key, mas não cobre o que um atacante **pode fazer com ela**. As Rules reduzem drasticamente a superfície de ataque: com elas, mesmo um cliente "modificado" (que burla o app) só consegue ler/escrever o **próprio** documento de usuário, e nada anônimo funciona. É a mitigação que **transforma a key exposta de vetor crítico em vetor de baixo risco**.

**Independent Test**: Ver `specs/001-firestore-rules-audit/quickstart.md` §2.1 cenários 1, 2, 3 (isolamento por uid, bloqueio anônimo, catch-all).

**Acceptance Scenarios**:
1. **Given** as Rules estão deployadas, **When** um atacante com a apiKey tenta `getDoc(doc(db,'users','uid-de-outra-pessoa'))` a partir de um cliente modificado, **Then** a operação é negada com `permission-denied`.
2. **Given** as Rules estão deployadas, **When** uma requisição anônima (sem Firebase Auth) tenta ler qualquer documento, **Then** a operação é negada.
3. **Given** as Rules estão deployadas e o app real é usado, **When** o usuário demo adiciona um prazo, **Then** ele persiste e é lido de volta normalmente (sem regressão funcional).

---

### User Story 4 — Documentação e processo de segurança (Priority: P2)

Cria/atualiza documentação no repositório que:
- Lista as credenciais GCP expostas e como foram tratadas.
- Explica a política de segurança da apiKey do Firebase em apps client-side.
- Adiciona `SECURITY.md` (ou similar) na raiz descrevendo como reportar vulnerabilidades.
- Atualiza o `README.md` mencionando que o projeto usa Firestore Security Rules.

**Why this priority**: Não é um finding técnico, mas é o que diferencia um projeto amador de um maduro. Manter essa documentação facilita auditoria futura e onboarding de novos contribuidores. Pode ser feito em paralelo com US1-US3.

**Independent Test**: `README.md` e `SECURITY.md` (se criado) contêm as informações listadas; `git log` mostra que mudanças relacionadas à segurança são rastreáveis.

**Acceptance Scenarios**:
1. **Given** um novo contribuidor clona o repositório, **When** lê `README.md` e `SECURITY.md`, **Then** entende (a) que a apiKey do Firebase é client-side por design, (b) que a segurança depende das Rules, (c) como reportar uma vulnerabilidade.
2. **Given** o relatório GitGuardian é público, **When** alguém compara com o estado atual do repositório, **Then** todas as ações de remediação estão visíveis no histórico do Git.

---

### Edge Cases

- **CDN indisponível durante o build do hash SRI**: se o CDN estiver fora do ar, o hash não pode ser calculado. Mitigação: o hash é calculado uma vez e commitado; se o CDN cair depois, o navegador mostra erro (SRI falhará), mas o app não fica "preso" esperando o script.
- **Chart.js atualiza versão automaticamente (CDN sem versão fixa)**: hoje está fixo em `chart.js@4.4.0`, então o hash é estável. Se for trocado por uma URL sem versão fixa, o SRI quebrará no próximo load — política do projeto deve manter versão fixa.
- **Tailwind via CDN sem versão fixa**: o link atual é `https://cdn.tailwindcss.com` (sem `@versão`). O conteúdo pode mudar sem aviso → SRI pode quebrar. Mitigação: substituir por uma versão fixa (`?v=3.4.0` ou `tailwindcss@3.4.0/...`) antes de calcular o hash.
- **Key antiga ainda referenciada em cache de CDN ou em issue/comment antigo**: a rotação isola o app atual, mas o histórico Git fica com a key antiga. Mitigação: BFG Repo-Cleaner (opcional, fora do escopo padrão); documentar a key como comprometida para evitar confusão futura.
- **Firestore Security Rules quebram o app por erro de sintaxe**: deploy mal feito pode derrubar o app. Mitigação: validar com `firebase emulators:start` antes do deploy; ter plano de rollback documentado (console Firebase → Rules → Activity → Restore).
- **API key do Firebase **precisa** estar exposta** (não dá para mover para variável de ambiente em SPA estática): sim, é esperado. A segurança vem de (a) restrições GCP, (b) Firestore Rules, (c) App Check (futuro).

## Requirements

### Functional Requirements

- **FR-001**: A GCP API key em `index.html:1795` MUST ter **restrição de HTTP referrer** aplicada no Google Cloud Console, permitindo apenas os domínios oficiais do app (`*.web.app`, `*.firebaseapp.com`) e localhost em dev.
- **FR-002**: A mesma key MUST ter **restrição de API** aplicada, permitindo apenas as APIs do Firebase usadas pelo app (IdentityToolkit, Firestore, Cloud Storage se aplicável). APIs não usadas (Maps, Translate, etc.) MUST ser desabilitadas.
- **FR-003**: Se houver **qualquer indício de abuso** (cota anômala, chamadas de origens suspeitas no logs do GCP), a key MUST ser **rotacionada** e o `index.html` atualizado com a nova key em até 24h.
- **FR-004**: Cada `<script src="https://...">` em `index.html` (5 tags: Tailwind, Chart.js, e 3 imports ESM do Firebase) MUST ter atributos `integrity="sha384-..."` e `crossorigin="anonymous"`.
- **FR-005**: Os hashes `integrity` MUST ser **calculados sobre o conteúdo exato** do recurso remoto no momento do commit (não chutados). Comando sugerido: `curl -s URL | openssl dgst -sha384 -binary | openssl base64 -A`.
- **FR-006**: Antes de adicionar SRI, as URLs MUST ser **fixadas em versão** (ex.: `chart.js@4.4.0/dist/chart.umd.min.js` já é; Tailwind CDN sem versão precisa ser fixada). URLs com versão não-fixada MUST ser corrigidas antes do SRI.
- **FR-007**: As Firestore Security Rules (features `001-firestore-rules-audit`) MUST ser deployadas em produção, com pelo menos o MVP (T001 + T005 + T006 + T007) antes desta remediação ser considerada completa.
- **FR-008**: O `README.md` MUST ser atualizado com uma seção "Security" explicando (a) que a apiKey é client-side por design, (b) que a segurança depende das Firestore Rules, (c) referência à `SECURITY.md` se existir.
- **FR-009**: Um arquivo `SECURITY.md` SHOULD ser criado na raiz descrevendo: (a) política de report de vulnerabilidades, (b) o que **não** é considerada vulnerabilidade (e.g., presença da apiKey no HTML).
- **FR-010**: Cada ação de remediação MUST ser commitada em um commit separado e rastreável (ex.: `chore(security): restrict firebase api key http referrers`, `chore(security): add sri to cdn scripts`).
- **FR-011**: Após cada commit, o app MUST ser testado manualmente para garantir que **nenhuma** das alterações quebrou o funcionamento (login, persistência, gráficos, etc.).

### Key Entities

- **GCP API key**: string alfanumérica em `index.html:1795`. Não é um "segredo" no sentido tradicional (apps Firebase client-side a expõem), mas deve ser restrita no GCP para limitar abuso.
- **Recursos CDN externos**: 5 URLs com `<script>` carregando de domínios terceiros. Cada um precisa de hash SRI.
- **Firestore Rules**: arquivo `firestore.rules` (a ser deployado conforme `001-firestore-rules-audit`).
- **Relatório GitGuardian**: artefato externo em `c:\Users\Marcio\Downloads\gitguard-report-...md` que disparou esta feature. Após remediação, é evidência da auditoria.

## Success Criteria

- **SC-001**: O GCP Cloud Console mostra a apiKey com **restrição de HTTP referrer** aplicada, contendo pelo menos os domínios `*.web.app`, `*.firebaseapp.com` e `localhost` na allowlist.
- **SC-002**: O GCP Cloud Console mostra a apiKey com **restrição de API** aplicada, contendo apenas Firebase APIs usadas pelo app (e nenhuma API não usada).
- **SC-003**: As 5 tags `<script src="https://...">` em `index.html` têm atributos `integrity` e `crossorigin` válidos; o app carrega sem erros de SRI no console.
- **SC-004**: As Firestore Security Rules estão deployadas no projeto `myunihub-4bb1f` e os 10 cenários do `quickstart.md` §2.1 passam (ou têm resultado conhecido).
- **SC-005**: O `README.md` e (opcionalmente) `SECURITY.md` contêm a seção de segurança; o Git log mostra ≥3 commits relacionados a esta remediação.
- **SC-006**: O fluxo demo (login → adicionar prazo → reload) **continua funcionando 100%** após todas as mudanças.
- **SC-007**: O fluxo de usuário comum (registro → setup → adicionar matéria/prazo/livro → logout → login → ver tudo persistido) **continua funcionando 100%** após todas as mudanças.

## Assumptions

- O time tem **acesso de Owner/Editor** ao Google Cloud Console do projeto `myunihub-4bb1f` (necessário para aplicar restrições na key).
- O time tem **acesso ao Firebase Console** para deployar as Rules (já verificado: Firebase CLI v15.22.3 + Node.js v22.15.0 estão instalados; `firebase login` precisa ser executado uma vez).
- A apiKey atual é **real** (não placeholder). O relatório GitGuardian diz que pode ter sido "propositalmente redigido", mas o valor que aparece no `index.html` (redigido por mim aqui também, por segurança) está no formato correto de uma GCP key (`AIzaSy...`, 39 chars). Vale confirmar antes de rotacionar.
- As **Firestore Rules** da feature `001-firestore-rules-audit` serão mergeadas em `main` **antes ou em conjunto** com esta feature (a ordem pode ser a mesma PR se a equipe decidir; o spec atual não força ordem).
- O **CDN do Tailwind** sem versão fixa será substituído por uma versão fixa antes do SRI. A URL recomendada é `https://cdn.tailwindcss.com/3.4.0` (versão estável atual). Versão 4 ainda em alpha → manter 3.x.
- O **Chart.js** já está fixado em `chart.js@4.4.0` (`index.html:71`), então o SRI é estável.
- Os **módulos do Firebase** já estão fixados em `firebasejs/11.0.1` (`index.html:1784, 1788, 1791`), então o SRI é estável.
- Não há **processo de CI/CD** automatizado nesta feature (D9 do spec das Rules); deploys são manuais via Firebase CLI.
- A **rotação da key** (FR-003) é **condicional** — só é feita se houver indício de abuso. Sem logs de uso anômalo, a rotação pode ser pulada (as restrições em si já mitigam o risco).
- Esta feature **não inclui** a aquisição de um plano PROguard do GitGuardian; a análise é feita uma vez e as ações são manuais.

## Out of Scope

- **BFG Repo-Cleaner** para reescrever o histórico Git e remover a key antiga. É uma operação pesada (force-push) e a key já é conhecida como comprometida. O GitGuardian e a rotação cobrem o aspecto prático; a limpeza do histórico é nice-to-have e pode ser feita em feature separada.
- **App Check** (recomendado no spec anterior como "próxima feature"). Continua sendo candidato natural a próxima feature independente.
- **Auditoria de TODAS as dependências** (npm audit, Snyk, etc.). O projeto é client-side puro sem `package.json`; a única "dependência" é o Firebase via CDN, e o SRI já cobre.
- **Migrar a apiKey para variável de ambiente**: não é viável em SPA estática. O build/hosting teria que injetar em tempo de execução, o que exige Firebase Hosting + configuração. Pode ser feito em feature futura.
- **Política de gestão de segredos** (ex.: Google Secret Manager) — overkill para uma SPA estática.

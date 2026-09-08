# Feature Specification: Firestore Security Rules — MyUniHub

**Feature Branch**: `001-firestore-rules-audit`

**Created**: 2026-08-31

**Status**: Draft

**Input**: User description: "Auditar e implementar Firestore Security Rules para o MyUniHub, cobrindo o modelo de dados existente. Investigue primeiro a estrutura completa dos documentos Firestore usados (perfil do usuário, dados de setup, livros da biblioteca, prazos do calendário interno, planos semanais) e proponha regras que: (1) garantam que cada usuário só possa ler/escrever seus próprios dados, (2) protejam contra escrita não autenticada, (3) sejam compatíveis com a conta de modo demo (demo@myunihub.app) sem abrir brechas de segurança para outros usuários."

## Visão Geral do Modelo de Dados (Investigação Prévia)

A aplicação MyUniHub armazena **um único documento por usuário** em `users/{uid}`. Não há subcoleções. O documento é criado no momento do registro (`createUserWithEmailAndPassword`) e continuamente atualizado via `setDoc`/`updateDoc`. Existe também uma conta de modo demo (`demo@myunihub.app` / `Demo@2025!`) que compartilha o mesmo fluxo de autenticação e armazenamento.

Estrutura do documento `users/{uid}`:

| Campo | Tipo | Descrição |
|---|---|---|
| `profile` | object | `{ name, course, institution, semester, goalMinutes, semStart, semEnd }` |
| `subjects` | array<object> | Grade horária semanal: `{ id, name, prof, day, start, end, room, color }` |
| `deadlines` | array<object> | Prazos: `{ id, title, type, prio, date, time, subj, notes, completed, completedAt }` |
| `weekPlans` | array<object> | Planos semanais: `{ id, subj, topic, date, dateMs, prio, estTime, status, done, notes, resources, createdAt, uid }` |
| `books` | array<object> | Biblioteca: `{ id, title, author, totalPages, currentPage, deadline, deadlineMs, pagesPerDay, done, createdAt, uid }` |
| `checklist` | object | `{ date, items: [...] }` |
| `studyHistory` | object<string, number> | Mapa `data → minutos estudados` |
| `timer` | object | `{ date, totalToday, overtime }` |
| `onboardingComplete` | boolean | Flag do tutorial |

Observações de segurança identificadas na investigação:
- A `apiKey` do Firebase está exposta no `index.html` (esperado em apps client-side; a segurança depende das Rules).
- O client já grava `uid` dentro de alguns itens (`weekPlans[i].uid`, `books[i].uid`) — a regra pode exigir que esse valor interno bata com `request.auth.uid`.
- Há uma função `demoLogin` que cria a conta `demo@myunihub.app` sob demanda se não existir — o fluxo de auth é o mesmo do usuário comum, portanto as Rules não precisam de tratamento especial além de garantir que o `uid` da demo continue isolado.

## Clarifications

### Session 2026-08-31

- Q: Como tratar itens legados (livros e planos) sem campo `uid` interno em operações de escrita? → A: Abordagem híbrida: aceitar itens sem `uid` (tratados como legítimos e legados), rejeitar `uid` presente mas diferente de `request.auth.uid`. Cliente faz migração silenciosa one-shot (preencher `uid` em todos os itens na primeira escrita legítima após deploy).
- Q: Qual a estratégia para edições concorrentes entre múltiplos dispositivos do mesmo usuário? → A: Ignorar — last-write-wins do Firestore permanece. Concorrência (merge, CRDT, indicadores "outro dispositivo editando") fica fora do escopo desta feature e será tratada em feature futura no Roadmap.
- Q: A FR-007 (limite de 1 MB) deve ser implementada como regra explícita ou removida, confiando no limite nativo do Firestore? → A: Remover a promessa de "1 MB total via Rules" (o Firestore já aplica nativamente 1.048.576 bytes/doc, e a linguagem de Rules não expõe operador de tamanho total). Substituir por FR-007b com validações granulares de strings e arrays como defesa em profundidade.
- Q: A integridade do `uid` interno (FR-006) deve cobrir SOMENTE `books` e `weekPlans` ou estender-se também a `deadlines` e `subjects`? → A: Manter escopo atual — só `books` e `weekPlans` recebem a regra de integridade de `uid`. `subjects` e `deadlines` confiam no isolamento por path (que já é a defesa primária e suficiente). Estender agora seria escopo creep desnecessário.
- Q: Quais itens estão explicitamente fora do escopo desta feature e devem ser endereçados separadamente? → A: Out-of-scope pragmático: rate limiting server-side, observabilidade/logging de tentativas negadas, anti-scraping, retenção/LGPD, multi-region, Cloud Armor. Candidato natural a próxima feature: Firebase App Check (attestation de origem, complementa a defesa da apiKey pública).

## User Scenarios & Testing

### User Story 1 — Isolamento por usuário (Priority: P1)

Cada usuário autenticado do MyUniHub só consegue ler e modificar **o próprio documento** `users/{seu_uid}`. Se um usuário autenticado (incluindo o demo) tentar ler ou escrever em `users/{outro_uid}`, a operação é negada silenciosamente pelo Firestore e o client recebe um erro de permissão.

**Why this priority**: É o requisito central de segurança multi-tenant. Sem isso, qualquer usuário autenticado poderia ler/alterar dados de outros. É a base para todos os outros cenários.

**Independent Test**: Logar como `demo@myunihub.app`, tentar `getDoc(doc(db,'users','uid-de-outro-usuario'))` no console do navegador — deve falhar com `permission-denied`. Adicionalmente, o app não pode ter funcionalidade que dependa de ler dados de outros usuários.

**Acceptance Scenarios**:
1. **Given** usuário A autenticado, **When** tenta `getDoc(users/{B})`, **Then** recebe erro `permission-denied`.
2. **Given** usuário A autenticado, **When** tenta `setDoc(users/{B}, ...)`, **Then** recebe erro `permission-denied`.
3. **Given** usuário A autenticado, **When** faz `getDoc(users/{seu_uid})`, **Then** recebe seus próprios dados.
4. **Given** usuário A autenticado, **When** faz `setDoc`/`updateDoc` em `users/{seu_uid}`, **Then** a operação é permitida.

---

### User Story 2 — Bloqueio de acesso não autenticado (Priority: P1)

Qualquer requisição ao Firestore sem um usuário autenticado (anônimo) é negada. Nenhum endpoint de leitura ou escrita fica acessível a partir do projeto público sem credenciais válidas.

**Why this priority**: Sem essa camada, qualquer pessoa com a `apiKey` pública (visível no HTML) poderia enumerar/ler documentos via REST. É a defesa primária contra scraping e abuso de cota.

**Independent Test**: Com o app deslogado (ou via REST anônimo com `curl`), tentar `firestore.googleapis.com/v1/projects/myunihub-4bb1f/databases/(default)/documents/users/qualquer-uid` — deve retornar `403 Forbidden` ou `401`.

**Acceptance Scenarios**:
1. **Given** requisição sem `Authorization` válido, **When** tenta ler `users/{qualquer}`, **Then** é negada.
2. **Given** requisição sem `Authorization` válido, **When** tenta escrever em `users/{qualquer}`, **Then** é negada.
3. **Given** requisição sem `Authorization` válido, **When** lista a coleção `users`, **Then** é negada (não há listagens públicas).

---

### User Story 3 — Compatibilidade com a conta demo (Priority: P2)

A conta de modo demo (`demo@myunihub.app`) continua funcionando normalmente: consegue logar, ler e gravar o próprio documento. Ao mesmo tempo, a conta demo **não** ganha nenhum privilégio extra — continua sujeita às mesmas regras de isolamento dos demais usuários.

**Why this priority**: A conta demo é parte da jornada de "experimentar antes de cadastrar" e é referenciada na UI. Bloqueá-la quebraria o onboarding. Mas tratá-la de forma especial (ex.: regras `isDemo()` abrindo exceções) seria um anti-padrão grave. O teste aqui é que as regras genéricas, sem exceção, sirvam para a conta demo.

**Independent Test**: Login como `demo@myunihub.app`, adicionar um prazo, recarregar a página, ver o prazo persistir. Sair e logar com outro e-mail — não deve ver os prazos da demo.

**Acceptance Scenarios**:
1. **Given** demo autenticada, **When** adiciona/remove itens em `users/{uid_da_demo}`, **Then** persiste e reflete em real-time.
2. **Given** usuário comum X autenticado, **When** tenta ler `users/{uid_da_demo}`, **Then** recebe `permission-denied`.
3. **Given** demo autenticada, **When** tenta ler `users/{uid_X}`, **Then** recebe `permission-denied`.

---

### User Story 4 — Integridade do `uid` interno (Priority: P3)

Quando o cliente grava itens em arrays (`books`, `weekPlans`, etc.) que carregam um campo `uid`, o valor gravado deve ser igual ao `request.auth.uid`. Isso impede que um cliente malicioso (ou com bug) "rotule" um item com o uid de outro usuário — uma defesa em profundidade contra inconsistências de dados, mesmo que o isolamento por path já baste.

**Why this priority**: Defesa em profundidade. Não é estritamente necessária para o isolamento (o path já garante), mas reduz superfície de bugs onde um item de um usuário apareça em uma listagem por `array-contains` ou similar.

**Independent Test**: Em um teste local, criar um documento com `books: [{ uid: 'outro-uid' }]` e verificar que a write é rejeitada. Sem essa regra, seria aceita.

**Acceptance Scenarios**:
1. **Given** usuário A gravando `books` em `users/{A}`, **When** algum item tem `uid != request.auth.uid`, **Then** a write é rejeitada.
2. **Given** usuário A gravando `weekPlans` em `users/{A}`, **When** algum item tem `uid != request.auth.uid`, **Then** a write é rejeitada.

---

### Edge Cases

- **Coleção inexistente**: requests para `users/inexistente` autenticados devem retornar "not found" (não "permission denied"), preservando o comportamento atual.
- **Renomeação do uid**: Auth UIDs são imutáveis no Firebase; não há migração de path.
- **Reautenticação**: tokens expirados são tratados pelo SDK do Firebase e resultam em `auth/user-token-expired` antes de chegar às Rules; as Rules não precisam tratar isso diretamente.
- **Múltiplas abas/dispositivos**: o `onSnapshot` é por sessão; múltiplas instâncias do mesmo `uid` devem coexistir sem interferência. A estratégia de concorrência entre dispositivos é **last-write-wins** (comportamento padrão do Firestore); esta feature **não** tenta prevenir ou detectar edições concorrentes. Concorrência real (merge, CRDT, indicadores de "outro dispositivo editando") é tratada como item fora do escopo, em feature futura no Roadmap. Ver Clarifications → Q2.
- **Itens legados sem `uid` interno**: livros e planos criados antes dessa implementação podem não ter o campo `uid` no item. A regra aceita itens sem `uid` (tratados como legítimos e legados) **mas** rejeita itens em que `uid` está presente e **diferente** de `request.auth.uid`. O cliente é responsável por preencher `uid` em todos os itens na primeira escrita legítima após o deploy (migração silenciosa one-shot). Ver FR-006 e Clarifications → Q1.

## Requirements

### Functional Requirements

- **FR-001**: Rules MUST negar todo `read` e `write` em qualquer path do Firestore quando `request.auth == null`.
- **FR-002**: Rules MUST permitir que um usuário autenticado leia e escreva **apenas** o documento `users/{seu_próprio_uid}`.
- **FR-003**: Rules MUST negar leitura de `users/{outro_uid}` mesmo para usuários autenticados.
- **FR-004**: Rules MUST negar listagem da coleção `users` para qualquer usuário (incluindo autenticados), exceto se explicitamente decidido de outra forma.
- **FR-005**: Rules MUST ser compatíveis com o fluxo de autenticação atual (e-mail/senha do Firebase Auth) e com a conta demo `demo@myunihub.app`, sem regras condicionais especiais para a demo.
- **FR-006**: Em operações de escrita que atualizam `books` ou `weekPlans` no documento `users/{uid}`, a regra MUST aceitar itens que **não** têm o campo `uid` (legado) e MUST **rejeitar** itens em que `uid` está **presente** mas é **diferente** de `request.auth.uid`. Na primeira escrita legítima após o deploy das Rules, o cliente é responsável por preencher `uid` em todos os itens existentes (migração silenciosa one-shot), eliminando gradualmente o caso "sem uid". **Escopo limitado a `books` e `weekPlans`** — `subjects` e `deadlines` continuam sem essa camada extra, pois o isolamento por path já é a defesa primária. Ver Clarifications → Q1 e → Q4.
- **FR-007 (removida)**: A promessa original de "limitar o tamanho total do documento a 1 MB via Rules" foi removida — o Firestore já aplica nativamente o limite de 1.048.576 bytes por documento, e a linguagem de Rules não expõe operador para tamanho total do documento. Ver Clarifications → Q3.
- **FR-007b (validação granular de campos)**: A regra MUST validar comprimentos máximos razoáveis de strings e tamanhos máximos de arrays, como defesa em profundidade para evitar campos absurdos. Limites sugeridos (a confirmar/refinar no plano):
  - `profile.name` ≤ 80 chars; `profile.course` ≤ 120; `profile.institution` ≤ 120
  - `subjects` length ≤ 30 itens; `deadlines` ≤ 200; `weekPlans` ≤ 200; `books` ≤ 200
  - Itens de `weekPlans.notes` ≤ 2.000 chars; `book.title` ≤ 200; `book.author` ≤ 200
  - `checklist.items` ≤ 50
  - Escritas que violem esses limites MUST ser rejeitadas.
- **FR-008**: Rules MUST ser deployadas via `firebase deploy --only firestore:rules` (ou via console), e o deploy MUST ser reversível (rollback para a versão anterior em caso de regressão).
- **FR-009**: Rules MUST incluir regras padrão catch-all que negam qualquer path não explicitamente permitido (fail-closed).
- **FR-010**: Rules MUST incluir comentários explicando cada bloco, o caminho protegido e o motivo, para facilitar auditoria futura.

### Key Entities

- **Usuário autenticado**: `request.auth.uid` é o identificador canônico usado como chave do path. É o "dono" do documento `users/{uid}`.
- **Documento `users/{uid}`**: a única entidade regida pelas Rules; contém perfil, subjects, deadlines, weekPlans, books, checklist, studyHistory, timer, onboardingComplete.
- **Conta demo**: um `uid` comum criado a partir de `demo@myunihub.app`; não tem papel especial, é apenas um usuário regular sob as mesmas regras.

### Out of Scope (não cobertos por esta feature)

Os seguintes itens **não** são responsabilidade desta feature e devem ser tratados em features/melhorias separadas:

- **Rate limiting server-side**: proteção contra abuso de cota via throttling por IP/uid — não é coberto pelas Firestore Rules; requer App Check ou Cloud Functions.
- **Observabilidade / logging estruturado de tentativas negadas**: Rules não enviam logs. Para auditoria, usar Cloud Logging via Cloud Function trigger, ou Firebase App Check telemetry.
- **Anti-scraping na REST API**: Firestore Rules são aplicadas a requisições autenticadas; a apiKey pública permite requests REST anônimos, mas a FR-002/FR-004 já negam todos. Scraper sofisticado (com auth obtido de outro jeito) está fora do escopo de Rules.
- **Retenção/LGPD de contas inativas**: regras de cleanup, export, delete-on-request — feature separada.
- **Multi-region, replicação cross-region, Cloud Armor**: topologia de infra — feature separada.

**Candidato natural a próxima feature**: **Firebase App Check** — complementa esta feature adicionando attestation de origem (reCAPTCHA/replay-protection) que protege a apiKey pública contra abuso por clientes modificados/bots. Ver Clarifications → Q5.

## Success Criteria

- **SC-001**: Após o deploy das Rules, **0%** das tentativas de leitura/escrita em `users/{outro_uid}` por usuário autenticado são bem-sucedidas (verificado por teste E2E ou regra de teste).
- **SC-002**: Após o deploy, **0%** das tentativas de leitura/escrita anônimas em qualquer path do Firestore são bem-sucedidas.
- **SC-003**: O fluxo demo (login → adicionar prazo → logout → login → ver prazo persistido) **continua funcionando 100%** do que funcionava antes, sem regressão funcional.
- **SC-004**: O fluxo de usuário comum (registro → setup → adicionar matéria/prazo/livro → logout → login → ver tudo persistido) **continua funcionando 100%** sem regressão.
- **SC-005**: O deploy das Rules é concluído em **uma única operação** reversível, com o conjunto de regras versionado no console do Firebase.
- **SC-006**: O conjunto de Rules é legível: 100% dos caminhos cobertos têm comentário explicativo sobre o que protegem e por quê.

## Assumptions

- O projeto Firebase `myunihub-4bb1f` está ativo e o time tem permissão de `Firebase Admin` (ou `Editor` em IAM) para deployar Rules.
- O app cliente continua a fazer **apenas** operações em `users/{uid}` (verificado na investigação — não há outras coleções sendo acessadas pelo app atual).
- A autenticação é exclusivamente por Firebase Auth (e-mail/senha). Provedores federados (Google, Apple) não são usados pelo cliente atual.
- A apiKey do Firebase permanece exposta no `index.html` (padrão de apps client-side do Firebase; a segurança depende das Rules, não do segredo da chave).
- A versão LTS/estável mais recente do formato de Rules do Firestore é aceitável; não há dependência de uma versão legada específica.
- O Firestore está no plano **Spark** (gratuito) ou **Blaze**; as Rules são compatíveis com ambos (a cobrança não muda com Rules).
- O time entende que **`request.auth.token.email == 'demo@myunihub.app'`** *não* será usado como condição de regra — a demo será tratada como usuário comum.
- O `uid` da demo é o `uid` real gerado pelo Firebase Auth no momento da criação da conta; a UI referencia apenas o e-mail, não o uid.

# Tasks: Recuperação de Senha

**Input**: Design documents from `specs/003-password-recovery/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`

**Tests**: Não foram solicitados testes automatizados nesta especificação. A validação será feita pelos cenários manuais de `quickstart.md`.

**Organization**: As tarefas estão organizadas por história de usuário para permitir implementação e validação independente de cada incremento.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Pode ser executada em paralelo com outras tarefas que não compartilham arquivo ou dependência.
- **[Story]**: Identifica a história de usuário (`US1`, `US2` ou `US3`).

---

## Phase 1: Setup (Infraestrutura Compartilhada)

**Purpose**: Confirmar a estrutura existente e preparar o único arquivo de implementação.

- [ ] T001 [P] Confirmar que `index.html` contém o módulo Firebase Auth existente e que não há build step em `index.html`.
- [ ] T002 [P] Confirmar que os artefatos de design estão disponíveis em `specs/003-password-recovery/research.md`, `specs/003-password-recovery/data-model.md`, `specs/003-password-recovery/quickstart.md` e `specs/003-password-recovery/contracts/`.
- [ ] T003 [P] Confirmar que a tela de login atual está associada a `authScreen` e que o modal será implementado em `index.html`.

---

## Phase 2: Foundational (Pré-requisitos Bloqueantes)

**Purpose**: Criar a base comum necessária antes de qualquer história de usuário.

- [ ] T004 [US0] Adicionar `sendPasswordResetEmail` à importação do Firebase Auth existente em `index.html`.
- [ ] T005 [US0] Criar as constantes e funções auxiliares de estado do fluxo de recuperação em `index.html`, incluindo referência aos elementos do modal, foco anterior, status e controle de cooldown.
- [ ] T006 [US0] Implementar `isPasswordResetCooldownActive()` em `index.html`, usando `localStorage` para persistir o timestamp de cooldown por 45 segundos.

**Checkpoint**: A integração com Firebase Auth e o controle de cooldown estão disponíveis para todas as histórias.

---

## Phase 3: User Story 1 - Solicitar recuperação de senha a partir do login (Priority: P1) 🎯 MVP

**Goal**: Permitir que o usuário inicie a recuperação diretamente da tela de login e receba feedback visual do envio.

**Independent Test**: Na tela de login, clicar em "Esqueci minha senha", informar um e-mail válido, confirmar e observar o feedback de sucesso sem sair do contexto de login.

**Implementation for User Story 1**

- [ ] T007 [P] [US1] Adicionar o link "Esqueci minha senha" próximo ao campo de senha dentro de `authScreen` em `index.html`.
- [ ] T008 [P] [US1] Criar o modal de recuperação de senha em `index.html` com título, subtítulo, campo de e-mail, botão de fechar, botão de cancelar e botão de envio.
- [ ] T009 [P] [US1] Aplicar ao modal de `index.html` o padrão visual dos modais existentes: glassmorphism, paleta ocean/violet/warm, backdrop, tema claro/escuro e layout responsivo.
- [ ] T010 [US1] Implementar `openPasswordResetModal()` em `index.html`, salvando o elemento com foco, abrindo o modal, limpando o estado anterior e focando o campo de e-mail.
- [ ] T011 [US1] Implementar `closePasswordResetModal()` em `index.html`, fechando o modal, limpando mensagens e retornando o foco ao link "Esqueci minha senha".
- [ ] T012 [US1] Implementar `showPasswordResetFeedback()` em `index.html` para exibir mensagens de sucesso ou erro no modal e no `loginErr` quando apropriado.
- [ ] T013 [US1] Implementar `requestPasswordReset()` em `index.html` para validar presença e formato do e-mail, respeitar cooldown, chamar `sendPasswordResetEmail(auth, email)` e atualizar o feedback de sucesso ou erro.

**Checkpoint**: A história US1 está funcional e testável independentemente.

---

## Phase 4: User Story 2 - Validar o e-mail antes do envio (Priority: P2)

**Goal**: Impedir envios desnecessários e orientar o usuário quando o e-mail estiver vazio ou com formato inválido.

**Independent Test**: Com o modal aberto, tentar enviar um valor vazio e depois um valor como `marcio@`; em ambos os casos, observar mensagem específica sem chamada ao Firebase.

**Implementation for User Story 2**

- [ ] T014 [P] [US2] Adicionar validação de e-mail obrigatório em `requestPasswordReset()` em `index.html`, exibindo feedback antes de qualquer chamada externa.
- [ ] T015 [P] [US2] Adicionar validação de formato de e-mail em `requestPasswordReset()` em `index.html`, normalizando espaços e rejeitando formatos inválidos antes do Firebase.
- [ ] T016 [US2] Adicionar atributos de acessibilidade no campo e nos botões do modal em `index.html`, incluindo `label`, `aria-label`, `aria-describedby`, `aria-live` e estado de desabilitado durante o envio.
- [ ] T017 [US2] Adicionar tratamento de `keydown` para fechar o modal com `Escape` e manter o foco gerenciado em `index.html`.

**Checkpoint**: A história US2 está funcional e testável independentemente.

---

## Phase 5: User Story 3 - Tratar indisponibilidade e preservar o contexto (Priority: P3)

**Goal**: Responder a falhas do Firebase de forma compreensível, manter o formulário utilizável e aplicar o cooldown após tentativas enviadas.

**Independent Test**: Com o serviço indisponível ou com um e-mail válido não cadastrado, confirmar a solicitação, observar mensagem amigável, fechar/reabrir o modal e verificar que o fluxo permanece utilizável.

**Implementation for User Story 3**

- [ ] T018 [P] [US3] Implementar mapeamento dos erros `auth/invalid-email`, `auth/too-many-requests`, `auth/network-request-failed` e fallback genérico em `requestPasswordReset()` em `index.html`.
- [ ] T019 [P] [US3] Implementar indicador de carregamento e desabilitar o botão de envio enquanto a Promise de `sendPasswordResetEmail` estiver pendente em `index.html`.
- [ ] T020 [US3] Registrar o timestamp de cooldown após a solicitação alcançar o Firebase e atualizar o estado visual do botão em `index.html`.
- [ ] T021 [US3] Impedir submissões concorrentes enquanto uma solicitação estiver em andamento em `index.html`, sem duplicar chamadas ao Firebase.
- [ ] T022 [US3] Garantir que sucesso, erro, cancelamento e fechamento do modal preservem a tela de login e os elementos de autenticação existentes em `index.html`.

**Checkpoint**: A história US3 está funcional e testável independentemente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Validar consistência visual, responsividade, acessibilidade e ausência de efeitos colaterais.

- [ ] T023 [P] Validar o modal em tema claro e escuro em `index.html`, conferindo contraste, glassmorphism e paleta ocean/violet/warm.
- [ ] T024 [P] Validar o modal em largura reduzida e dispositivo móvel em `index.html`, sem rolagem horizontal ou perda dos controles principais.
- [ ] T025 [P] Executar os cenários de `specs/003-password-recovery/quickstart.md` no navegador e registrar os resultados.
- [ ] T026 [US0] Revisar `index.html` para confirmar que não há novas coleções do Firestore, Cloud Functions, backend ou auditoria dedicada à recuperação de senha.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sem dependências; pode começar imediatamente.
- **Foundational (Phase 2)**: Depende apenas da confirmação da estrutura; bloqueia as histórias de usuário.
- **User Story 1 (P1)**: Depende da Phase 2; é o MVP da funcionalidade.
- **User Story 2 (P2)**: Depende da Phase 2; pode ser executada após ou em paralelo com US1, desde que T004–T006 estejam concluídas.
- **User Story 3 (P3)**: Depende da Phase 2; pode ser executada após ou em paralelo com US1/US2, desde que T004–T006 estejam concluídas.
- **Polish (Phase 6)**: Depende da conclusão das histórias que serão entregues.

### User Story Dependencies

- **US1 (P1)**: T004 → T005 → T006 → T007–T013.
- **US2 (P2)**: T004 → T005 → T006 → T014–T017.
- **US3 (P3)**: T004 → T005 → T006 → T018–T022.

### Within Each User Story

- Elementos de UI antes da interação.
- Abertura/fechamento e foco antes da chamada ao Firebase.
- Validação antes do envio.
- Feedback e cooldown após a chamada.
- Validar cada história independentemente antes de avançar.

### Parallel Opportunities

- T001, T002 e T003 podem rodar em paralelo.
- T007, T008 e T009 podem rodar em paralelo após T004–T006.
- T014 e T015 podem rodar em paralelo após T004–T006.
- T018 e T019 podem rodar em paralelo após T004–T006.
- T023, T024 e T026 podem rodar em paralelo após a implementação principal.

---

## Parallel Execution Examples

### US1

```text
T007: adicionar link em authScreen
T008: criar estrutura do modal
T009: aplicar estilos do modal
```

### US2

```text
T014: validar e-mail obrigatório
T015: validar formato de e-mail
```

### US3

```text
T018: mapear erros do Firebase
T019: implementar loading e desabilitação
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Concluir T001–T006.
2. Concluir T007–T013.
3. Validar o fluxo completo de abertura, envio, sucesso e fechamento.
4. Entregar o MVP com validação básica e feedback visual.

### Incremental Delivery

1. MVP: US1.
2. Adicionar US2: validação clara de e-mail e acessibilidade.
3. Adicionar US3: tratamento de falhas, loading e cooldown.
4. Executar Polish e os cenários de `quickstart.md`.

### Parallel Team Strategy

Com mais de uma pessoa, US1, US2 e US3 podem ser implementadas em paralelo após T004–T006, desde que cada pessoa trabalhe em trechos distintos de `index.html` e haja integração final para evitar conflitos no mesmo arquivo.

---

## Notes

- Todas as tarefas modificam apenas `index.html`, conforme a constituição do projeto e o plano aprovado.
- Não criar novos arquivos de código-fonte, endpoints, Cloud Functions ou coleções do Firestore.
- Não registrar auditoria dedicada para solicitações de recuperação de senha.
- Manter a mensagem de sucesso genérica para evitar confirmação de existência de conta.

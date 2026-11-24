# Especificação da Funcionalidade: Recuperação de Senha

**Feature Branch**: `003-password-recovery`

**Created**: 2026-09-16

**Status**: Draft

**Input**: Adicionar funcionalidade de recuperação de senha ao MyUniHub usando Firebase Auth (sendPasswordResetEmail). Deve incluir: (1) um link "Esqueci minha senha" na tela de login, próximo ao campo de senha, (2) um pequeno formulário/modal pedindo o e-mail do usuário, (3) chamada à função sendPasswordResetEmail do Firebase, (4) feedback visual de sucesso ("e-mail enviado, confira sua caixa de entrada") ou erro (e-mail não encontrado, formato inválido), (5) ajuste no layout da tela de login para acomodar esse novo link de forma consistente com o design atual (glassmorphism, paleta ocean/violet/warm).

## Clarifications

### Session 2026-09-16

- Q: Qual o comportamento desejado para rate limiting em solicitações de recuperação de senha? → A: Sem limitação de taxa geral, com cooldown de 45 segundos entre solicitações.
- Q: Qual o nível de acessibilidade esperado para o modal de recuperação? → A: Gerenciar foco ao abrir/fechar + labels e atributos ARIA.
- Q: Qual o nível de registro de eventos desejado para solicitações de recuperação de senha? → A: Não registrar eventos de recuperação de senha.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Solicitar recuperação de senha a partir do login (Priority: P1)

Como usuário que não lembra a própria senha, quero conseguir iniciar a recuperação diretamente da tela de login para recuperar o acesso à minha conta sem precisar entrar em contato com o suporte.

**Why this priority**: É o fluxo principal da funcionalidade. Sem ele, o usuário continua bloqueado no login e não consegue resolver o problema sozinho.

**Independent Test**: Na tela de login, o usuário clica em "Esqueci minha senha", informa o e-mail cadastrado, confirma o envio e recebe uma mensagem visual de sucesso ou erro.

**Acceptance Scenarios**:

1. **Given** que o usuário está na tela de login, **When** clica no link "Esqueci minha senha", **Then** um pequeno formulário ou modal é aberto pedindo o e-mail da conta.
2. **Given** que o formulário de recuperação está aberto, **When** o usuário informa um e-mail válido e confirma a solicitação, **Then** o sistema solicita o envio de um e-mail de recuperação para esse endereço.
3. **Given** que o e-mail foi aceito, **When** a solicitação é concluída com sucesso, **Then** o usuário vê uma mensagem informando que o e-mail foi enviado e que deve verificar a caixa de entrada.

---

### User Story 2 - Validar o e-mail antes do envio (Priority: P2)

Como usuário, quero receber uma indicação clara quando o e-mail informado estiver em formato inválido para corrigir o dado antes de tentar novamente.

**Why this priority**: Evita tentativas desnecessárias, reduz mensagens confusas e melhora a experiência de quem digitou o endereço incorretamente.

**Independent Test**: O usuário abre o formulário, informa um valor inválido como `marcio@` ou `sem-arroba`, confirma e observa uma mensagem de erro específica e visível.

**Acceptance Scenarios**:

1. **Given** que o campo de e-mail está vazio, **When** o usuário tenta confirmar a recuperação, **Then** o sistema exibe uma mensagem indicando que o e-mail é obrigatório.
2. **Given** que o campo contém um formato inválido, **When** o usuário tenta confirmar, **Then** o sistema não envia a solicitação e mostra uma mensagem de formato inválido.
3. **Given** que o campo contém um formato válido, **When** o usuário confirma, **Then** o sistema permite que a solicitação seja processada.

---

### User Story 3 - Tratar e-mail não encontrado ou indisponibilidade do serviço (Priority: P3)

Como usuário, quero receber uma mensagem de erro compreensível quando a recuperação não puder ser concluída, sem perder o contexto da tela de login.

**Why this priority**: Nem toda solicitação terá sucesso. O sistema precisa evitar que o usuário fique sem resposta ou confuso quando o e-mail não existir ou o serviço de autenticação não responder.

**Independent Test**: O usuário informa um e-mail válido que não corresponde a uma conta existente e confirma a solicitação; o sistema exibe uma mensagem de erro clara e mantém o formulário utilizável.

**Acceptance Scenarios**:

1. **Given** que o usuário informa um e-mail válido que não corresponde a uma conta existente, **When** confirma a solicitação, **Then** o sistema exibe uma mensagem de erro informando que não foi possível encontrar a conta ou que não foi possível enviar o e-mail.
2. **Given** que o serviço de autenticação está indisponível, **When** o usuário tenta solicitar a recuperação, **Then** o sistema exibe uma mensagem de falha e permite nova tentativa.
3. **Given** que uma tentativa falhou, **When** o usuário corrige ou altera o e-mail, **Then** o sistema permite enviar uma nova solicitação sem recarregar a página.

---

### Edge Cases

- Campo de e-mail vazio.
- E-mail com espaços antes ou depois do endereço.
- Cooldown de 45 segundos entre solicitações de recuperação concluídas; o botão permanece desabilitado durante esse período.
- Não haverá registro de auditoria dedicado para solicitações de recuperação de senha nesta versão.
- E-mail em formato inválido, como ausência de `@`, domínio incompleto ou caracteres inválidos.
- E-mail válido, mas não cadastrado no sistema.
- Falha temporária do serviço de autenticação.
- Usuário fecha o modal antes de concluir a solicitação.
- Usuário tenta enviar a recuperação enquanto a solicitação anterior ainda está em andamento.
- Usuário acessa a tela de login em dispositivo móvel ou em largura reduzida.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema DEVE exibir um link ou botão com o texto "Esqueci minha senha" na tela de login, posicionado próximo ao campo de senha.
- **FR-002**: O sistema DEVE abrir um formulário ou modal de recuperação quando o usuário acionar "Esqueci minha senha".
- **FR-003**: O formulário de recuperação DEVE conter um campo para informar o e-mail da conta.
- **FR-004**: O sistema DEVE validar se o e-mail informado está preenchido antes de enviar a solicitação.
- **FR-005**: O sistema DEVE validar o formato do e-mail antes de enviar a solicitação.
- **FR-006**: O sistema DEVE solicitar ao serviço de autenticação o envio de um e-mail de recuperação quando o e-mail informado for válido.
- **FR-007**: O sistema DEVE exibir uma mensagem visual de sucesso informando que o e-mail foi enviado e que o usuário deve verificar a caixa de entrada.
- **FR-008**: O sistema DEVE exibir uma mensagem visual de erro quando o e-mail não for encontrado, quando o formato for inválido ou quando o envio não puder ser concluído.
- **FR-009**: O sistema DEVE manter o usuário no contexto da tela de login após o sucesso ou erro da solicitação.
- **FR-010**: O sistema DEVE oferecer uma ação clara para fechar ou cancelar o formulário de recuperação.
- **FR-011**: O layout da recuperação DEVE seguir o padrão visual da tela de login, incluindo glassmorphism e a paleta ocean/violet/warm.
- **FR-012**: O formulário de recuperação DEVE permanecer legível e utilizável em telas de largura reduzida.
- **FR-013**: O sistema DEVE impedir que uma nova solicitação seja enviada antes de decorridos 45 segundos da solicitação anterior.
- **FR-014**: O sistema DEVE exibir um indicador de carregamento e desabilitar o botão de envio enquanto a solicitação estiver em andamento.
- **FR-015**: O modal DEVE gerenciar o foco ao abrir/fechar e usar labels/ARIA nos campos e botões.
- **FR-016**: O sistema NÃO DEVE registrar auditoria dedicada para solicitações de recuperação de senha nesta versão.

### Key Entities

- **Usuário**: pessoa que possui uma conta no MyUniHub e precisa recuperar o acesso.
- **E-mail da conta**: identificador informado pelo usuário para localizar a conta e enviar a instrução de recuperação.
- **Solicitação de recuperação de senha**: ação iniciada pelo usuário para pedir o envio de instruções de redefinição de senha.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Usuários conseguem iniciar uma solicitação de recuperação de senha em até 3 interações a partir da tela de login.
- **SC-002**: O tempo entre confirmar um e-mail válido e receber feedback visual de sucesso ou erro é de no máximo 3 segundos em condições normais de conexão.
- **SC-003**: 100% das tentativas com e-mail vazio ou formato inválido recebem feedback antes de qualquer envio.
- **SC-004**: 100% das tentativas bem-sucedidas exibem uma mensagem informando que o usuário deve verificar a caixa de entrada.
- **SC-005**: Usuários conseguem concluir o fluxo em telas móveis sem precisar rolar horizontalmente ou perder acesso aos controles principais.
- **SC-006**: Em testes de usabilidade, pelo menos 90% dos usuários conseguem identificar e concluir a recuperação de senha sem ajuda externa.

## Assumptions

- O MyUniHub já possui uma tela de login funcional e uma integração com serviço de autenticação.
- O serviço de autenticação é capaz de enviar e-mails de recuperação para endereços cadastrados.
- O usuário tem acesso ao e-mail cadastrado na conta.
- Não haverá registro de auditoria dedicado para solicitações de recuperação de senha nesta versão.
- A funcionalidade cobre apenas a solicitação do envio do e-mail de recuperação; a criação da nova senha na página recebida por e-mail fica fora do escopo desta etapa.
- Não será adicionada uma etapa extra de verificação de identidade nesta versão, além do envio para o e-mail cadastrado.
- A funcionalidade deve funcionar em desktop e mobile, mantendo o padrão visual existente.

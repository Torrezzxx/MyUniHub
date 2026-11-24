# Implementation Plan: Recuperação de Senha

**Branch**: `003-password-recovery` | **Date**: 2026-09-16 | **Spec**: `specs/003-password-recovery/spec.md`

**Input**: Implementação client-side pura da recuperação de senha no MyUniHub, usando `sendPasswordResetEmail` do Firebase Auth, com modal de recuperação, validação de e-mail, feedback de sucesso/erro, cooldown de 45 segundos no cliente e layout consistente com o design atual.

**Note**: Este plano cobre apenas a implementação da funcionalidade de recuperação de senha no `index.html` existente. Não há build step, backend próprio, Cloud Functions ou novas coleções do Firestore.

## Summary

Adicionar à tela de login do MyUniHub um link "Esqueci minha senha" que abre um modal de recuperação com campo de e-mail, validação de formato, chamada a `sendPasswordResetEmail` do Firebase Auth, feedback visual de sucesso ou erro, e cooldown de 45 segundos entre tentativas. O modal deve seguir o padrão visual dos modais existentes (glassmorphism, paleta ocean/violet/warm), com foco acessível e sem persistência adicional no Firestore.

## Technical Context

**Language/Version**: HTML5, CSS (Tailwind via CDN), JavaScript moderno (ES modules no navegador)

**Primary Dependencies**: Firebase Auth (SDK v11.0.1 já importado no `index.html`), Tailwind CSS v3.4.0 via CDN

**Storage**: N/A para esta feature (não há persistência nova no Firestore)

**Testing**: Validação manual no navegador; verificação de fluxo de login, modal, validações, sucesso, erro e cooldown

**Target Platform**: Navegadores modernos com suporte a ES modules, `fetch`, `classList`, `backdrop-filter` e Firebase Auth

**Project Type**: Web application client-side puro

**Performance Goals**: Feedback visual de sucesso/erro em até 3 segundos em condições normais de conexão; modal renderizado sem travar a tela de login

**Constraints**:
- Implementação integralmente dentro do `index.html` existente
- Sem build step
- Sem backend próprio
- Sem Cloud Functions
- Sem novas coleções do Firestore
- Cooldown de 45 segundos implementado no lado do cliente
- Reutilizar o padrão visual dos modais existentes
- Manter compatibilidade com tema claro/escuro
- Não expor dados sensíveis em mensagens de erro

**Scale/Scope**: 1 tela de login, 1 modal de recuperação, 1 fluxo de envio de e-mail, 1 cooldown local

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- [x] A funcionalidade é uma melhoria incremental no `index.html` existente, sem nova arquitetura ou projeto
- [x] O escopo é pequeno e delimitado
- [x] Não há nova infraestrutura, backend ou armazenamento persistente
- [x] A funcionalidade reutiliza a integração de autenticação existente
- [x] A implementação mantém o padrão visual e de acessibilidade do app
- [x] Não há necessidade de migração de dados
- [x] A feature é testável de forma independente na tela de login

## Project Structure

### Documentation (this feature)

```text
specs/003-password-recovery/
├── plan.md              # Este arquivo
├── research.md          # Fase 0 — decisões e melhores práticas
├── data-model.md        # Fase 1 — entidades e estados do fluxo
├── quickstart.md        # Fase 1 — guia de validação
├── contracts/           # Fase 1 — contratos de interface
└── tasks.md             # Fase 2 — /speckit-tasks
```

### Source Code (repository root)

```text
index.html
```

**Structure Decision**: A feature será implementada integralmente em `index.html`, seguindo o padrão atual do projeto client-side puro. Não haverá criação de novos arquivos de código-fonte.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

N/A — nenhuma violação de constituição identificada.

## Phase 0: Research

### Unknowns / decisions to resolve

1. **Import de `sendPasswordResetEmail`**: confirmar o ponto correto de importação no módulo Firebase Auth existente.
2. **Comportamento de erro do Firebase Auth**: mapear códigos de erro relevantes para mensagens amigáveis em português.
3. **Cooldown de 45 segundos**: definir mecanismo de controle no lado do cliente (timestamp em `localStorage` ou variável em memória).
4. **Segurança de mensagem de erro**: evitar confirmação de existência de e-mail para usuários não cadastrados.
5. **Acessibilidade do modal**: garantir foco inicial, retorno de foco e fechamento por `Escape`.

### Research outputs expected

- `research.md` com decisões e alternativas consideradas
- `data-model.md` com entidades e estados do fluxo
- `quickstart.md` com cenários de validação end-to-end
- `contracts/` com contrato de UI e contrato de integração com Firebase Auth

## Phase 1: Design & Contracts

### Data model

- Entidade principal: solicitação de recuperação de senha (estado efêmero no cliente)
- Estados: fechado, aberto, validando, enviando, sucesso, erro
- Campo de e-mail: string, trim, validação de formato
- Cooldown: timestamp de última tentativa bem-sucedida ou enviada, 45 segundos

### Interface contracts

- `openPasswordResetModal()`: abre o modal, limpa estado anterior, foca o campo de e-mail
- `closePasswordResetModal()`: fecha o modal e limpa mensagens/estado
- `requestPasswordReset()`: valida e-mail, respeita cooldown, chama Firebase Auth, atualiza UI
- `showPasswordResetFeedback(type, message)`: renderiza feedback visual de sucesso ou erro
- `isPasswordResetCooldownActive()`: verifica se o intervalo de 45 segundos ainda está ativo

### Validation scenarios

- Link "Esqueci minha senha" visível na tela de login
- Modal abre com foco no campo de e-mail
- E-mail vazio gera erro sem chamada ao Firebase
- E-mail inválido gera erro sem chamada ao Firebase
- E-mail válido dispara envio e bloqueia novas tentativas por 45 segundos
- Sucesso exibe mensagem "e-mail enviado, confira sua caixa de entrada"
- Erro exibe mensagem amigável sem expor dados sensíveis
- Modal fecha por botão, backdrop e `Escape`
- Tema claro/escuro preserva o visual do modal
- Layout responsivo em largura reduzida

## Phase 2: Implementation Tasks

> A criação de `tasks.md` será feita pelo `/speckit-tasks`.

Tarefas previstas:
1. Importar `sendPasswordResetEmail` no módulo Firebase Auth
2. Adicionar link "Esqueci minha senha" na tela de login
3. Criar modal de recuperação com padrão visual dos modais existentes
4. Implementar abertura/fechamento, foco e acessibilidade
5. Implementar validação de e-mail e mensagens de erro
6. Implementar chamada a `sendPasswordResetEmail`
7. Implementar feedback de sucesso/erro
8. Implementar cooldown de 45 segundos no cliente
9. Ajustar layout responsivo e tema claro/escuro
10. Validar fluxo completo no navegador

## Re-check Constitution

- [x] Nenhuma violação introduzida
- [x] Escopo permanece dentro de `index.html`
- [x] Nenhuma dependência nova de backend ou infraestrutura
- [x] Funcionalidade continua testável de forma independente

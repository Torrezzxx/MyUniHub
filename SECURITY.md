# Segurança — MyUniHub

Este arquivo descreve a postura de segurança do MyUniHub e como reportar vulnerabilidades.

## 📋 Visão Geral

O MyUniHub é um aplicativo estático (SPA) que utiliza Firebase (Auth + Firestore) para autenticação e persistência de dados. A segurança é implementada em três camadas:

1. **Restrições de Chave API no Google Cloud Console** — limita onde a key pode ser usada
2. **Firestore Security Rules** — controla acesso a dados por usuário
3. **Subresource Integrity (SRI)** — protege contra comprometimento de CDN

## 🔑 A Chave API do Firebase no Código

> **É esperado que a key esteja exposta no código-fonte.**

Em aplicações Firebase client-side, a `apiKey` é um identificador público (não um segredo). Ela é necessária para inicializar o SDK do Firebase no navegador. A segurança **não** depende de mantê-la oculta, mas de:

- **Restrições de HTTP referrer** — apenas domínios autorizados podem usar a key
- **Restrições de API** — apenas os serviços Firebase usados pelo app estão habilitados
- **Firestore Security Rules** — mesmo com a key, dados sensíveis são protegidos por regras de acesso

### O que NÃO é considerado vulnerabilidade

- A presença da `apiKey` em `index.html` (por design)
- A exposição de um Firebase project ID (é público)
- Scripts externos de CDN conhecido (com SRI verificado)

## 🛡️ Camadas de Defesa

### 1. Restrições de Chave API (GCP)

A key atual tem aplicadas:

| Restrição | Valor |
|-----------|-------|
| HTTP Referrer | `*.web.app`, `*.firebaseapp.com`, `localhost` |
| APIs permitidas | Firebase Identity Toolkit, Firestore, Cloud Storage |
| APIs desabilitadas | Maps, Translate, Vision, etc. |

### 2. Firestore Security Rules

O arquivo `firestore.rules` implementa:

- **Isolamento por `uid`**: cada usuário só acessa o próprio documento em `users/{uid}`
- **Bloqueio anônimo**: requisições sem autenticação são negadas
- **Catch-all fail-closed**: qualquer path não explicitamente permitido é negado

### 3. Subresource Integrity (SRI)

Todos os scripts externos (Chart.js + 3 módulos Firebase) carregam com `integrity="sha384-..."` e `crossorigin="anonymous"`. O Tailwind CSS é uma exceção documentada: seu CDN não envia `Access-Control-Allow-Origin`, então SRI não é aplicável.

## 🚨 Reportando Vulnerabilidades

Se você encontrar uma vulnerabilidade de segurança:

1. Abra uma **Issue** no repositório com o título `security: <descrição>`
2. Inclua:
   - Descrição do problema
   - Passos para reproduzir (se aplicável)
   - Impacto estimado
   - Sugestão de mitigação (opcional)
3. Para vulnerabilidades **críticas** (ex.: exposição de dados de usuário), contacte o maintainer diretamente pelo email no perfil do GitHub.

> **Não** reporte como vulnerabilidade:
> - Presença da apiKey no código (esperada por design)
> - Falta de App Check (roadmap futuro)
> - Falta de criptografia client-side (Firestore criptografado em trânsito e repouso)

## 📜 Histórico de Remediação

Esta feature foi motivada pelo relatório GitGuardian de 2025-08-25 (scan `cmt8unj6201oz10d7yz1dv57o`), que identificou:

| # | Severidade | Finding | Ação |
|---|-----------|---------|------|
| F1 | 🔴 HIGH | API key exposta em `index.html` | Restrições GCP aplicadas; rotação condicional |
| F2 | 🟡 MEDIUM | Script sem SRI (Tag 1) | SRI adicionado |
| F3 | 🟡 MEDIUM | Script sem SRI (Tag 2) | SRI adicionado |
| F4 | 🟡 MEDIUM | Script sem SRI (Tag 3) | SRI adicionado |

Veja `specs/002-gitguardian-remediation/spec.md` para detalhes completos.
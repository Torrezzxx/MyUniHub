# Implementation Plan: GitGuardian Remediation — SRI em scripts externos

**Branch**: `002-gitguardian-remediation`  
**Date**: 2026-09-10  
**Spec**: specs/002-gitguardian-remediation/spec.md

---

## Summary

Esta feature implementa **Subresource Integrity (SRI)** nos 5 scripts externos carregados pelo `index.html`:
1. **Tailwind CSS** — CDN sem versão (precisa fixar em `3.4.0`)
2. **Chart.js** — já fixo em `4.4.0` via jsDelivr
3. **Firebase App** — ESM import, fixo em `11.0.1`
4. **Firebase Auth** — ESM import, fixo em `11.0.1`
5. **Firebase Firestore** — ESM import, fixo em `11.0.1`

**Abordagem técnica**: 
- Fixar versão do Tailwind (FR-006)
- Gerar hashes SHA-384 localmente via `openssl` ou srihash.org (FR-005)
- Adicionar `integrity="sha384-..."` + `crossorigin="anonymous"` a cada tag (FR-004)
- Validar no navegador que scripts carregam e funcionalidades funcionam

---

## Technical Context

| Campo | Valor |
|-------|-------|
| **Language/Version** | HTML5 + ES Modules (ESM) |
| **Primary Dependencies** | Tailwind CSS 3.4.0 (CDN), Chart.js 4.4.0 (jsDelivr), Firebase JS SDK 11.0.1 (gstatic) |
| **Storage** | N/A (client-side static HTML) |
| **Testing** | Manual via navegador (DevTools Network/Console); validação de hash via SRI |
| **Target Platform** | Navegadores modernos (Chrome, Edge, Firefox, Safari) — suporte universal a SRI |
| **Project Type** | Single-page application (SPA) estática / Web application |
| **Performance Goals** | SRI adiciona overhead ~0ms (hash verificado pelo navegador nativamente) |
| **Constraints** | - Hashes **devem** ser gerados do conteúdo exato do CDN (não adivinhados)<br>- Tailwind CDN **precisa** versão fixa antes do SRI<br>- Imports ESM Firebase usam `type="module"` |
| **Scale/Scope** | 5 scripts no único arquivo `index.html` |

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Status | Justificativa |
|-----------|--------|---------------|
| Library-First | ✅ Pass | SRI é atributo HTML nativo; sem biblioteca extra |
| CLI Interface | N/A | Não há CLI; feature é alteração em HTML |
| Test-First | ✅ Pass | Quickstart define validação antes da implementação |
| Integration Testing | ✅ Pass | Validação end-to-end no navegador |
| Observability | ✅ Pass | Erros de SRI aparecem no Console nativamente |
| Simplicity | ✅ Pass | Adição de 2 atributos por tag; sem complexidade |

> **Observação**: A constituição do projeto (`.specify/memory/constitution.md`) está em template vazio. Este check usa princípios padrão de segurança web.

---

## Project Structure

### Documentation (esta feature)
```text
specs/002-gitguardian-remediation/
├── plan.md              # Este arquivo
├── research.md          # Phase 0: versões, processo de hash, placeholders
├── data-model.md        # Phase 1: modelo ExternalScript (5 instâncias)
├── quickstart.md        # Phase 1: guia de validação no navegador
├── contracts/
│   └── sri-script.md    # Phase 1: contrato de tag <script> com SRI
└── tasks.md             # Phase 2: (gerado por /speckit-tasks)
```

### Source Code (repository root)
```text
index.html              # Arquivo único a modificar (5 tags <script>)
```

**Structure Decision**: SPA estática em arquivo único. Nenhuma estrutura de build/bundler — SRI aplicado direto no HTML.

---

## Complexity Tracking

> **Nenhuma violação** — a mudança é simples (atributos HTML), sem padrões complexos.

---

## Phase 0: Research (Concluído)

**Output**: `research.md`
- ✅ Versões exatas confirmadas
- ✅ Processo de geração SHA-384 documentado (openssl local / srihash.org)
- ✅ Placeholders definidos para preenchimento manual
- ✅ Decisões e alternativas registradas

---

## Phase 1: Design & Contracts (Concluído)

**Outputs**:
- ✅ `data-model.md` — Entidade `ExternalScript` com 5 instâncias
- ✅ `contracts/sri-script.md` — Contrato de tag `<script>` com SRI
- ✅ `quickstart.md` — Guia de validação passo a passo

---

## Phase 2: Tasks (Próximo — `/speckit-tasks`)

Tasks sugeridas para implementação:

| ID | Task | Descrição |
|----|------|-----------|
| T01 | Fixar Tailwind v3.4.0 | Mudar `cdn.tailwindcss.com` → `cdn.tailwindcss.com/3.4.0` |
| T02 | Gerar hashes locais | Rodar `openssl` para cada um dos 5 arquivos baixados |
| T03 | Aplicar SRI no index.html | Adicionar `integrity` + `crossorigin` nas 5 tags |
| T04 | Validar no navegador | Checklist do quickstart.md |
| T05 | Testes funcionais | Login, gráfico, styling, persistência |
| T06 | Commit | `chore(security): add sri to cdn scripts` |

---

## Referências

- [spec.md](spec.md) — Especificação completa (US2: SRI)
- [research.md](research.md) — Detalhes técnicos de geração de hash
- [data-model.md](data-model.md) — Modelo de dados dos scripts
- [contracts/sri-script.md](contracts/sri-script.md) — Contrato de implementação
- [quickstart.md](quickstart.md) — Validação prática
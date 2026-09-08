# Specification Quality Checklist: GitGuardian Remediation — MyUniHub

**Purpose**: Validate specification completeness and quality before proceeding to planning.
**Created**: 2026-08-31
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details — menciona Firebase CLI, GCP Console, openssl, mas apenas como ferramentas de execução dos FRs, não como decisões de design.
- [x] Focused on user value and business needs — a "user" aqui é o produto MyUniHub e seus usuários; os benefícios são (a) redução de superfície de ataque, (b) conformidade com auditoria, (c) defesa em profundidade.
- [x] Written for non-technical stakeholders — termos técnicos (SRI, REST, JWT) são explicados em contexto; os SCs são verificáveis sem conhecimento profundo.
- [x] All mandatory sections completed — Visão geral, User Scenarios, Requirements, Success Criteria, Assumptions, Out of Scope presentes.

## Requirement Completeness

- [x] No `[NEEDS CLARIFICATION]` markers remain — todas as decisões foram tomadas com base no relatório GitGuardian + investigação do `index.html` + padrões do setor.
- [x] Requirements are testable and unambiguous — cada FR tem critério objetivo (e.g., "tem atributo integrity", "tem restrição de HTTP referrer").
- [x] Success criteria are measurable — SC-001 a SC-007 são quantificáveis ou verificáveis por inspeção/teste.
- [x] Success criteria are technology-agnostic — exceto referências inevitáveis a "Firestore Rules" (que é o objeto da feature `001` citada) e "GCP Console" (já que o GitGuardian é sobre GCP).
- [x] All acceptance scenarios are defined — 4 user stories cobrem: restrições GCP, SRI, deploy de Rules, documentação.
- [x] Edge cases are identified — listados explicitamente (CDN indisponível, versão não-fixada, key antiga em histórico, rollback de Rules).
- [x] Scope is clearly bounded — seção "Out of Scope" explícita.
- [x] Dependencies and assumptions identified — "Assumptions" lista o que foi presumido; dependência de `001-firestore-rules-audit` é destacada.

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria — cada FR tem SC correspondente ou cenário que o exercita.
- [x] User scenarios cover primary flows — US1 (key), US2 (SRI), US3 (Rules), US4 (docs) cobrem todos os findings.
- [x] Feature meets measurable outcomes defined in Success Criteria — cada cenário é mapeado para pelo menos um SC.
- [x] No implementation details leak into specification — não há código-fonte; apenas "o que" e o "por quê".

## Notes

- **Achado da investigação**: o GitGuardian reportou 3 SRI findings, mas a investigação no `index.html` encontrou **5 tags `<script src="https://...">`** sem SRI (incluindo 3 imports ESM do Firebase que o Semgrep pode não ter contado individualmente). A spec cobre as 5 — se ficar constatado depois que o Semgrep só conta as 2 síncronas (Tailwind, Chart.js), o trabalho é apenas menor, não escopo creep.
- **Dependência importante**: US3 desta feature depende do **deploy** (não apenas da criação) das Rules da feature `001-firestore-rules-audit`. Recomenda-se merge de `001` em `main` **antes** de fechar esta feature, ou ambos no mesmo PR.
- **Recomendação de sequência** (próximos comandos):
  1. **`/speckit-clarify`** para validar premissas (especialmente: rotação é condicional? Tailwind fixar versão?)
  2. **`/speckit-plan`** para desenhar a implementação (incluindo como calcular os hashes SRI)
  3. **`/speckit-tasks`** para decompor em tasks atribuíveis
  4. **Implementação**: cada FR em um commit separado (FR-010 já exige isso).
- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`.

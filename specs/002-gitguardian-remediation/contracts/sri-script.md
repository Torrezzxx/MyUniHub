# Contrato: Subresource Integrity (SRI) em scripts externos

**Feature**: 002-gitguardian-remediation  
**Versão**: 1.0  
**Data**: 2026-09-10

---

## Contrato de Tag `<script>` com SRI

### Campos Obrigatórios

| Atributo | Valor | Obrigatório | Descrição |
|----------|-------|-------------|-----------|
| `src` | URL absoluta com versão fixa | Sim | URL do CDN, deve conter versão explícita |
| `integrity` | `sha384-<base64>` | Sim | Hash do conteúdo do script, gerado com SHA-384 |
| `crossorigin` | `anonymous` | Sim | Requerido pelo SRI em requests cross-origin |

### Campos Opcionais

| Atributo | Valor | Descrição |
|----------|-------|-----------|
| `async` | — | Para scripts clássicos, carrega assim que disponível |
| `defer` | — | Para scripts clássicos, espera parse do HTML |
| `type` | `module` | Para imports ESM |

---

## Contrato por Tipo de Script

### Tipo 1: Script Clássico (src)

```html
<script src="URL" integrity="sha384-HASH" crossorigin="anonymous"></script>
```

Aplica-se a:
- Tailwind CSS (classic)
- Chart.js (classic)

### Tipo 2: Módulo ESM (import)

```html
<script type="module">
  import { initializeApp } from "URL";
  // ...
</script>
```

> **Nota**: O SRI para imports ESM aplica-se no `import` (via `import "URL"` ou `import x from "URL"`). No HTML5, a especificação permite `integrity` em tags `<script type="module">` e em URLs de `import`. A prática comum é adicionar `integrity` à tag `<script>` que contém o import.

---

## Formato do Hash

```
integrity = "sha384-" + base64(sha384(conteudo_do_arquivo))
```

- Algoritmo: SHA-384 (obrigatório)
- Codificação: base64 padrão (RFC 4648, com `+`, `/`, `=`)
- Comprimento: ~64 chars base64 (48 bytes de hash)

---

## Regras de Geração

1. **Baixar o arquivo exato** do CDN no momento do commit
2. **Calcular hash localmente** via `openssl dgst -sha384 -binary arquivo.js | openssl base64 -A`
3. **Não usar web search** — CDNs podem variar por região/cache
4. **Validar que o hash bate** antes de commitar (rebaixar e rehashear)

---

## Exemplo Prático

```html
<!-- Tailwind CSS v3.4.0 -->
<script src="https://cdn.tailwindcss.com/3.4.0"
        integrity="sha384-PLACEHOLDER_TAILWIND"
        crossorigin="anonymous"></script>

<!-- Chart.js v4.4.0 -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"
        integrity="sha384-PLACEHOLDER_CHARTJS"
        crossorigin="anonymous"></script>

<!-- Firebase App v11.0.1 -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js";
  // integrity: sha384-PLACEHOLDER_FIREBASE_APP
</script>
```

---

## Contrato de Validação

| Teste | Esperado |
|-------|----------|
| Todos os 5 scripts têm `integrity` | ✅ |
| `integrity` começa com `sha384-` | ✅ |
| `crossorigin="anonymous"` presente | ✅ |
| URLs têm versão fixa | ✅ |
| App carrega sem erro de SRI no console | ✅ |
| Scripts executam (styling, gráfico, Firebase) | ✅ |
# Research: Subresource Integrity (SRI) para 5 scripts externos

**Data**: 2026-09-10  
**Feature**: 002-gitguardian-remediation  
**Contexto**: Adicionar SRI aos 5 scripts do `index.html` (Tailwind, Chart.js, 3 imports ESM Firebase)

---

## 1. Versões exatas confirmadas no CDN

| # | Script | Linha no index.html | URL atual | Versão fixa? |
|---|--------|---------------------|-----------|--------------|
| 1 | **Tailwind CSS** | 23 | `https://cdn.tailwindcss.com` | ❌ **NÃO** — sem versão |
| 2 | **Chart.js** | 71 | `https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js` | ✅ Sim: `4.4.0` |
| 3 | **Firebase App** | 1784 | `https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js` | ✅ Sim: `11.0.1` |
| 4 | **Firebase Auth** | 1788 | `https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js` | ✅ Sim: `11.0.1` |
| 5 | **Firebase Firestore** | 1791 | `https://www.gstatic.com/firebasejs/11.0.1/firebase-firestore.js` | ✅ Sim: `11.0.1` |

**Ação requerida para Tailwind (FR-006)**: Fixar versão antes de gerar SRI. Recomendação: `https://cdn.tailwindcss.com/3.4.0` (versão estável 3.x; v4 ainda em alpha).

---

## 2. Processo de geração dos hashes SHA-384 (FR-005)

**⚠️ IMPORTANTE**: Não tente calcular hashes via pesquisa na web. Os hashes **devem** ser gerados a partir do conteúdo **exato** baixado do CDN no momento do commit.

### Método 1: OpenSSL local (recomendado — offline, determinístico)
```bash
# Baixar cada arquivo
curl -sL "https://cdn.tailwindcss.com/3.4.0" -o tailwind.js
curl -sL "https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js" -o chart.js
curl -sL "https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js" -o firebase-app.js
curl -sL "https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js" -o firebase-auth.js
curl -sL "https://www.gstatic.com/firebasejs/11.0.1/firebase-firestore.js" -o firebase-firestore.js

# Gerar hash SHA-384 em base64 (formato SRI)
openssl dgst -sha384 -binary tailwind.js | openssl base64 -A
openssl dgst -sha384 -binary chart.js | openssl base64 -A
openssl dgst -sha384 -binary firebase-app.js | openssl base64 -A
openssl dgst -sha384 -binary firebase-auth.js | openssl base64 -A
openssl dgst -sha384 -binary firebase-firestore.js | openssl base64 -A
```

### Método 2: Ferramenta online (srihash.org)
1. Acesse https://www.srihash.org/
2. Cole a **URL exata** de cada script (versão fixa)
3. Clique "Generate"
4. Copie o hash `sha384-...` gerado

> **Por que não usar web search**: CDNs podem servir conteúdo diferente por região, cache, ou user-agent. O hash **deve** corresponder ao que o navegador do usuário final vai baixar. Gerar localmente garante match exato.

---

## 3. Estrutura do atributo `integrity`

Formato: `integrity="sha384-<base64-hash>" crossorigin="anonymous"`

Exemplo resultado:
```html
<script 
  src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"
  integrity="sha384-<HASH_AQUI>"
  crossorigin="anonymous">
</script>
```

---

## 4. Placeholders para preenchimento manual

Como não posso acessar URLs externas de forma confiável neste ambiente, deixo os hashes como placeholders. **O desenvolvedor DEVE rodar os comandos acima localmente e preencher**.

| Script | Hash SHA-384 (placeholder) |
|--------|----------------------------|
| Tailwind (v3.4.0) | `sha384-TAILWIND_HASH_AQUI` |
| Chart.js 4.4.0 | `sha384-CHARTJS_HASH_AQUI` |
| Firebase App 11.0.1 | `sha384-FIREBASE_APP_HASH_AQUI` |
| Firebase Auth 11.0.1 | `sha384-FIREBASE_AUTH_HASH_AQUI` |
| Firebase Firestore 11.0.1 | `sha384-FIREBASE_FIRESTORE_HASH_AQUI` |

---

## 5. Decisões e Alternativas

| Decisão | Racional | Alternativas consideradas |
|---------|----------|---------------------------|
| Fixar Tailwind em v3.4.0 | CDN sem versão quebra SRI; v4 alpha | Usar `@tailwindcss/browser` (novo, mas exige build step) |
| Usar `crossorigin="anonymous"` | Necessário para SRI em requests cross-origin | `crossorigin="use-credentials"` (não necessário; sem cookies) |
| SHA-384 (não SHA-256/512) | Padrão SRI recomendado; suporte universal | SHA-512 (maior, sem benefício prático) |
| Placeholders no plano | Não posso baixar/hashear confiavelmente aqui | Tentar web fetch (não confiável; CDN pode variar) |

---

## 6. Validação pós-implementação (quickstart reference)

Após preencher os hashes reais:
1. Abrir `index.html` no navegador
2. DevTools → Network → verificar que todos os 5 scripts carregam com status 200
3. Console limpo (sem erros de "Failed to load resource: integrity check failed")
4. Funcionalidades: styling Tailwind, gráfico Chart.js, login Firebase → tudo OK
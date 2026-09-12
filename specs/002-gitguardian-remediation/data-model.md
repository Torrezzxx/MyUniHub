# Data Model: SRI Script Metadata

**Feature**: 002-gitguardian-remediation  
**Entidade principal**: `ExternalScript` — representa cada `<script src="https://...">` que precisa de SRI

---

## Entidade: ExternalScript

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `id` | string | Sim | Identificador único (ex.: `tailwind`, `chartjs`, `firebase-app`, `firebase-auth`, `firebase-firestore`) |
| `url` | string | Sim | URL completa do script no CDN (com versão fixa) |
| `lineNumber` | integer | Sim | Linha no `index.html` onde a tag `<script>` aparece |
| `type` | enum | Sim | `"classic"` (src) ou `"module"` (ESM import) |
| `integrity` | string | Sim | Hash SRI no formato `sha384-<base64>` |
| `crossorigin` | string | Sim | Sempre `"anonymous"` para CDNs públicas |
| `version` | string | Sim | Versão fixa extraída da URL (ex.: `3.4.0`, `4.4.0`, `11.0.1`) |

---

## Instâncias (5 scripts)

### 1. Tailwind CSS
```json
{
  "id": "tailwind",
  "url": "https://cdn.tailwindcss.com/3.4.0",
  "lineNumber": 23,
  "type": "classic",
  "integrity": "sha384-TAILWIND_HASH_AQUI",
  "crossorigin": "anonymous",
  "version": "3.4.0"
}
```

### 2. Chart.js
```json
{
  "id": "chartjs",
  "url": "https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js",
  "lineNumber": 71,
  "type": "classic",
  "integrity": "sha384-CHARTJS_HASH_AQUI",
  "crossorigin": "anonymous",
  "version": "4.4.0"
}
```

### 3. Firebase App (ESM)
```json
{
  "id": "firebase-app",
  "url": "https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js",
  "lineNumber": 1784,
  "type": "module",
  "integrity": "sha384-FIREBASE_APP_HASH_AQUI",
  "crossorigin": "anonymous",
  "version": "11.0.1"
}
```

### 4. Firebase Auth (ESM)
```json
{
  "id": "firebase-auth",
  "url": "https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js",
  "lineNumber": 1788,
  "type": "module",
  "integrity": "sha384-FIREBASE_AUTH_HASH_AQUI",
  "crossorigin": "anonymous",
  "version": "11.0.1"
}
```

### 5. Firebase Firestore (ESM)
```json
{
  "id": "firebase-firestore",
  "url": "https://www.gstatic.com/firebasejs/11.0.1/firebase-firestore.js",
  "lineNumber": 1791,
  "type": "module",
  "integrity": "sha384-FIREBASE_FIRESTORE_HASH_AQUI",
  "crossorigin": "anonymous",
  "version": "11.0.1"
}
```

---

## Regras de Validação

1. **URL deve ter versão fixa** — não aceitar `cdn.tailwindcss.com` sem `/3.4.0`
2. **Integrity deve ser SHA-384 base64** — formato `sha384-[A-Za-z0-9+/=]+`
3. **Crossorigin = "anonymous"** — obrigatório para SRI em cross-origin
4. **Line number deve bater** — alterar `index.html` na linha exata
5. **Type "module" → usa `import`** — não tem tag `<script src>`, usa `import ... from "url"`

---

## Relacionamentos

- `ExternalScript` → `index.html` (1:1 por linha)
- Nenhum relacionamento entre scripts (cada um independente)

---

## Estado de Migração

| Script | URL fixada? | Hash gerado? | Aplicado no index.html? |
|--------|-------------|--------------|-------------------------|
| tailwind | ❌ (precisa `/3.4.0`) | ❌ | ❌ |
| chartjs | ✅ | ❌ | ❌ |
| firebase-app | ✅ | ❌ | ❌ |
| firebase-auth | ✅ | ❌ | ❌ |
| firebase-firestore | ✅ | ❌ | ❌ |
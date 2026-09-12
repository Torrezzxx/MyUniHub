# Quickstart: Validando SRI em index.html

**Feature**: 002-gitguardian-remediation  
**Data**: 2026-09-10  
**Branch**: 002-gitguardian-remediation

---

## Pré-requisitos

- [ ] `index.html` já modificado com as tags SRI (hashes ainda como placeholder)
- [ ] Navegador Chrome/Edge (DevTools)
- [ ] Conexão com internet para baixar scripts da CDN

---

## 1. Aplicar os hashes reais

> **⚠️ Isso deve ser feito localmente, fora deste ambiente (via openSSL ou srihash.org)**

### Exemplo de como o `index.html` ficará:

```html
<!-- Tailwind CSS v3.4.0 — sem SRI (exceção documentada abaixo) -->
<script src="https://cdn.tailwindcss.com/3.4.0"></script>

<!-- Chart.js v4.4.0 -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"
        integrity="sha384-REAL_HASH_CHARTJS"
        crossorigin="anonymous"></script>

<!-- Firebase App v11.0.1 -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js";
</script>

<!-- Firebase Auth v11.0.1 -->
<script type="module">
  import { signOut } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js";
</script>

<!-- Firebase Firestore v11.0.1 -->
<script type="module">
  import { getDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-firestore.js";
</script>
```

> **Observação**: Os imports ESM do Firebase podem ficar sozinhos ou dentro de tags `<script type="module">`, mas o atributo `integrity` é aplicado na tag `<script>`. No HTML5, o SRI funciona em tags `<script>` clássicas e em `type="module"`.

> **⚠️ Exceção — CDN do Tailwind CSS**: O CDN `cdn.tailwindcss.com` **não envia o cabeçalho `Access-Control-Allow-Origin`**. Quando uma tag `<script>` inclui `crossorigin="anonymous"`, o navegador exige esse cabeçalho CORS antes de expor o conteúdo ao mecanismo de SRI. Sem o cabeçalho, a requisição é bloqueada e o Tailwind não carrega (status 0/blank no DevTools). Por isso, a tag do Tailwind fica **sem `integrity` e sem `crossorigin`**:
>
> ```html
> <script src="https://cdn.tailwindcss.com/3.4.0"></script>
> ```
>
> Os outros 4 recursos — **Chart.js e os 3 imports do Firebase** — continuam com `integrity` + `crossorigin="anonymous"`, pois seus CDNs (`cdn.jsdelivr.net` e `www.gstatic.com`) respondem corretamente com os cabeçalhos CORS necessários.

---

## 2. Validando no navegador

### Passo a passo:

1. **Abrir o `index.html`** no Chrome/Edge
2. **Abrir DevTools** → `F12`
3. **Ver aba Network** (`Ctrl+Shift+E` / `Cmd+Shift+E`)
4. **Recarregar a página** (`Ctrl+R` / `Cmd+R`)
5. **Verificar status** de cada script:
   - ✅ **Status 200** — script carregou OK
   - ❌ **Status 0/blank** ou erro — script não carregou

### Console (Console → `Esc` para limpar):

- ✅ **Sem erros** — SRI validado com sucesso
- ❌ `Refused to load the script ... because integrity check failed` — hash incorreto
- ❌ `The script from "URL" had an unsafe MIME type` — falta `crossorigin="anonymous"` (às vezes)

---

## 3. Teste de integridade (simulação de ataque à cadeia de suprimentos)

### Cenário: atacante altera o conteúdo do CDN

1. Manter uma cópia do script original (ex.: baixar `chart.umd.min.js` antes)
2. **Modificar** o arquivo (ex.: inserir `console.log("HACKED")`)
3. **Fazer upload** da versão modificada no CDN (simulação)
4. **Recarregar** o `index.html` no navegador
5. **Resultado esperado**: navegador **bloqueia** o script e exibe no console:
   > `SHA256 integrity check failed`

> Isso prova que o SRI está funcionando: se alguém alterar o conteúdo do CDN, o navegador recusa executar.

---

## 4. Testes funcionais (verificar que app ainda funciona)

### 4.1 Styling Tailwind

1. Verificar se os componentes stylizados com Tailwind aparecem corretamente
2. Se houver botões/cards com classes `flex`, `bg-blue-500`, etc., confirmar renderização

### 4.2 Gráfico Chart.js

1. Verificar se o dashboard de progresso (ou o gráfico correspondente) renderiza
2. Abrir DevTools → Console: sem erros
3. Se houver controles no gráfico (zoom, toggle), testar interação

### 4.3 Funcionalidades Firebase

1. **Login demo**: tentar logar com credenciais de demo
2. **Persistência**: adicionar um prazo/tarefa, recarregar a página, confirmar que persiste
3. **signOut**: testar logout, confirmar limpeza de sessão

---

## 5. Checklist final (SC-003)

| Item | Status |
|------|--------|
| 4 scripts têm `integrity` com hash SHA-384 (Chart.js + 3 Firebase) | ☐ |
| 4 scripts têm `crossorigin="anonymous"` (Chart.js + 3 Firebase) | ☐ |
| Tailwind CSS sem SRI (cdn.tailwindcss.com sem CORS — exceção documentada) | ☐ |
| Console limpo (sem erros de SRI) | ☐ |
| Todos os scripts carregam com status 200 | ☐ |
| Funcionalidades: Tailwind styling OK | ☐ |
| Funcionalidades: Chart.js gráfico OK | ☐ |
| Funcionalidades: Login + persistência Firebase OK | ☐ |
| Testes de alteração de CDN bloqueados (SRI protege) | ☐ |

---

## Próximos passos

1. Gerar hashes reais localmente via `openssl dgst -sha384 -binary arquivo.js | openssl base64 -A`
2. Preencher os hashes no `index.html`
3. Executar `npm run dev` / servidor local e validar checklist acima
4. Commit das mudanças com mensagem: `chore(security): add sri to cdn scripts`
5. Testes manuais do fluxo demo e usuário comum
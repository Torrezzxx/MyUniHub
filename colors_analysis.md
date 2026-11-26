# MyUniHub - Cores no Código (Análise Completa)

## 1. MODO CLARO (`:root`)

### Fundos
- **--bg-surface: `#ffffff`** - Branco puro (fundo principal)
- **--bg-subtle: `#f8fafc`** - Azul-acinzentado muito claro (10% do branco)

### Textos Principais
- **--text-primary: `#0f172a`** - Azul-navio quase preto (15.9:1 contraste sobre branco)

### Textos Secundários
- **--text-secondary: `#475569`** - Slate meio-escuro (7.0:1 contraste sobre branco)

### Textos Muted/Placeholder
- **--text-muted: `#64748b`** - Slate médio (4.7:1 contraste sobre branco - *regra placeholder*)
- **--text-placeholder: `#94a3b8`** - Slate claro (3.0:1 contraste sobre branco - *regra placeholder oficial*)

### Bordas
- **--border-subtle: `#e2e8f0`** - Slate claro (bordas sutis)

### Destaques e Gradientes
- **--accent: `#2E5A9E`** - Ocean principal (azul forte)
- **--accent-hover: `#1e4a8A`** - Ocean mais escuro (hover states)

---

## 2. MODO ESCURO (`.dark`)

### Fundos
- **--bg-surface: `#0f172a`** - Navy quase puro (fundo principal dark)
- **--bg-subtle: `#1e293b`** - Slate mais escuro (cards e áreas secundárias)

### Textos Principais
- **--text-primary: `#f1f5f9`** - Sky quase branco (15.8:1 sobre #0f172a)

### Textos Secundários
- **--text-secondary: `#cbd5e1`** - Slate claro (8.6:1 sobre #0f172a)

### Textos Muted/Placeholder
- **--text-muted: `#94a3b8`** - Slate claro (5.4:1 sobre #0f172a)
- **--text-placeholder: `#64748b`** - Slate médio (3.6:1 sobre #0f172a)

### Bordas
- **--border-subtle: `#334155`** - Slate escuro (bordas no modo escuro)

---

## 3. ALTO CONTRAPosto E OUTROS MODOS DE ACESSIBILIDADE

### CSS Classes de Override (`html.high-contrast`)

- **`html.high-contrast :focus-visible`**: `outline: 3px solid var(--accent)` = `outline: 3px solid #2E5A9E`
- **`html.high-contrast :focus-visible`**: `outline-offset: 2px` (sempre)
- **`html.dark.high-contrast :focus-visible`**: `outline: 3px solid #fff` (branco sobre fundo escuro)
- **`html.dark.high-contrast :focus-visible`**: `outline-offset: 2px` (sempre)

### Toggle de Alto Contraste (`window.toggleHighContrast`)
- Adiciona classe `high-contrast` ao `html`
- Remove/restitui classe `dark` conforme necessário
- Persiste em `localStorage` como `'hub_theme' = 'high-contrast'`

### Foco Visível Ativado (`html.focus-visible-enabled`)
- `*:focus-visible`: `outline: 3px solid #2E5A9E` !important + `outline-offset: 2px` !important + `border-radius: 4px` !important
- `html.dark.focus-visible-enabled *:focus-visible`: `outline-color: #7c5cbf` !important
- `html.high-contrast.focus-visible-enabled *:focus-visible`: `outline: 3px solid #000` !important + `outline-offset: 2px` !important
- `html.dark.high-contrast.focus-visible-enabled *:focus-visible`: `outline: 3px solid #fff` !important + `outline-offset: 2px` !important

### SVG/Transparency para Alto Contraste
- **`rgba(124,92,191,.18)`** - Glow do violet com 18% de opacidade (visível em alto contraste)
- **`rgba(0,0,0,.05)`** - Fundo de botão skip hover (alto contraste dark)
- **`rgba(255,255,255,.06)`** - Fundo de botão skip hover (alto contraste dark)
- **`rgba(255,255,255,.35)`** - Bordas focus-visible (alto contraste)
- **`rgba(15,23,42,.48)`** - Background pill alto contraste (dark)
- **`rgba(241,245,249,.8)`** - Background pending (lighter theme)
- **`rgba(15,23,42,.5)`** - Background pendingD (dark)

---

## 4. DESTAQUES, BOTÕES, GRADIENTS E CORES SEMÂNTICAS

### Cores Principais da Paleta (Tailwind + Custom)

| Nome | HEX | Uso |
|------|-----|-----|
| **Ocean 700** | `#2E5A9E` | Primary accent - buttons, links, gradients (mais usado) |
| **Violet Soft** | `#7c5cbf` | Secondary accent - gradientes secundários, SVG stops |
| **Ocean 600** | `#1e4a8a` | Ocean hover state (segundo nível) |
| **Emerald 500** | `#22c55e` | Success/Concluído - livros feitos, checkmarks |
| **Amber 500** | `#f59e0b` | Warning/Atenção - próximos prazos, labels |
| **Red 500** | `#ef4444` | Error/Urgent - vencido, erros de formulário |
| **Slate 50** | `#f8fafc` | bg-subtle light (já listado acima) |
| **Slate 100** | `#e2e8f0` | border-subtle light |
| **Slate 200** | `#e2e8f0` | (repetido) |
| **Slate 300** | `#f8fafc` | (variação) |
| **Slate 400** | `#64748b` | text-muted light |
| **Slate 500** | `#94a3b8` | text-placeholder light |
| **Slate 600** | `#475569` | text-secondary light |
| **Slate 700** | `#0f172a` | text-primary light (matches bg-dark) |
| **Slate 800** | `#1e293b` | bg-subtle dark |
| **Slate 900** | `#0f172a` | (match com dark mode) |
| **Violet 500** | `#7c5cbf` | (já listado) |
| **Violet 600** | `#6d28d9` | (referência Tailwind, não usado diretamente) |

### Gradientes Principais (usados em botões, cards, progress bars)

1. **`linear-gradient(135deg, #2E5A9E, #7c5cbf)`** - Ocean para violet (primary: botões, plan mods, book cards)
2. **`linear-gradient(135deg, #1e4a8a, #2E5A9E)`** - Ocean mais escuro p/ mais claro (hover states)
3. **`linear-gradient(90deg, #f59e0b, #ef4444)`** - Amber p/ vermelho (warnings: dias próximos/vencidos)
4. **`linear-gradient(90deg, #22c55e, #16a34a)`** - Emerald p/ verde escuro (concluído extremo)
5. **`linear-gradient(90deg, #f59e0b, #ef4444)`** - Warning gradient (repetido)
6. **`linear-gradient(135deg, #2E5A9E22, #7c5cbf22)`** - Mesmos tons com 12% opacidade (sombras suaves)
7. **`linear-gradient(90deg, #2E5A9E, #7c5cbf)`** - Progress fill vertical (charts, progress bars)

### Estados de Botões e Elementos Interativos

- **bg-gradient-to-r from-ocean-700 to-ocean-600** - Botões principais (CTA)
- **bg-gradient-to-br from-ocean-700 to-violet-soft** - Cards, modais, setup screens
- **bg-white dark:bg-ocean-800** - Pills de aba ativa
- **text-ocean-700 dark:text-ocean-300** - Texto em estado ativo
- **bg-ocean-50 dark:bg-ocean-900/30** - Hover states discretos
- **border-ocean-700** - Bordas de foco/estado ativo

### Cores de Status/Semânticas (lógica de negócio)

| Cor | HEX | Significado |
|-----|-----|-------------|
| `#ef4444` | Red 500 | Vencido/Erro - prazos vencidos, erros de formulário |
| `#f59e0b` | Amber 500 | Aviso - próximos 1-3 dias, warning |
| `#22c55e` | Emerald 500 | Concluído - livros finalizados, ✅ |
| `#94a3b8` | Slate 400 | Placeholder/Médio - estados intermediários |
| `#64748b` | Slate 500 | Text secundário em estados |
| `#2E5A9E` | Ocean 700 | Primary - ações primárias |
| `#7c5cbf` | Violet Soft | Secondary - gradientes, SVG |

### Cores de SVG/Linear Gradients (definidas no `<defs>`)

- **stop offset 0%: `#2E5A9E`** - Azul ocean (gradient starts)
- **stop offset 100%: `#7c5cbf`** - Violet soft (gradient ends)
- **stop offset 0%: `#f59e0b`** - Amber (warnings gradient)
- **stop offset 100%: `#ef4444`** - Red (warnings gradient)
- **Linear ring gradient**: `#2E5A9E` → `#7c5cbf` → `#f59e0b` → `#ef4444` (timer ring)

### Cores Específicas de Componentes

- **timerDigits**: `#2E5A9E` dark, `#64748b` light (texto do cronômetro)
- **otPill**: `bg-gradient-to-r from-amber-400 to-red-500` = `#f59e0b` → `#ef4444`
- **book card border-left**: Dynamic based on book color (`b.color || '#2E5A9E'`)
- **focus-visible ring**: `focus:ring-ocean-700/10` = `rgba(46,90,158,.1)`
- **bg-ocean-50**: `#f0f6ff` (50% da paleta ocean)
- **bg-ocean-900/30**: `rgba(15,23,42,.3)` (900% + 30% opacity)

### Definições Importantes de Variáveis

No bloco `:root` (linhas 88-96 de index.html):
```
--text-primary: #0f172a;
--text-secondary: #475569;
--text-muted: #64748b;
--text-placeholder: #94a3b8;
--bg-surface: #ffffff;
--bg-subtle: #f8fafc;
--border-subtle: #e2e8f0;
--accent: #2E5A9E;
--accent-hover: #1e4a8a;
```

No bloco `.dark` (linhas 102-110):
```
--text-primary: #f1f5f9;
--text-secondary: #cbd5e1;
--text-muted: #94a3b8;
--text-placeholder: #64748b;
--bg-surface: #0f172a;
--bg-subtle: #1e293b;
--border-subtle: #334155;
```

Nota: `--font-scale: 1;` é uma variável CSS usada para ajuste proporcional do tamanho do texto (accessibilidade), não uma cor.

### Cores Não-Existentes (não encontradas no código)

- Não foram encontrados códigos `#fbbf24` (amber-200), `#f87171` (red-400), `#a5f3cf` (emerald-200) ou outros da paleta Tailwind não usadas.
- Cores `#1e293b`, `#334155`, `#64748b`, `#94a3b8` são do sistema Slate (usados como texto/bordas) e não como nomes de classe Tailwind diretos (exceto `#64748b` e `#94a3b8` em alguns contexts).
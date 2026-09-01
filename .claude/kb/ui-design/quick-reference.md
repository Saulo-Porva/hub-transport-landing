# UI Design — Quick Reference (Workforce Management)

## Tokens reais — `tailwind.config.ts` (fonte única de verdade)

```ts
colors: {
  bg: "#0F1117",                              // fundo da página
  surface: "#1A1D27",                         // cards, sidebar, header de tabela
  "surface-hover": "#242836",                 // hover de linha/item
  border: "#2A2E3A",                          // bordas, divisores
  brand: { DEFAULT: "#3B82F6", hover: "#2563EB" },  // ação primária
  text: { primary: "#F5F6FA", secondary: "#9CA3AF" },
  success: "#22C55E",
  warning: "#F59E0B",
  danger: "#EF4444",
}
borderRadius: { card: "1rem", input: "0.75rem" }
fontFamily: { sans: ["Inter", "system-ui", "-apple-system", "sans-serif"] }
```

Nunca usar `brand-red`, `oklch(...)`, ou qualquer token que não esteja na lista acima — se precisar de
um tom novo, adicionar em `tailwind.config.ts` primeiro, nunca hardcoded na classe.

## Layout de Página de Gestão — Padrão Único (as 8 telas)

```
┌─────────────────────────────────────────────────────────┐
│ PageHeader: <h1>Título</h1>            [+ Novo →]        │  ← ação abre modal, nunca inline
├─────────────────────────────────────────────────────────┤
│ Toolbar: [🔍 Buscar...] [Filtro ▾] [Filtro ▾]  X de Y    │  ← direto acima da tabela que filtra
├─────────────────────────────────────────────────────────┤
│ Table: cabeçalho fixo, linhas com hover, ações à direita │
├─────────────────────────────────────────────────────────┤
│ Paginação — só renderiza se total > itens por página     │
└─────────────────────────────────────────────────────────┘
```

Nunca colocar o formulário de criar entre a toolbar e a tabela — isso quebra a leitura vertical da
lista (reportado pelo usuário como layout "sanduíche").

## Tipografia

| Tailwind | px | Uso |
|----------|----|-----|
| `text-2xl font-bold` | 24 | Título de página (h1) |
| `text-xl font-semibold` | 20 | Título de seção |
| `text-base font-medium` | 16 | Labels, nav items |
| `text-base` | 16 | Corpo de texto principal |
| `text-sm` | 14 | Linhas de tabela, metadados |
| `text-xs` | 12 | Badges — nunca texto principal ou dado crítico (ex: horário de turno) |

## Espaçamento — Base 4px

| Token | px | Uso |
|-------|----|-----|
| `gap-1` / `p-1` | 4 | Espaço entre ícone e texto |
| `gap-2` / `p-2` | 8 | Itens inline, célula de tabela compacta |
| `gap-4` / `p-4` | 16 | Padding interno de card |
| `gap-6` / `p-6` | 24 | Entre seções |
| `gap-8` / `p-8` | 32 | Entre blocos de página |

## Estados — Bordas e Rings

```
default:  border-border
hover:    bg-surface-hover
focus:    ring-2 ring-brand/40 outline-none border-brand
error:    border-danger bg-danger/5
disabled: opacity-50 cursor-not-allowed pointer-events-none
```

## Contraste WCAG — Combinações deste projeto

| Combinação | Ratio aproximado | Status |
|------------|-------|--------|
| `text-primary` (#F5F6FA) em `bg` (#0F1117) | ~15:1 | ✓ AAA |
| `text-secondary` (#9CA3AF) em `surface` (#1A1D27) | ~5.3:1 | ✓ AA |
| `white` em `brand` (#3B82F6) | ~3.7:1 | ✓ AA para texto grande/botão; evitar texto pequeno |
| `text-secondary` em `surface-hover` (#242836) | ~4.6:1 | ✓ AA |

## Raios de Borda — Padrão do Projeto

| Elemento | Classe |
|---------|--------|
| Cards, containers, modais | `rounded-card` |
| Botões, inputs, selects, badges | `rounded-input` |

## Tabela / Grid (componente central deste app)

- Cabeçalho: `border-b border-border text-text-secondary`, sticky quando a tabela rola horizontalmente
  (ex: grid de Escalas — primeira coluna com `sticky left-0 bg-surface`).
- Linha: `border-b border-border last:border-0`, hover opcional `hover:bg-surface-hover` quando clicável.
- Coluna de ação: sempre a última, botões em `text-sm`, nunca ícone sem label/aria-label.
- Densidade > espaço generoso — este não é um layout de cards mobile.

## Paginação / Contador — Regra

Só renderizar o bloco de paginação (números de página + texto "Mostrando X–Y de Z") quando
`total > porPagina`. Quando tudo cabe em uma página, não mostrar contador nenhum — ele não comunica
nada de útil e gera confusão (relatado pelo usuário: "Mostrando 1–2 de 2????").

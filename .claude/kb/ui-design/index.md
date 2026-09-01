# KB: UI Design

> Padrões de design visual para o Workforce Management (Copagril) — SaaS B2B **desktop-first, dark mode**,
> para gerentes de RH/loja montarem escalas de trabalho.

## Contexto do Projeto

**App:** Workforce Management (Escala de Trabalho) — piloto Copagril
**Usuários:** Gerentes de RH/loja, desktop, mouse e teclado, sessões longas e recorrentes
**Stack:** Next.js 15 App Router, Tailwind CSS v3.4 (`darkMode: "class"`), `cva` para variantes
**Cor de marca:** `#3B82F6` (azul, token `brand` em `tailwind.config.ts`) sobre fundo escuro (`#0F1117`)

> Este KB já foi, num passado próximo, copiado de outro projeto (PR Trasporti — app de viagens para
> motoristas, mobile-first, cor de marca vermelha). Isso já causou inconsistência visível no produto
> (tokens light-mode/vermelhos vazando em telas que deveriam ser dark-mode/azul). O conteúdo abaixo foi
> recalibrado para este projeto — não reintroduzir referências ao produto anterior.

## Quando consultar este KB

- Criando ou refinando componentes visuais (cards, forms, badges, botões, tabelas)
- Definindo tokens de design (cores, tipografia, espaçamento, sombras) — sempre a partir de `tailwind.config.ts`
- Revisando inconsistências visuais entre as telas de gestão (Funcionários, Unidades, Escalas, Grupo de
  Regras, Feriados, Jornada, Sazonalidade, Analytics)
- Implementando estados de UI (loading, error, empty, success)
- Garantindo acessibilidade visual (contraste, foco de teclado)

## Navegação

### Conceitos
| Arquivo | Conteúdo |
|---------|----------|
| [color-theory.md](concepts/color-theory.md) | Paleta semântica, WCAG, tokens de cor |
| [typography.md](concepts/typography.md) | Escala tipográfica, legibilidade |
| [spacing-layout.md](concepts/spacing-layout.md) | Grid, base 4px |
| [visual-hierarchy.md](concepts/visual-hierarchy.md) | Peso, tamanho, foco único, agrupamento |
| [component-design.md](concepts/component-design.md) | Estados, variantes, microinterações |

### Padrões
| Arquivo | Conteúdo |
|---------|----------|
| [dashboard-design.md](patterns/dashboard-design.md) | Layout de telas administrativas — a principal referência deste app |
| [form-design.md](patterns/form-design.md) | Inputs, selects, modais de cadastro/edição |
| [tailwind-design-system.md](patterns/tailwind-design-system.md) | Tokens reais deste projeto, cva, cn() utility |

> `mobile-first-ui.md` (BottomNav, thumb zones) não se aplica a este produto — mantido apenas como
> referência histórica, não seguir para telas novas.

## Princípios Inegociáveis para este Projeto

1. **Um único padrão de página de gestão:** PageHeader (título + ação "Novo") → Toolbar (busca/filtro) →
   Tabela → Paginação (só quando há mais de 1 página). As 8 telas de gestão devem seguir exatamente essa
   ordem e esse layout — nenhuma variação por tela.
2. **Cadastro/edição em modal**, nunca inline no meio da página (evita o form "sanduichado" entre filtro e lista).
3. **Contraste:** mínimo 4.5:1 para texto, preferir 7:1.
4. **Feedback imediato:** toda ação (salvar, filtrar, paginar) deve ter resposta visual em < 200ms.
5. **Consistência de raio:** `rounded-input` (0.75rem) para botões, inputs e badges; `rounded-card` (1rem)
   para cards e containers — ambos definidos em `tailwind.config.ts`, nunca hardcoded.
6. **Tokens únicos:** `bg`, `surface`, `surface-hover`, `border`, `brand`/`brand-hover`, `text-primary`,
   `text-secondary`, `success`, `warning`, `danger` — são os ÚNICOS tokens de cor deste projeto.

# UX — Quick Reference (Workforce Management)

## Heurísticas de Nielsen — Auditoria (achados reais, 2026-07)

| # | Heurística | Status antes do redesenho |
|---|-----------|---------------------------|
| 1 | Visibilidade do status do sistema | ⚠️ Toast existe, mas grade de Escalas não indicava estado pendente/erro com clareza |
| 2 | Match com o mundo real | ✗ Datas como "07-22" em vez de "Ter 22/07" — não intuitivo |
| 3 | Controle e liberdade | ✓ Cancelar/reverter alocação existe (marcar folga) |
| 4 | Consistência e padrões | ✗ Cada tela de gestão tinha um layout diferente (filtro no topo, form no meio, lista embaixo, sem ordem fixa) |
| 5 | Prevenção de erros | ✗ Clique duplo na grade de Escalas podia gerar duas alocações ativas no mesmo dia (bug real, corrigido com upsert + índice único) |
| 6 | Reconhecimento vs recall | ⚠️ Filtros por select ajudam, mas posição inconsistente entre telas |
| 7 | Flexibilidade e eficiência | ⚠️ Sem atalho de teclado; navegação semana/mês existe |
| 8 | Estética e design minimalista | ✗ "Tudo na mesma cor", pouca hierarquia visual na grade de Escalas |
| 9 | Recuperação de erros | ⚠️ Mensagens de erro existem via toast, mas sem sempre indicar o que fazer a seguir |
| 10 | Ajuda e documentação | ✗ Sem hint/tooltip nos campos de regra CLT (limites, mínimos) |

## Padrão de Página de Gestão — Único para as 8 telas

```
PageHeader (título + botão "+ Novo" → abre modal)
  → Toolbar (busca + filtros, imediatamente acima da lista)
    → Tabela (dados)
      → Paginação (só se total > itens por página)
```

Nunca: formulário de cadastro renderizado entre a toolbar e a tabela (relatado pelo usuário como layout
"sanduíche" — Unidades era o pior caso).

## Contador / Paginação — Regra

| Situação | O que mostrar |
|----------|---------------|
| `total <= porPagina` | Nada — sem paginação, sem contador |
| `total > porPagina` | "Mostrando X–Y de Z" + números de página |

"Mostrando 1–2 de 2" sem nenhuma paginação real é ruído — o usuário não consegue interpretar o que
aquele número está contando.

## Grade de Escalas — Requisitos de Legibilidade

| Elemento | Errado (estado anterior) | Correto |
|----------|--------------------------|---------|
| Cabeçalho do dia | `07-22` | `Ter` + `22/07` — dia da semana sempre visível |
| Fim de semana / feriado | Mesma cor que dia útil | Tom de fundo distinto na coluna |
| Célula com alocação | Badge de status sempre visível + horário, competindo visualmente | Horário como informação primária; badge só para estados de exceção (gap de pessoal, aguardando revisão) |
| Duas alocações no mesmo dia | Possível (bug de concorrência) | Impossível — 1 alocação ativa por funcionário/dia, garantido por índice único no banco |
| Formato de hora | `08:00:00` (com segundos, do tipo `time` do Postgres) | `08:00` — sempre truncar segundos na consulta (`to_char(col, 'HH24:MI')`) |

## Friction Points Identificados (achados reais, 2026-07)

| Tela | Friction | Severidade |
|------|---------|-----------|
| Escalas | Cabeçalho de dia não intuitivo (`07-22` em vez de dia da semana) | Alta |
| Escalas | Duas alocações ativas no mesmo dia por condição de corrida no clique | Crítica (corrigido) |
| Escalas | Horário exibido com segundos (`08:00:00`) | Média (corrigido) |
| Unidades | Formulário de cadastro renderizado no meio da página, entre filtro e lista | Alta |
| Todas as 8 telas | Padrão de filtro/cadastro/paginação diferente em cada uma | Alta |
| Funcionários/Unidades | "Mostrando 1–2 de 2" exibido mesmo sem paginação real | Média |

## WCAG 2.1 — Checklist Rápida

- [ ] Todos os inputs têm `<label>` associado via `for`/`id`
- [ ] Erros identificados por texto (não só cor)
- [ ] Contraste de texto ≥ 4.5:1
- [ ] Botões têm texto descritivo ou `aria-label`
- [ ] Foco visível em todos os elementos interativos (teclado, não só mouse)
- [ ] Não depende só de cor para comunicar status (badge sempre tem texto)

## Estados de UI — Quando Usar Cada Um

| Estado | Quando mostrar | Duração típica |
|--------|---------------|----------------|
| Skeleton | Carregamento inicial de dados | 0 → dados chegam |
| Spinner inline / `disabled` + opacidade | Ação em botão em andamento (ex: célula de escala salvando) | 0 → resposta |
| Toast sucesso | Após salvar/criar com sucesso | 3s auto-dismiss |
| Toast erro | Após falha em ação | Permanente até dispensar |
| Empty state | Lista/dados sem itens | Permanente, com CTA claro |
| Error state | Falha no carregamento inicial | Com botão "Tentar novamente" |

# KB: UX (User Experience)

> Padrões de experiência do usuário para o Workforce Management (Copagril) — SaaS B2B **desktop-first**,
> gerentes de RH/loja montando escalas de trabalho.

## Contexto do Projeto

**App:** Workforce Management (Escala de Trabalho) — piloto Copagril
**Usuários:** Gerentes de RH/loja — desktop, mouse e teclado, sessões longas e recorrentes (não uso
esporádico entre outras tarefas)
**Tipo de uso:** Tarefas administrativas — cadastrar/editar funcionários e unidades, montar/ler a grade
de escala semanal ou mensal, configurar regras (CLT) e feriados, consultar indicadores (Analytics)
**Criticidade:** Alta — a grade de escala tem implicação legal direta (CLT); dado incorreto ou ambíguo
na tela pode gerar decisão errada do gerente

> Este KB foi, num passado próximo, copiado de outro projeto (PR Trasporti — app de campo para
> motoristas, mobile, touch, offline-first). Esse desalinhamento já é uma causa raiz identificada de
> inconsistência visual/UX no produto. O conteúdo abaixo foi recalibrado para este projeto — não
> reintroduzir personas ou heurísticas de campo/mobile aqui.

## Quando consultar este KB

- Projetando ou revisando fluxos de navegação entre as 8 telas de gestão
- Identificando friction points em formulários, filtros ou na grade de escala
- Validando acessibilidade e usabilidade (teclado, contraste, foco)
- Projetando estados de erro/sucesso/loading/vazio
- Garantindo que o mesmo tipo de ação (filtrar, criar, editar, paginar) siga sempre o mesmo padrão

## Navegação

### Conceitos
| Arquivo | Conteúdo |
|---------|----------|
| [usability-principles.md](concepts/usability-principles.md) | 10 heurísticas de Nielsen, lei de Fitts, Hick |
| [information-architecture.md](concepts/information-architecture.md) | Navegação entre as 8 telas, mental models |
| [user-flows.md](concepts/user-flows.md) | Happy path, error paths, task completion |
| [accessibility.md](concepts/accessibility.md) | WCAG 2.1 AA, ARIA, keyboard nav |

### Padrões
| Arquivo | Conteúdo |
|---------|----------|
| [form-ux.md](patterns/form-ux.md) | Modal de cadastro/edição, smart defaults, validação |
| [feedback-patterns.md](patterns/feedback-patterns.md) | Toasts, skeletons, empty states, error recovery |
| [data-table-ux.md](patterns/data-table-ux.md) | Listas grandes, filtro, paginação, ações de linha — o padrão central deste app |

> `mobile-ux.md` e `field-worker-ux.md` (thumb zones, offline, luvas) não se aplicam a este produto —
> mantidos apenas como referência histórica, não seguir para telas novas. `admin-vs-field-ux.md` também
> não se aplica: este app tem uma persona única (gerente admin), não duas.

## Princípios Inegociáveis para este Projeto

1. **Zero ambiguidade em ações que sobrescrevem dado existente:** trocar o turno de um dia já preenchido,
   desligar funcionário, ou eliminar um grupo de regras sempre requerem confirmação explícita
2. **Feedback em < 200ms:** o gerente não pode achar que a ação travou
3. **Mesmo padrão em toda tela de gestão:** filtro sempre acima da lista, criar/editar sempre em modal,
   paginação só quando há mais de uma página — nenhuma tela foge disso
4. **Erros com recuperação clara:** nunca mostrar mensagem de erro sem indicar o que fazer
5. **Datas sempre legíveis:** dia da semana + data (ex: "Ter 22/07"), nunca só um número ambíguo
   (ex: "07-22")

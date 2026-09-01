# Data Table UX — Padrão Central deste App

> As 8 telas de gestão (Funcionários, Unidades, Escalas, Grupo de Regras, Feriados, Jornada,
> Sazonalidade, Analytics) são, na essência, listas filtráveis de registros administrativos. Este é o
> componente de UX mais repetido no app — a consistência aqui vale mais do que qualquer refinamento
> visual isolado numa tela.

## Anatomia do Padrão

```
1. PageHeader     — título da entidade + botão "+ Novo {Entidade}" (abre modal, não navega)
2. Toolbar         — busca (texto livre) + filtros (select), lado a lado, acima da tabela
3. Contador        — "Mostrando X–Y de Z" — só existe quando há mais de 1 página
4. Tabela          — cabeçalho fixo, linha por registro, ações à direita
5. Paginação       — Anterior/Próxima + números — só existe quando há mais de 1 página
```

## Por que "Novo" abre modal, e não navega para outra rota

Navegar para `/funcionarios/novo` perde o contexto da lista (filtro aplicado, página atual) e obriga
o gerente a refazer o caminho de volta. Um modal (`FormDialog`) preserva o contexto, e no fluxo deste
app o formulário de cadastro nunca é grande o suficiente para justificar uma página própria.

## Por que o form nunca fica inline na página

Um formulário sempre visível entre a toolbar e a tabela força o olho a pular por cima dele toda vez
que o gerente só quer ler a lista — e em telas com o form fixo no meio, ele passa a competir
visualmente com o header e a lista (reportado pelo usuário como "sanduíche"). Regra: cadastro/edição
= modal. Sempre.

## Filtro

- Sempre imediatamente acima da tabela que ele filtra — nunca separado por outro bloco (como o form
  de criar).
- Aplicado via `searchParams` (GET), preservando estado na URL — permite compartilhar/voltar com filtro
  mantido.
- Reseta a paginação para a página 1 ao mudar o filtro.

## Paginação e Contador

```
total <= porPagina  → não renderizar nada (nem contador, nem números de página)
total >  porPagina  → "Mostrando {inicio}–{fim} de {total}" + números de página
```

Nunca mostrar "Mostrando 1–2 de 2" — se a lista inteira cabe numa página, o contador não comunica
nada além de ruído.

## Ações de Linha

- Sempre a última coluna da tabela, alinhada à direita.
- Texto (não só ícone) para a ação, ou `aria-label` se for ícone.
- Ação destrutiva ou irreversível (desligar, eliminar) sempre com `ConfirmDialog`.
- Ação de eliminar que pode falhar por dependência (ex: grupo de regras em uso) deve retornar o motivo
  específico, não uma falha genérica.

## Empty State (lista vazia)

- Nunca só "Nenhum registro encontrado" sem contexto.
- Sempre: por que está vazio (sem filtro aplicado vs. filtro sem resultado) + CTA (criar o primeiro
  registro, ou limpar filtro).

## Aplicação na Grade de Escalas

A grade de Escalas é uma variação deste padrão (tabela 2D — funcionário × dia, não uma lista linear),
mas os mesmos princípios valem: cabeçalho legível (dia da semana, não só data numérica), ações de
edição em modal/popover (não navegação), e nenhuma ambiguidade sobre qual célula está sendo editada.
Ver `.claude/kb/workforce-scheduling/` para as regras de negócio (CLT) que a grade precisa respeitar.

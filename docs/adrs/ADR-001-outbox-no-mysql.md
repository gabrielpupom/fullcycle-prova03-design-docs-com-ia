# ADR-001: Outbox no MySQL

## Status

Aceito.

## Contexto

O envio HTTP síncrono durante a mudança de status foi descartado porque a lentidão ou
indisponibilidade de um cliente prolongaria a transação e tornaria o sucesso da mudança
dependente de um sistema externo. A reunião escolheu o padrão outbox, persistido no
MySQL que o sistema já usa ([transcrição, 09:04–09:08](../../TRANSCRICAO.md#L32-L58)).

O código atual já oferece o limite transacional necessário: `OrderService.changeStatus`
executa em `this.prisma.$transaction` e, no mesmo callback, atualiza o pedido, o estoque
quando aplicável e `orderStatusHistory`
([`OrderService.changeStatus`](../../src/modules/orders/order.service.ts#L126-L179)). O
datasource do Prisma é MySQL
([`prisma/schema.prisma`](../../prisma/schema.prisma#L1-L8)).

## Decisão

Persistir os eventos de mudança de status em uma outbox no MySQL existente. A inserção
na outbox fará parte da mesma transação SQL usada por `OrderService.changeStatus` para
alterar o pedido e registrar o histórico. Se a inserção falhar, toda a transação sofrerá
rollback; se houver commit, a mudança e seu evento ficarão registrados juntos
([transcrição, 09:40–09:41](../../TRANSCRICAO.md#L238-L244)).

O disparo HTTP não fará parte dessa transação. Ele será responsabilidade do consumidor
assíncrono da outbox.

## Alternativas Consideradas

- **Chamada HTTP síncrona na transação:** descartada por acoplar a mudança de status à
  latência e disponibilidade do endpoint externo, além de deixar ambíguo quando aplicar
  rollback ([transcrição, 09:04–09:06](../../TRANSCRICAO.md#L32-L48)).
- **Redis Streams ou Redis Cluster:** descartado porque introduziria infraestrutura
  adicional considerada overengineering para o time
  ([transcrição, 09:07](../../TRANSCRICAO.md#L50-L52)).

## Consequências

### Positivas

- A mudança de status e o registro do evento compartilham o mesmo resultado
  transacional, removendo a janela em que um deles poderia ser confirmado sem o outro.
- A solução reutiliza MySQL e Prisma, sem adicionar um broker à operação.

### Negativas e trade-off

- Novas tabelas, índices e consultas passam a disputar recursos do banco transacional.
- A outbox exige worker, monitoramento e uma política de limpeza ou arquivamento. O
  prazo de retenção não é fixado por este ADR; o arquivamento foi deixado fora do escopo
  atual ([transcrição, 09:08](../../TRANSCRICAO.md#L54-L58)).
- O trade-off aceito é assumir custo operacional e crescimento de dados no MySQL para
  obter atomicidade sem operar uma infraestrutura de mensageria adicional.

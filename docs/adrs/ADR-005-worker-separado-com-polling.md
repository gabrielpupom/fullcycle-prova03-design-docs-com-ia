# ADR-005: Worker separado com polling

## Status

Aceito, com ordering durante backoff em aberto.

## Contexto

A outbox precisa ser consumida sem vincular o ciclo de vida da entrega ao processo da
API. A reunião considerou o intervalo de 2 segundos compatível com o objetivo de entrega
abaixo de 10 segundos e escolheu operação inicial com um único worker
([transcrição, 09:09–09:13](../../TRANSCRICAO.md#L60-L86)).

## Decisão

Executar o consumidor da outbox em processo separado da API, usando o mesmo banco e a
mesma stack, mas uma instância de `PrismaClient` própria do processo. O worker fará
polling a cada 2 segundos, buscando os registros pendentes mais antigos. A operação
inicial terá um único worker
([transcrição, 09:09–09:13](../../TRANSCRICAO.md#L60-L86) e
[09:29–09:30](../../TRANSCRICAO.md#L174-L180)).

Não há garantia de ordering global nem de ordering em uma futura operação
multi-worker. A intenção registrada para esta fase é preservar a ordem por `order_id`
enquanto houver single-worker, sujeita à lacuna abaixo
([transcrição, 09:12–09:14](../../TRANSCRICAO.md#L78-L88)).

## Questão em aberto

Single-worker e seleção por antiguidade não resolvem ordering quando um evento anterior
entra em backoff e o evento seguinte do mesmo pedido continua elegível. A reunião não
definiu esse comportamento; portanto, este ADR não garante a ordem nesse cenário.

## Proposta derivada

Para revisão, aplicar head-of-line blocking por `order_id`: um evento posterior do mesmo
pedido só se torna elegível depois que o anterior é entregue ou enviado à DLQ. A
proposta preserva a ordem por pedido durante backoff, ao custo de bloquear
temporariamente as entregas daquele pedido. Ela **não é uma decisão fechada da reunião**.

## Alternativas Consideradas

- **Worker dentro do processo da API:** descartado para não subordinar o consumidor aos
  reinícios e ao ciclo de vida da instância HTTP
  ([transcrição, 09:11](../../TRANSCRICAO.md#L70-L76)).
- **Trigger do MySQL como notificação:** descartado porque o trigger executa SQL, mas não
  notifica diretamente um processo externo
  ([transcrição, 09:09](../../TRANSCRICAO.md#L60-L64)).
- **Broker ou infraestrutura externa:** não escolhido nesta fase porque adicionaria
  operação considerada desnecessária para o time
  ([transcrição, 09:07](../../TRANSCRICAO.md#L50-L52)).

## Consequências

### Positivas

- O ciclo de vida do worker fica separado do servidor HTTP.
- Polling no MySQL evita um broker adicional e o intervalo decidido é compatível com a
  meta de latência desta fase.
- Single-worker simplifica concorrência na operação inicial.

### Negativas e trade-off

- Polling adiciona consultas recorrentes ao MySQL, inclusive quando não há trabalho.
- Um único worker limita throughput e disponibilidade e não oferece caminho automático
  para escala horizontal.
- Head-of-line, se aprovado, troca latência dos eventos posteriores pela preservação de
  ordem do pedido.
- O trade-off aceito é priorizar simplicidade operacional nesta fase, mantendo escala e
  ordering sob falha como limitações explícitas.

# ADR-002: Retry com backoff e DLQ

## Status

Aceito, com a contagem exata de tentativas e a classificação de falhas em aberto.

## Contexto

Endpoints de clientes podem ficar temporariamente indisponíveis. A reunião escolheu
retentativas com esperas crescentes e um limite, evitando tanto abandonar o evento na
primeira falha quanto mantê-lo pendente indefinidamente. Para falhas permanentes, foi
escolhida uma dead-letter queue persistida em tabela separada
([transcrição, 09:15–09:19](../../TRANSCRICAO.md#L90-L116)).

## Decisão

Aplicar retry com backoff às falhas que a política de entrega classificar como
retentáveis. Quando a política aprovada se esgotar, remover a entrega do fluxo normal e
registrá-la em uma DLQ separada, preservando payload, motivo da falha e timestamp. A
recuperação será manual por endpoint administrativo, que recoloca o registro como
pendente ([transcrição, 09:15 e 09:18–09:19](../../TRANSCRICAO.md#L90-L116)).

Esta decisão fecha o uso de backoff, a separação da DLQ e o replay manual. Ela não fecha
o número de chamadas HTTP nem o tratamento de cada categoria de resposta ou erro.

## Questões em aberto

- **Contagem e agenda exatas:** a reunião fala em “cinco tentativas” e também declara
  cinco esperas (`1m/5m/30m/2h/12h`). Não ficou reconciliado se a tentativa inicial
  pertence ao total de cinco ([transcrição, 09:15–09:17](../../TRANSCRICAO.md#L92-L108)
  e [resumo das 09:48](../../TRANSCRICAO.md#L280-L284)).
- **Classificação de falhas:** ainda é preciso decidir o tratamento de respostas HTTP,
  redirects e falhas de DNS, TLS, conexão e timeout antes de determinar o que recebe
  retry ou vai para DLQ. O único caso fechado fora deste ADR é que timeout de 10 segundos
  conta como falha para retry ([transcrição, 09:42](../../TRANSCRICAO.md#L246-L252)).

## Proposta derivada

Para revisão, interpretar a política como uma tentativa inicial imediata seguida de
cinco retries após `1m`, `5m`, `30m`, `2h` e `12h`, totalizando seis chamadas HTTP. Essa
proposta preserva todos os intervalos citados, mas **não é uma decisão fechada da
reunião**.

## Alternativas Consideradas

- **Três tentativas:** descartada por cobrir uma janela curta demais para
  indisponibilidades já observadas ([transcrição, 09:16](../../TRANSCRICAO.md#L98-L102)).
- **Retry indefinido:** descartado porque poderia manter eventos pendentes para sempre
  quando um cliente deixasse de operar ([transcrição, 09:15](../../TRANSCRICAO.md#L92-L96)).
- **Manter status `failed` na própria outbox:** não escolhido; a tabela separada mantém
  a leitura da outbox principal focada no fluxo ativo e preserva evidências para suporte
  e replay ([transcrição, 09:17–09:18](../../TRANSCRICAO.md#L108-L114)).

## Consequências

### Positivas

- Falhas transitórias podem se recuperar sem intervenção imediata.
- O limite impede retries eternos, e a DLQ separada conserva o diagnóstico e permite
  recuperação manual.

### Negativas e trade-off

- O backoff aumenta o tempo até a entrega e exige agendamento, persistência de estado e
  observabilidade das tentativas.
- DLQ e replay adicionam armazenamento e trabalho operacional; replay também pode
  produzir nova entrega de um evento já recebido.
- O trade-off aceito é limitar o esforço automático e manter um caminho de recuperação
  manual, deixando a agenda exata para aprovação posterior.

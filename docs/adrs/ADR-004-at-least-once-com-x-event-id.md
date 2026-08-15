# ADR-004: At-least-once com X-Event-Id

## Status

Aceito. A estabilidade da identidade em retry e replay permanece como proposta.

## Contexto

Uma entrega pode ser repetida quando o worker não consegue distinguir uma falha antes
do envio de uma falha depois do processamento pelo cliente. A reunião aceitou
explicitamente que duplicatas podem chegar e atribuiu ao cliente a deduplicação por um
identificador do evento ([transcrição, 09:24–09:26](../../TRANSCRICAO.md#L146-L158)).

## Decisão

Oferecer garantia de entrega **at-least-once**. Cada evento receberá um UUID quando
entrar na outbox, enviado no header `X-Event-Id`. Duplicatas são permitidas, e o cliente
é responsável por deduplicá-las usando esse valor
([transcrição, 09:24–09:26](../../TRANSCRICAO.md#L146-L158)).

## Proposta derivada

Para revisão, retry deve preservar o mesmo `event_id` e a identidade da entrega; replay
de DLQ deve preservar o `event_id`, criar uma nova tentativa e manter a auditoria. Essa
estabilidade é necessária para a deduplicação continuar útil, mas **não foi fechada pela
reunião** e não integra a decisão aceita deste ADR.

## Alternativas Consideradas

- **Exactly-once:** descartada porque exigiria coordenação entre plataforma e cliente e
  adicionaria complexidade desproporcional ao caso de uso
  ([transcrição, 09:25](../../TRANSCRICAO.md#L148-L154)).

## Consequências

### Positivas

- A plataforma pode repetir entregas diante de incerteza sem exigir um protocolo
  distribuído de exactly-once.
- `X-Event-Id` oferece ao cliente uma chave explícita para tornar seu processamento
  idempotente.

### Negativas e trade-off

- O cliente precisa persistir ou reconhecer IDs já processados e proteger seus efeitos
  contra duplicatas.
- Sem aprovação da proposta de estabilidade, retry e replay ainda não têm um contrato
  completo de identidade.
- O trade-off aceito é transferir a deduplicação ao consumidor para manter o mecanismo
  de entrega simples e recuperável.

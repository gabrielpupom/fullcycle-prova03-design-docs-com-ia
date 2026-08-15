# FDD — Webhooks outbound de status de pedidos

**Status:** em revisão. Este documento é uma especificação de implementação; não descreve
webhooks como capacidade já existente.

**Fontes normativas:** [RFC](RFC.md), [ADR-001](adrs/ADR-001-outbox-no-mysql.md),
[ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md),
[ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md),
[ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md),
[ADR-005](adrs/ADR-005-worker-separado-com-polling.md),
[ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) e
[matriz de evidências](superpowers/evidence/evidence-matrix.md).

### Convenções de decisão e rastreabilidade

- `FECHADO: <Evidence ID>` identifica requisito confirmado na matriz.
- `CÓDIGO: <Evidence ID>` identifica somente o comportamento comprovado hoje no
  repositório.
- `PROPOSTA_DERIVADA: <Evidence ID ou FDD-PROP-*> — EM REVISÃO` identifica uma escolha
  implementável, mas ainda não aprovada como requisito. A implementação não deve
  tratá-la como decisão fechada.
- `QUESTÃO_ABERTA: <Evidence ID>` identifica uma decisão que precisa de aprovação antes
  de estabilizar o contrato correspondente.

Os nomes de campos, tipos, índices, estados, rotas completas, envelopes, status HTTP,
cardinalidade de fan-out, lease, head-of-line, bytes/formato HMAC, operação do grace
period, redirects, classificação HTTP/rede e tracing abaixo são propostas ou questões
abertas sempre que a matriz não os fecha.

## Contexto e motivação técnica

Atlas Comercial, MaxDistribuição e Nova Cargo precisam receber mudanças de status de
pedidos, e abaixo de dez segundos é a referência de tempo real para a primeira entrega
`[FECHADO: EV-TR-001-A, EV-TR-001-B]`. O fluxo é exclusivamente **outbound**: a
plataforma chama endpoints dos clientes e não recebe webhooks deles
`[FECHADO: EV-TR-002-A]`.

Hoje o datasource é MySQL, `OrderService.changeStatus` altera pedido e histórico dentro
de `PrismaClient.$transaction`, e não publica evento algum
`[CÓDIGO: EV-CODE-001-A, EV-CODE-002-A, EV-CODE-002-B, EV-CODE-002-D]`. O schema também
não contém configuração de endpoint, outbox, histórico de tentativas ou DLQ
`[CÓDIGO: EV-CODE-001-D]`. A outbox no MySQL, escrita na mesma transação da mudança e do
histórico, elimina a janela entre confirmar o negócio e registrar a intenção de entrega;
a chamada HTTP fica fora da transação `[FECHADO: EV-TR-003-B, EV-TR-003-C]`
([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

A entrega aceita duplicatas e exige deduplicação do consumidor por `X-Event-Id`
`[FECHADO: EV-TR-010-A, EV-TR-010-B, EV-TR-010-C]`. Retry limitado, DLQ separada,
assinatura por endpoint e observabilidade tornam essa garantia operável sem acoplar a
API à disponibilidade dos clientes.

## Objetivos técnicos

1. Gravar mudança de status, histórico e unidades de outbox elegíveis no mesmo commit;
   qualquer falha de outbox causa rollback conjunto `[FECHADO: EV-TR-003-C,
   EV-TR-017-A]`.
2. Iniciar a primeira tentativa em menos de dez segundos como referência de negócio,
   usando worker separado, polling a cada dois segundos e operação inicial single-worker
   `[FECHADO: EV-TR-001-B, EV-TR-004-A, EV-TR-004-B, EV-TR-004-C]`.
3. Entregar de forma at-least-once, com retry, DLQ e replay administrativo auditado
   `[FECHADO: EV-TR-007-A, EV-TR-007-C, EV-TR-010-A, EV-TR-015-A, EV-TR-015-B]`.
4. Oferecer API autenticada para gerenciar configurações e consultar as últimas 100
   tentativas, sem misturar essa API com a chamada outbound
   `[FECHADO: EV-TR-012-A, EV-TR-012-C, EV-TR-014-A]`.
5. Proteger o corpo outbound com HTTPS e HMAC-SHA256 por endpoint, suportando rotação e
   limite de payload sem truncamento `[FECHADO: EV-TR-008-A, EV-TR-008-B,
   EV-TR-008-C, EV-TR-009-A, EV-TR-009-B]`.
6. Reutilizar controller, service, repository, routes, Zod, Prisma, `AppError`,
   middleware de erro e Pino existentes `[FECHADO: EV-TR-011-A a EV-TR-011-H]`.

## Escopo e exclusões

### Incluído nesta fase

- Mudanças realizadas por `OrderService.changeStatus`; a criação inicial do pedido em
  `PENDING` não entra no gatilho proposto porque o ponto fechado de integração é
  `changeStatus` `[FECHADO: EV-TR-017-A; CÓDIGO: EV-CODE-002-A]`.
- Cadastro, listagem, alteração, remoção lógica e rotação de secret de endpoints por API
  autenticada. `customer_id` vem de body ou path, nunca é inferido do JWT
  `[FECHADO: EV-TR-012-A a EV-TR-012-D]`.
- Filtro pela lista de status do endpoint, snapshot do payload, fan-out, entrega
  assíncrona, retry, DLQ, replay e histórico das últimas 100 tentativas
  `[FECHADO: EV-TR-013-A, EV-TR-014-A, EV-TR-020-B]`.
- Métricas e logs operacionais. Tracing aparece somente como proposta futura porque não
  existe dependência atual `[QUESTÃO_ABERTA: EV-AMB-007]`.
- Dois dias úteis reservados para revisão de segurança antes do deploy
  `[FECHADO: EV-TR-019-B]`.

### Excluído ou não garantido

- Receber webhooks, chamada HTTP síncrona na transação, Redis/broker, trigger MySQL como
  notificador e exactly-once `[DESCARTADO: EV-TR-002-A, EV-TR-003-A, EV-TR-003-D,
  EV-TR-004-D, EV-TR-010-D]`.
- Dashboard visual e aviso por e-mail nesta fase `[DESCARTADO/ADIADO: EV-TR-016-A,
  EV-TR-016-B]`.
- Rate limiting e política de arquivamento/retenção; ambos permanecem fora do contrato
  atual `[QUESTÃO_ABERTA/ADIADO: EV-TR-016-C, EV-TR-016-D]`.
- Ordering global, escala multi-worker e exactly-once. A intenção é ordering por pedido
  no single-worker, sujeita à questão de backoff detalhada adiante
  `[FECHADO: EV-TR-005-A, EV-TR-005-B; QUESTÃO_ABERTA: EV-AMB-001]`.

## Fluxos detalhados

### Modelo de dados — PROPOSTA_DERIVADA, EM REVISÃO

`[PROPOSTA_DERIVADA: EV-PROP-FANOUT-001 a EV-PROP-FANOUT-006; FDD-PROP-DATA-001]`.
Nenhum dos modelos abaixo existe hoje `[CÓDIGO: EV-CODE-001-D]`. Os nomes, campos,
tipos, relações, índices e estados formam uma proposta completa para implementação e
precisam ser aprovados junto com a cardinalidade.

#### `WebhookEndpoint`

| Campo proposto | Tipo/semântica proposta |
| --- | --- |
| `id` | `String @id @default(uuid()) @db.Char(36)`; enviado em `X-Webhook-Id`. |
| `customerId` | FK obrigatória para `Customer.id`; o cliente vem do path/body, não do JWT. |
| `url` | `VarChar(2048)`, normalizada e validada como HTTPS. |
| `subscribedStatuses` | `Json` com conjunto não vazio e sem duplicatas de valores de `OrderStatus`. |
| `active` | `Boolean @default(true)`; somente ativos participam do filtro. |
| `secretEncrypted` | Material da secret atual cifrado em repouso; nunca hash irreversível, pois o worker precisa assinar. |
| `secretVersion` | Inteiro monotônico associado à secret atual. |
| `previousSecretEncrypted` | Secret anterior cifrada, opcional durante o grace period. |
| `previousSecretVersion` | Versão anterior, opcional. |
| `previousSecretExpiresAt` | Fim do período de convivência de 24 horas. |
| `createdAt`, `updatedAt`, `deletedAt` | Auditoria e remoção lógica; `deletedAt` é opcional. |

Índices propostos: `@@index([customerId, active])` e `@@index([deletedAt])`. Não se
propõe unicidade de URL porque a cardinalidade de endpoints por cliente não foi fechada.
A geração candidata usa 32 bytes aleatórios e retorno base64url apenas na criação ou
rotação. Cifra em repouso, fonte da chave mestra e política de rotação dessa chave são
`QUESTÃO_ABERTA: FDD-OPEN-SECRET-STORAGE-001`; não devem ser substituídas por texto puro.

#### `WebhookOutbox`

Cada linha é a unidade de trabalho de **um endpoint**, não o evento lógico. Isso separa
o `eventId` compartilhado da identidade `id` por endpoint
`[PROPOSTA_DERIVADA: EV-PROP-FANOUT-001 a EV-PROP-FANOUT-004]`.

| Campo proposto | Tipo/semântica proposta |
| --- | --- |
| `id` | UUID da unidade de entrega por endpoint; permanece estável em retry e replay. |
| `eventId` | UUID lógico da mudança, compartilhado por todas as linhas do fan-out e usado em `X-Event-Id`. |
| `webhookEndpointId` | FK para `WebhookEndpoint`; compõe unicidade com `eventId`. |
| `customerId`, `orderId` | UUIDs denormalizados para filtro, auditoria e head-of-line. |
| `eventType` | Candidato fixo `order.status_changed`. |
| `payload` | Snapshot JSON imutável renderizado na inserção. |
| `endpointUrl` | Snapshot da URL usada pela unidade; evita que PATCH mude retroativamente o destino. |
| `secretVersion` | Versão candidata para resolver a secret na assinatura; depende da decisão de grace period. |
| `status` | `PENDING`, `PROCESSING`, `RETRY_WAIT`, `SUCCEEDED`, `DEAD_LETTERED` ou `CANCELLED`. |
| `attemptInCycle` | Chamadas já iniciadas no ciclo inicial ou no ciclo de replay. |
| `replayCycle` | Zero no fluxo inicial; incrementado em cada replay aceito. |
| `nextAttemptAt` | Instante de elegibilidade para tentativa inicial/retry. |
| `leaseOwner`, `leaseExpiresAt` | Claim recuperável pelo worker; ambos opcionais fora de `PROCESSING`. |
| `lastErrorCode`, `lastErrorMessage` | Último diagnóstico sanitizado, sem secret/assinatura. |
| `occurredAt`, `createdAt`, `updatedAt`, `completedAt` | Ordenação, lag e auditoria. |

Restrições/índices propostos: `@@unique([eventId, webhookEndpointId])`,
`@@index([status, nextAttemptAt, createdAt])`,
`@@index([orderId, occurredAt, id])`, `@@index([leaseExpiresAt])` e
`@@index([webhookEndpointId, createdAt])`. Os IDs da outbox são UUIDs
`[FECHADO: EV-TR-020-A]`.

#### `WebhookDelivery`

Uma linha representa **uma chamada HTTP iniciada**. Ela é o histórico de tentativas; a
unidade estável continua sendo `WebhookOutbox.id`.

| Campo proposto | Tipo/semântica proposta |
| --- | --- |
| `id` | UUID novo por chamada HTTP. |
| `outboxId` | FK para a unidade estável da outbox. |
| `eventId`, `webhookEndpointId` | IDs denormalizados para correlação. |
| `attemptNumber` | Sequência global crescente dentro da unidade, inclusive após replay. |
| `attemptInCycle`, `replayCycle` | Posição na agenda e ciclo de replay. |
| `startedAt`, `finishedAt`, `durationMs` | Tempos da chamada; `finishedAt` pode ser nulo após crash. |
| `outcome` | `SUCCEEDED`, `RETRYABLE_FAILURE`, `PERMANENT_FAILURE` ou `UNKNOWN`. |
| `httpStatus` | Status recebido, se houver. |
| `responseBody`, `responseTruncated` | Resposta sanitizada; proposta de armazenar no máximo 16 KiB e marcar truncamento. |
| `errorCode`, `errorMessage` | Classificação interna e mensagem sanitizada para falhas HTTP/rede. |

Restrições/índices propostos: `@@unique([outboxId, attemptNumber])` e
`@@index([webhookEndpointId, startedAt])`. O endpoint de histórico junta esta tabela à
outbox para expor o snapshot de payload exigido, sem duplicá-lo
`[FECHADO: EV-TR-014-A]`. O limite de 16 KiB para resposta é
`PROPOSTA_DERIVADA: FDD-PROP-RESPONSE-LIMIT-001 — EM REVISÃO`; ele não altera a regra de
não truncar o payload outbound.

#### `WebhookDeadLetter`

| Campo proposto | Tipo/semântica proposta |
| --- | --- |
| `id` | UUID usado pelo endpoint administrativo de replay. |
| `outboxId`, `eventId`, `webhookEndpointId` | Identidades preservadas. |
| `replayCycle` | Ciclo que terminou em DLQ; único com `outboxId`. |
| `payload` | Cópia do snapshot que falhou. |
| `reasonCode`, `reasonDetail`, `failedAt` | Motivo sanitizado e timestamp exigidos. |
| `lastDeliveryId` | Última tentativa que encerrou o ciclo. |
| `status` | `OPEN` ou `REPLAYED`. |
| `replayedAt`, `replayedByUserId`, `replayReason` | Auditoria do replay; opcionais enquanto `OPEN`. |

Restrições/índices propostos: `@@unique([outboxId, replayCycle])`,
`@@index([status, failedAt])` e FK de `replayedByUserId` para `User.id`. A tabela separada,
o payload, o motivo e o timestamp são requisitos fechados
`[FECHADO: EV-TR-007-A, EV-TR-007-B]`; a forma exata é proposta.

#### Transições de estado propostas

| Origem | Destino | Gatilho |
| --- | --- | --- |
| `PENDING` / `RETRY_WAIT` | `PROCESSING` | Claim atômico cria/renova lease. |
| `PROCESSING` | `SUCCEEDED` | Resultado classificado como sucesso. |
| `PROCESSING` | `RETRY_WAIT` | Falha retentável com retry disponível. |
| `PROCESSING` | `DEAD_LETTERED` | Falha permanente ou política esgotada, com criação atômica da DLQ. |
| `PROCESSING` com lease expirado | `PENDING` | Recuperação após crash; tentativa incompleta vira `UNKNOWN`. |
| `PENDING` / `RETRY_WAIT` | `CANCELLED` | Remoção lógica do endpoint, conforme proposta de DELETE. |
| `DEAD_LETTERED` | `PENDING` | Replay ADMIN auditado; novo ciclo, mesmos `eventId` e `outboxId`. |

Toda a tabela é `PROPOSTA_DERIVADA: FDD-PROP-STATES-001 — EM REVISÃO`.

### Fluxo 1 — mudança de status, filtro, snapshot e fan-out

1. `OrderService.changeStatus` mantém a validação por `canTransition` e os efeitos de
   estoque definidos por `shouldDebitStock`/`shouldReplenishStock`
   `[CÓDIGO: EV-CODE-003-A a EV-CODE-003-C]`.
2. Dentro do callback existente de `$transaction`, ele atualiza pedido e histórico e
   consulta endpoints ativos do `order.customerId` inscritos no novo `toStatus`.
3. Sem endpoint inscrito, nenhuma linha de outbox é inserida
   `[FECHADO: EV-TR-013-A]`.
4. Com um ou mais endpoints, gera-se um `eventId` lógico e um timestamp, e o payload é
   renderizado uma única vez a partir do estado confirmado dentro da transação. O
   snapshot não é reconstruído durante retry `[FECHADO: EV-TR-020-B;
   PROPOSTA_DERIVADA: EV-PROP-FANOUT-001]`.
5. Mede-se o corpo serializado em UTF-8. A proposta interpreta 64 KB como 65.536 bytes;
   acima disso, lança `WEBHOOK_PAYLOAD_TOO_LARGE`, não trunca e provoca rollback de
   pedido, histórico, estoque e outbox `[FECHADO: EV-TR-009-B, EV-TR-017-A;
   PROPOSTA_DERIVADA: FDD-PROP-64K-001 — EM REVISÃO]`.
6. Para cada endpoint inscrito, cria-se uma `WebhookOutbox` com `id` próprio, o mesmo
   `eventId`, payload idêntico e snapshots de URL/configuração. Uma falha em qualquer
   inserção aborta todo o fan-out e a transação
   `[PROPOSTA_DERIVADA: EV-PROP-FANOUT-002, EV-PROP-FANOUT-003]`.
7. Somente depois do commit o worker pode observar as linhas. Nenhuma chamada HTTP é
   feita pelo request de mudança de status `[FECHADO: EV-TR-003-A, EV-TR-003-C]`.

O helper candidato
`publishWebhookEvent(tx, { order, fromStatus, toStatus, changedAt, reason })` deve
receber o `Prisma.TransactionClient`; nome e assinatura são
`PROPOSTA_DERIVADA: EV-TR-017-B — EM REVISÃO`.

### Fluxo 2 — polling, claim e entrega

1. Um processo separado da API, com `PrismaClient` próprio, inicia polling a cada dois
   segundos e procura o trabalho elegível mais antigo
   `[FECHADO: EV-TR-004-A, EV-TR-004-B]`.
2. O claim candidato ocorre em transação e muda exatamente uma linha para `PROCESSING`
   somente se ela continuar elegível. Registra `leaseOwner` e `leaseExpiresAt`; os
   detalhes estão em Estratégias de resiliência.
3. Antes da rede, o worker cria `WebhookDelivery` com novo UUID e incrementa
   `attemptInCycle`. `eventId` e `WebhookOutbox.id` permanecem estáveis em retry
   `[PROPOSTA_DERIVADA: EV-PROP-FANOUT-004]`.
4. O worker serializa uma vez o snapshot, resolve a secret, calcula a assinatura sobre
   os mesmos bytes que enviará e executa POST HTTPS com timeout total candidato de dez
   segundos. Bytes/formato e operação da secret continuam em revisão.
5. O resultado é classificado pela matriz outbound proposta. A finalização atualiza a
   tentativa e a outbox em uma transação curta: sucesso, agendamento de retry ou criação
   de DLQ.
6. Uma falha ao persistir o resultado não autoriza assumir sucesso. Após o lease, a
   unidade volta ao fluxo e pode gerar duplicata, preservando `X-Event-Id`, como exige a
   semântica at-least-once `[FECHADO: EV-TR-010-A a EV-TR-010-C]`.

### Fluxo 3 — DLQ e replay auditado

1. Falha permanente proposta ou esgotamento da agenda cria `WebhookDeadLetter` e move a
   outbox para `DEAD_LETTERED` na mesma transação
   `[FECHADO: EV-TR-007-A, EV-TR-007-B]`.
2. Um usuário `ADMIN` solicita replay, informando o ID da DLQ. O serviço valida que ela
   está `OPEN`, registra `req.user.id`, instante e motivo e a marca `REPLAYED`
   `[FECHADO: EV-TR-015-A, EV-TR-015-B]`.
3. Na mesma transação, a outbox volta a `PENDING`, incrementa `replayCycle`, zera
   `attemptInCycle`, limpa lease e recebe `nextAttemptAt = now`. O envio permanece
   assíncrono `[FECHADO: EV-TR-007-C; PROPOSTA_DERIVADA: EV-PROP-FANOUT-005]`.
4. O replay preserva `eventId` e `outboxId`, mas a próxima chamada cria novo
   `WebhookDelivery.id`. Se falhar novamente até o limite, cria outra linha de DLQ para
   o novo ciclo.

## Contratos públicos

### Separação dos contratos

- A **API pública de gerenciamento** abaixo recebe chamadas dos usuários da aplicação,
  usa Bearer JWT e responde com o envelope de erro já usado pela API. As sete rotas
  completas, schemas e respostas são `PROPOSTA_DERIVADA: FDD-PROP-API-001 — EM REVISÃO`;
  o requisito de autenticação e a role de replay são fechados.
- A **chamada outbound** é um POST feito pelo worker diretamente à URL HTTPS cadastrada.
  Ela não é uma rota `/api/v1`, não usa o JWT da API e autentica a origem pelo HMAC.

Em todas as rotas autenticadas, ausência/token inválido retorna `401 UNAUTHORIZED` pelo
`authenticate` atual `[CÓDIGO: EV-CODE-005-A]`. O envelope contém `error.code` string,
`error.message` string e `error.details` opcional como objeto ou lista
`[CÓDIGO: EV-CODE-006-B a EV-CODE-006-D]`.

Nesta fase, “qualquer role autenticada” significa autorização apenas por autenticação
para o CRUD; o JWT atual contém usuário, e-mail e role, não `customerId`
`[FECHADO: EV-TR-012-D, EV-TR-015-C; CÓDIGO: EV-CODE-005-A]`. Não há regra fechada de
ownership por customer a ser inferida silenciosamente.

### POST /api/v1/customers/:customerId/webhooks — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT; `ADMIN` ou `OPERATOR`, sem inferir `customerId` do token
  `[FECHADO: EV-TR-012-C, EV-TR-012-D, EV-TR-015-C]`.
- **Request:** path `customerId` UUID; body
  `{"url":"https://cliente.example/webhooks/orders","statuses":["PAID","SHIPPED"]}`.
  `statuses` é não vazio, sem duplicatas e limitado ao `OrderStatus` existente.
- **Response `201`:** configuração sanitizada com `id`, `customerId`, `url`, `statuses`,
  `active`, `createdAt` e `secret`. A secret é gerada pelo sistema e exibida em texto
  claro somente nesta resposta `[FECHADO: EV-TR-012-A, EV-TR-012-B]`.
- **Status codes:** `201`; `400 WEBHOOK_INVALID_URL` ou `WEBHOOK_INVALID_STATUS`;
  `401`; `404 NOT_FOUND` para customer inexistente; `500 WEBHOOK_SECRET_GENERATION_FAILED`.
- **Semântica:** persiste endpoint e secret cifrada atomicamente; nunca retorna os
  campos cifrados. URL HTTP é recusada `[FECHADO: EV-TR-009-A]`.

### GET /api/v1/customers/:customerId/webhooks — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT; `ADMIN` ou `OPERATOR`
  `[FECHADO: EV-TR-012-C, EV-TR-015-C]`.
- **Request:** path `customerId` UUID; query candidata `page` (default 1) e `pageSize`
  (default 20, máximo 100); sem body.
- **Response `200`:** objeto com `data`, uma lista de configurações sanitizadas contendo
  `id`, `customerId`, `url`, `statuses`, `active`, `createdAt` e `updatedAt`, e
  `pagination` com `page`, `pageSize`, `total` e `totalPages`; sem secret atual/anterior
  ou material cifrado.
- **Status codes:** `200`; `400 VALIDATION_ERROR`; `401`; `404 NOT_FOUND` para customer
  inexistente.
- **Semântica:** lista configurações não removidas daquele customer, ordenadas por
  `createdAt desc`; `customerId` vem do path.

### PATCH /api/v1/webhooks/:id — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT; `ADMIN` ou `OPERATOR`
  `[FECHADO: EV-TR-012-C, EV-TR-015-C]`.
- **Request:** path `id` UUID; body parcial com pelo menos um de `url`, `statuses` ou
  `active`, usando as mesmas validações da criação.
- **Response `200`:** configuração sanitizada atualizada, sem secrets.
- **Status codes:** `200`; `400 WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_STATUS` ou
  `VALIDATION_ERROR`; `401`; `404 WEBHOOK_ENDPOINT_NOT_FOUND`; `409
  WEBHOOK_ENDPOINT_DISABLED` quando a transição solicitada não for permitida.
- **Semântica:** alteração afeta apenas novos eventos. URL e lista já copiadas para
  outboxes existentes não mudam retroativamente
  `[PROPOSTA_DERIVADA: FDD-PROP-ENDPOINT-SNAPSHOT-001 — EM REVISÃO]`.

### DELETE /api/v1/webhooks/:id — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT; `ADMIN` ou `OPERATOR`
  `[FECHADO: EV-TR-012-C, EV-TR-015-C]`.
- **Request:** path `id` UUID; sem body.
- **Response `204`:** sem corpo.
- **Status codes:** `204`; `400 VALIDATION_ERROR`; `401`; `404
  WEBHOOK_ENDPOINT_NOT_FOUND`; `409 WEBHOOK_ENDPOINT_DELETE_CONFLICT` se houver estado
  que não possa ser cancelado com segurança.
- **Semântica proposta:** remoção lógica (`active=false`, `deletedAt=now`) impede novos
  eventos e cancela outboxes `PENDING`/`RETRY_WAIT` na mesma transação. Uma chamada já
  em `PROCESSING` pode terminar; o resultado é auditado. Esta política de cancelar
  pendências é `PROPOSTA_DERIVADA: FDD-PROP-DELETE-001 — EM REVISÃO`.

### POST /api/v1/webhooks/:id/rotate-secret — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT; `ADMIN` ou `OPERATOR`
  `[FECHADO: EV-TR-012-C, EV-TR-015-C]`.
- **Request:** path `id` UUID; sem body.
- **Response `200`:** objeto com `id` UUID, `secret` opaca exibida uma única vez e
  `previousSecretValidUntil` em ISO 8601; não retorna a secret anterior.
- **Status codes:** `200`; `400 VALIDATION_ERROR`; `401`; `404
  WEBHOOK_ENDPOINT_NOT_FOUND`; `409 WEBHOOK_SECRET_ROTATION_CONFLICT`; `500
  WEBHOOK_SECRET_GENERATION_FAILED`.
- **Semântica fechada:** secret por endpoint, rotacionável, com a anterior válida em
  paralelo por 24 horas `[FECHADO: EV-TR-008-B, EV-TR-008-C]`.
- **Semântica ainda aberta:** qual secret assina cada entrega durante o período, como o
  consumidor distingue versões e se uma segunda rotação durante as 24 horas é aceita
  `[QUESTÃO_ABERTA: EV-AMB-006]`. A proposta inicial é rejeitar nova rotação enquanto
  houver versão anterior ativa, mas isso não é contrato aprovado.

### GET /api/v1/webhooks/:id/deliveries — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT; `ADMIN` ou `OPERATOR`
  `[FECHADO: EV-TR-012-C, EV-TR-015-C]`.
- **Request:** path `id` UUID; sem body. O limite é fixo em 100 nesta fase.
- **Response `200`:** objeto com `data` em `startedAt desc`, contendo até 100 entradas.
  Cada entrada contém `deliveryId`, `outboxId`, `eventId`, `attemptNumber`,
  `replayCycle`, `outcome`, `payload`, `httpStatus`, `responseBody`,
  `responseTruncated`, `durationMs`, `errorCode`, `startedAt` e `finishedAt`. Nunca
  inclui secret, assinatura ou headers sensíveis.
- **Status codes:** `200`; `400 VALIDATION_ERROR`; `401`; `404
  WEBHOOK_ENDPOINT_NOT_FOUND`.
- **Semântica:** inclui sucesso e falha, payload, response e duração conforme requisito
  `[FECHADO: EV-TR-014-A]`. Uma tentativa incompleta recuperada após crash aparece como
  `UNKNOWN` pela proposta de lease.

### POST /api/v1/admin/webhooks/dead-letter/:id/replay — PROPOSTA_DERIVADA

- **Autenticação:** Bearer JWT seguido de `requireRole('ADMIN')`; `OPERATOR` recebe 403
  `[FECHADO: EV-TR-015-A; CÓDIGO: EV-CODE-005-B]`.
- **Request:** path `id` UUID da DLQ; body candidato
  `{"reason":"endpoint do cliente recuperado"}` com 1 a 500 caracteres.
- **Response `202`:** objeto com `deadLetterId`, `outboxId` e `eventId` UUID,
  `status="PENDING"` e `replayCycle` inteiro incrementado.
- **Status codes:** `202`; `400 VALIDATION_ERROR`; `401`; `403`; `404
  WEBHOOK_DLQ_NOT_FOUND`; `409 WEBHOOK_REPLAY_CONFLICT`; `500 WEBHOOK_REPLAY_FAILED`.
- **Semântica:** registra o ID de `req.user`, preserva identidades e recoloca a unidade
  como pendente, sem chamada HTTP síncrona `[FECHADO: EV-TR-007-C, EV-TR-015-B;
  PROPOSTA_DERIVADA: EV-PROP-FANOUT-005]`. Atualização condicional impede dois replays da
  mesma linha `OPEN`.

### Chamada outbound para o endpoint do cliente

Este contrato é diferente da API de gerenciamento: o destino é
`WebhookEndpoint.url`, o método é `POST`, não há Bearer JWT e não existe rota inbound
equivalente nesta aplicação.

#### Payload JSON

Forma candidata completa:

```json
{
  "event_id": "6f9ad87e-08f4-4eca-9f2d-c5976686f123",
  "type": "order.status_changed",
  "timestamp": "2026-08-15T18:42:31.123Z",
  "customer_id": "7b8179cd-5c8a-48db-946c-e73cd13d853a",
  "data": {
    "order_id": "26a0ae6d-0927-41f8-9a82-906f9e75b48f",
    "order_number": "ORD-000123",
    "from_status": "PENDING",
    "to_status": "PAID",
    "changed_at": "2026-08-15T18:42:31.123Z",
    "reason": "payment confirmed"
  }
}
```

JSON enxuto com `event_id`, tipo, timestamp ISO 8601, dados do pedido e `customer_id`,
sem items, é fechado `[FECHADO: EV-TR-018-B]`. Nomes exatos além dos campos fechados,
`order_number`, `from_status`, `to_status`, `changed_at`, `reason`, nulabilidade e o
valor `order.status_changed` são `PROPOSTA_DERIVADA: FDD-PROP-PAYLOAD-001 — EM REVISÃO`.
O snapshot é imutável `[FECHADO: EV-TR-020-B]`.

O limite fechado é 64 KB, sem truncamento `[FECHADO: EV-TR-009-B]`. A interpretação
candidata é `Buffer.byteLength(serializedBody, 'utf8') <= 65_536`; mede-se o corpo final
antes de persistir a outbox e novamente antes de enviar como defesa
`[PROPOSTA_DERIVADA: FDD-PROP-64K-001 — EM REVISÃO]`.

#### Headers

| Header | Valor/semântica |
| --- | --- |
| `X-Event-Id` | UUID lógico presente também em `event_id` `[FECHADO: EV-TR-018-C]`. |
| `X-Signature` | Assinatura HMAC-SHA256 `[FECHADO: EV-TR-018-D]`; formato em aberto. |
| `X-Timestamp` | Timestamp da tentativa `[FECHADO: EV-TR-018-E]`; ISO 8601 UTC é proposta. |
| `X-Webhook-Id` | UUID do endpoint cadastrado `[FECHADO: EV-TR-018-F]`. |
| `Content-Type` | `application/json` `[FECHADO: EV-TR-018-G]`. |

Esses são os quatro headers de webhook definidos, além do `Content-Type`. Propagar
`traceparent` ou qualquer header adicional não está aprovado.

#### HMAC e rotação

- HMAC-SHA256 sobre o corpo e secret exclusiva por endpoint são fechados
  `[FECHADO: EV-TR-008-A, EV-TR-008-B]`.
- Os bytes assinados, codificação e formato de `X-Signature` não estão fechados
  `[QUESTÃO_ABERTA: EV-AMB-005]`.
- **Candidato para revisão:** serializar uma vez; assinar exatamente os bytes UTF-8
  enviados, sem canonicalização posterior; digest em base64; header
  `v1=<base64>`. Isso é `PROPOSTA_DERIVADA: FDD-PROP-HMAC-001 — EM REVISÃO`, não um
  contrato interoperável aprovado.
- A secret anterior permanece válida por 24 horas `[FECHADO: EV-TR-008-C]`, mas qual
  versão assina, como o header identifica a versão e o comportamento de retries que
  atravessam a expiração são `QUESTÃO_ABERTA: EV-AMB-006`. O deploy outbound fica
  bloqueado até uma decisão e vetores de teste aprovados.

#### Transporte, timeout e classificação

- Somente HTTPS; HTTP é rejeitado no cadastro/alteração `[FECHADO: EV-TR-009-A]`.
- Timeout de dez segundos é falha para retry `[FECHADO: EV-TR-018-A]`. Timeout total de
  parede via `AbortController`, incluindo conexão e leitura do corpo, é
  `PROPOSTA_DERIVADA: FDD-PROP-TIMEOUT-001 — EM REVISÃO`.
- Redirects e a classificação de HTTP/rede não foram fechados
  `[QUESTÃO_ABERTA: EV-AMB-003, EV-AMB-004]`. A matriz abaixo é integralmente uma
  `PROPOSTA_DERIVADA: FDD-PROP-CLASSIFICATION-001 — EM REVISÃO`:

| Resultado candidato | Classificação candidata | Efeito candidato |
| --- | --- | --- |
| `200–299` | sucesso | `SUCCEEDED`; não retentar. |
| `300–399` | permanente, sem seguir redirect | DLQ imediata; evita reenviar assinatura/payload a destino não validado. |
| `408`, `425`, `429` | retentável | Agenda retry; `Retry-After` ainda não prevalece sobre a tabela aprovada. |
| `400`, `401`, `403`, `404`, `405`, `410`, `422` | permanente | DLQ imediata. |
| Demais `4xx` | permanente | DLQ imediata. |
| `500–599` | retentável | Agenda retry; DLQ ao esgotar. |
| Timeout de 10 s | retentável | Agenda retry `[FECHADO: EV-TR-018-A]`. |
| DNS temporário (`EAI_AGAIN`), conexão recusada/reset ou indisponível | retentável | Agenda retry. |
| DNS inexistente (`ENOTFOUND`) | permanente | DLQ imediata. |
| Certificado TLS inválido/expirado ou hostname incompatível | permanente | DLQ imediata. |
| Erro TLS transitório não atribuível a certificado | retentável | Agenda retry. |

Redirect automático fica desabilitado na proposta. A validação de HTTPS também deve
decidir proteção SSRF contra loopback, redes privadas, link-local, DNS rebinding e mudança
de destino entre cadastro e envio; essa é `QUESTÃO_ABERTA: FDD-OPEN-SSRF-001` e faz parte
da revisão de segurança.

## Matriz de erros

Os códigos alvo do novo domínio usam `WEBHOOK_` `[FECHADO: EV-TR-011-I]`, mas **não
existem hoje**. `validate` converte `ZodError` em `ValidationError` com
`VALIDATION_ERROR`, e `NotFoundError` usa `NOT_FOUND`
`[CÓDIGO: EV-CODE-006-A a EV-CODE-006-C]`. Portanto, a coluna “atual” não promete um
código específico; ela mostra o comportamento genérico que deverá ser substituído por
classes/erros de domínio quando a matriz alvo for implementada.

| Código futuro proposto | Contexto | HTTP/efeito proposto | Código genérico atual |
| --- | --- | --- | --- |
| `WEBHOOK_INVALID_URL` | URL ausente, inválida ou não HTTPS | 400 | `VALIDATION_ERROR` |
| `WEBHOOK_ENDPOINT_NOT_FOUND` | Configuração inexistente/removida | 404 | `NOT_FOUND` |
| `WEBHOOK_INVALID_STATUS` | Status de inscrição fora de `OrderStatus` | 400 | `VALIDATION_ERROR` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Snapshot UTF-8 excede 64 KB | 422 e rollback de `changeStatus` | `VALIDATION_ERROR` ou erro ainda inexistente |
| `WEBHOOK_SECRET_GENERATION_FAILED` | Falha ao gerar/cifrar secret | 500; nenhuma secret parcial persistida | `INTERNAL_SERVER_ERROR` |
| `WEBHOOK_SECRET_UNAVAILABLE` | Worker não resolve versão/material de assinatura | falha operacional; classificação depende do grace period | `INTERNAL_SERVER_ERROR` |
| `WEBHOOK_SECRET_ROTATION_CONFLICT` | Rotação concorrente ou durante grace ativo, se proposta aprovada | 409 | `CONFLICT` |
| `WEBHOOK_ENDPOINT_DISABLED` | Operação incompatível com endpoint inativo | 409 | `CONFLICT` |
| `WEBHOOK_ENDPOINT_DELETE_CONFLICT` | Remoção não pode cancelar estado com segurança | 409 | `CONFLICT` |
| `WEBHOOK_DELIVERY_FAILED` | Falha outbound genérica já classificada | histórico/log; retry ou DLQ | não existe |
| `WEBHOOK_DELIVERY_TIMEOUT` | Dez segundos excedidos | histórico/log e retry | não existe |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | HTTP fora da classe de sucesso | histórico/log; efeito pela matriz em revisão | não existe |
| `WEBHOOK_DELIVERY_LEASE_EXPIRED` | Crash/lease expirado deixa resultado incerto | tentativa `UNKNOWN`, nova entrega possível | não existe |
| `WEBHOOK_DLQ_WRITE_FAILED` | Falha ao persistir dead letter | mantém/reclama unidade; alerta crítico | não existe |
| `WEBHOOK_DLQ_NOT_FOUND` | ID de dead letter inexistente | 404 | `NOT_FOUND` |
| `WEBHOOK_REPLAY_CONFLICT` | DLQ já reprocessada ou claim concorrente | 409 | `CONFLICT` |
| `WEBHOOK_REPLAY_FAILED` | Transação de replay falha | 500, sem estado parcial | `INTERNAL_SERVER_ERROR` |

Status, classes e mapeamentos desta tabela são
`PROPOSTA_DERIVADA: FDD-PROP-ERRORS-001 — EM REVISÃO`; somente o prefixo é fechado.

## Estratégias de resiliência

### Agenda de tentativas — PROPOSTA_DERIVADA, EM REVISÃO

Há uma contradição não resolvida: a reunião declara **cinco tentativas**, mas também
declara **cinco intervalos** (`1m/5m/30m/2h/12h`). Não é possível aplicar cinco esperas
após a tentativa inicial sem totalizar seis chamadas
`[QUESTÃO_ABERTA: EV-TR-006-A, EV-TR-006-B, EV-TR-006-C]`.

A recomendação inicial preserva todos os intervalos: uma tentativa imediata mais cinco
retries, totalizando seis chamadas `[PROPOSTA_DERIVADA: EV-PROP-RETRY-001 — EM REVISÃO]`.
Cada espera é contada após a falha anterior; os tempos cumulativos abaixo supõem falhas
imediatas e servem para teste verificável.

| Chamada HTTP | Tipo | Espera após a falha anterior | Momento cumulativo mais cedo |
| ---: | --- | ---: | ---: |
| 1 | tentativa inicial | 0 | `T+0` |
| 2 | retry 1 | 1 min | `T+1 min` |
| 3 | retry 2 | 5 min | `T+6 min` |
| 4 | retry 3 | 30 min | `T+36 min` |
| 5 | retry 4 | 2 h | `T+2 h 36 min` |
| 6 | retry 5 | 12 h | `T+14 h 36 min` |
| — | DLQ após nova falha retentável | imediatamente após a chamada 6 | `>= T+14 h 36 min` |

`nextAttemptAt` é calculado a partir do término da tentativa anterior. Uma falha
classificada como permanente pode ir à DLQ antes da sexta chamada, se a matriz de
classificação for aprovada. Até reconciliar a contradição, a tabela não deve ser
codificada como regra definitiva.

### Ordering e head-of-line — PROPOSTA_DERIVADA, EM REVISÃO

O requisito busca ordem por `order_id` no single-worker, mas `createdAt` sozinho não
resolve o evento anterior em backoff `[FECHADO: EV-TR-005-A; QUESTÃO_ABERTA:
EV-AMB-001]`. A proposta é head-of-line por pedido
`[PROPOSTA_DERIVADA: EV-PROP-ORDER-001]`:

- uma unidade é elegível somente se não existir para o mesmo `orderId` uma unidade mais
  antiga em `PENDING`, `PROCESSING` ou `RETRY_WAIT`;
- a ordem estável usa `(occurredAt, id)`; outro pedido continua elegível e não sofre
  bloqueio global;
- se o evento A falhar e aguardar 30 minutos, o evento B posterior do mesmo pedido fica
  bloqueado até A chegar a `SUCCEEDED` ou `DEAD_LETTERED`;
- replay de A depois de DLQ pode ocorrer após B já ter sido entregue; replay não promete
  restaurar a ordem histórica.

O custo é aumentar a latência de eventos posteriores do mesmo pedido. Sem aprovação,
ordering durante backoff permanece questão aberta e não pode ser anunciado aos clientes.

### Claim, lease e crash recovery — PROPOSTA_DERIVADA, EM REVISÃO

Lease não foi fechado na reunião `[PROPOSTA_DERIVADA: EV-PROP-FANOUT-006]`. A proposta
inicial usa lease de **30 segundos**, superior ao timeout outbound de dez segundos, com
`leaseOwner` único por processo:

1. Em transação curta, selecionar a unidade mais antiga elegível e executar update
   condicional para `PROCESSING`, `leaseOwner=<worker-id>` e
   `leaseExpiresAt=now+30s`. Em MySQL 8, `SELECT ... FOR UPDATE SKIP LOCKED` é candidato;
   a consulta Prisma/raw exata deve ter teste de concorrência.
2. Criar a tentativa antes do envio. Enquanto a chamada estiver ativa, somente o dono
   do lease pode finalizá-la.
3. Na inicialização e em cada poll, reclaimar `PROCESSING` com lease expirado: marcar a
   tentativa sem `finishedAt` como `UNKNOWN`/`WEBHOOK_DELIVERY_LEASE_EXPIRED`, limpar o
   lease e tornar a outbox `PENDING`.
4. Se o crash ocorreu depois de o cliente processar, a nova chamada é duplicata. Ela
   conserva `eventId` e `outboxId`; somente `WebhookDelivery.id` muda.

O valor de 30 segundos, renovação de lease, SQL de claim e relógio autoritativo (banco ou
processo) são `PROPOSTA_DERIVADA: FDD-PROP-LEASE-001 — EM REVISÃO`. Usar tempo do banco
é a recomendação para reduzir skew.

### Isolamento e idempotência

- O HTTP nunca participa da transação de pedido `[FECHADO: EV-TR-003-A,
  EV-TR-003-C]`.
- `@@unique([eventId, webhookEndpointId])` evita fan-out duplicado na mesma transação;
  claim condicional evita dois donos simultâneos na proposta single-worker.
- At-least-once continua permitindo duplicatas; o consumidor deve persistir
  `X-Event-Id` e tornar seus efeitos idempotentes `[FECHADO: EV-TR-010-A a
  EV-TR-010-C]`.
- Falha de escrita na DLQ ou de finalização não apaga a outbox. Lease expirado torna a
  unidade novamente observável.

## Observabilidade

### Métricas propostas

Pino não fornece métricas; não há cliente de métricas no `package.json`. Nomes,
instrumentação e backend abaixo são `PROPOSTA_DERIVADA: FDD-PROP-METRICS-001 — EM
REVISÃO`, enquanto a necessidade de medir lag, tentativas, duração, respostas, retries,
DLQ e leases é requisito deste FDD.

| Métrica proposta | Tipo | Uso |
| --- | --- | --- |
| `webhook_first_attempt_lag_seconds` | histograma | Tempo entre commit/`createdAt` e início da primeira chamada; exclui retries. |
| `webhook_outbox_oldest_pending_age_seconds` | gauge | Idade da unidade elegível mais antiga. |
| `webhook_delivery_attempts_total{outcome,status_class}` | counter | Volume de sucesso/falha por classe, sem IDs como labels. |
| `webhook_delivery_duration_seconds{outcome}` | histograma | Duração das chamadas outbound. |
| `webhook_http_responses_total{status_class}` | counter | Respostas `2xx` a `5xx` e ausência de resposta. |
| `webhook_retries_scheduled_total{reason}` | counter | Retries por timeout, HTTP, DNS, TLS ou conexão. |
| `webhook_dlq_depth` | gauge | Linhas `OPEN` na DLQ. |
| `webhook_dlq_entries_total{reason}` | counter | Entradas por motivo sanitizado. |
| `webhook_replays_total{outcome}` | counter | Replays aceitos, em conflito ou falhos. |
| `webhook_leases_active` | gauge | Claims em `PROCESSING` com lease válido. |
| `webhook_lease_events_total{action}` | counter | Claims, releases, expirações e reclaims. |
| `webhook_poll_cycles_total{result}` | counter | Polls vazios, claims e erros. |

A métrica de referência para os dez segundos mede commit até início da primeira
tentativa, sem inventar percentil/SLO não decidido
`[PROPOSTA_DERIVADA: EV-PROP-SLI-001]`. Não usar `eventId`, `webhookId`, `customerId`,
`orderId` ou URL como labels, para evitar alta cardinalidade.

### Logs estruturados

Usar Pino, já existente `[FECHADO: EV-TR-011-G; CÓDIGO: EV-CODE-007-A]`, com eventos
`webhook_outbox_created`, `webhook_claimed`, `webhook_attempt_started`,
`webhook_attempt_finished`, `webhook_retry_scheduled`, `webhook_dead_lettered`,
`webhook_replayed`, `webhook_lease_expired` e `webhook_worker_shutdown`.

Campos de correlação: `eventId`, `webhookId`, `outboxId`, `deliveryId`, `orderId`,
`replayCycle`, `attemptNumber`, `durationMs`, `httpStatus`, `errorCode` e `requestId`
quando originado na API. Não registrar corpo completo por padrão.

A redaction atual cobre authorization, cookie, password e token, mas não secret ou
`X-Signature` `[CÓDIGO: EV-CODE-007-B, EV-CODE-007-C]`. Antes do deploy, ampliar para
`*.secret`, `*.secretEncrypted`, `*.previousSecretEncrypted`, `*.signature`,
`req.headers.x-signature` e `headers.x-signature`. Secrets, assinatura, chave mestra e
headers completos nunca podem aparecer em logs, erros, métricas ou tracing.

### Tracing — PROPOSTA FUTURA

Tracing não é capacidade atual: não há dependência correspondente nem entrypoint do
worker `[QUESTÃO_ABERTA: EV-AMB-007; CÓDIGO: EV-CODE-008-C]`. Uma integração futura com
OpenTelemetry pode criar spans para transação de status/outbox, poll/claim, tentativa
HTTP e replay. SDK/exporter, sampling, propagação de contexto e envio de `traceparent`
são `QUESTÃO_ABERTA: FDD-OPEN-TRACING-001`. Nenhum critério desta fase deve pressupor
traces disponíveis; métricas, logs e IDs persistidos fazem a correlação inicial.

## Dependências e compatibilidade

### Existentes e reutilizadas

- Node.js `>=20`, Express 4, Prisma 5/MySQL, Zod 3 e Pino 9 estão no
  [`package.json`](../package.json#L1-L52) `[CÓDIGO: EV-CODE-001-A,
  EV-CODE-007-A]`.
- HMAC, geração aleatória, cifragem candidata e HTTP podem usar APIs nativas de Node
  (`node:crypto`, `fetch`, `AbortController`), sem afirmar que já há cliente outbound.
- UUID é o padrão dos modelos atuais e a outbox exige UUID
  `[CÓDIGO: EV-CODE-001-B; FECHADO: EV-TR-020-A]`.

### Adições futuras necessárias

- Migration Prisma aditiva para enums/modelos/relações e índices propostos.
- Novo módulo com controller, service, repository, routes e schemas Zod, seguindo
  [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md).
- Entrypoint e scripts separados do worker; eles não existem hoje
  `[CÓDIGO: EV-CODE-008-B, EV-CODE-008-C]`.
- Configuração segura da chave de cifragem das secrets. O nome da variável, provedor de
  segredo e procedimento de rotação são questão aberta e bloqueiam produção.
- Biblioteca/exportador de métricas a selecionar. Tracing permanece proposta futura,
  não dependência desta implementação inicial.

### Compatibilidade e rollout propostos

1. Aplicar migration aditiva antes de subir código que grave outbox; confirmar suporte
   do MySQL de produção à estratégia de claim escolhida.
2. Subir API com módulo/rotas e publicação transacional, mas manter worker desabilitado
   até secrets, classificação e HMAC serem aprovados e testados.
3. Subir exatamente um worker e observar lag/carga do MySQL antes de escala
   `[FECHADO: EV-TR-004-C]`.
4. Executar revisão de segurança por dois dias úteis antes do deploy
   `[FECHADO: EV-TR-019-B]`.
5. Mudanças são aditivas para rotas existentes; nenhum endpoint atual é removido. Essa
   compatibilidade deve ser confirmada pela suíte de regressão.

## Integração com o sistema existente

Os símbolos abaixo são atuais; todo símbolo de webhook mencionado como novo é proposta,
não capacidade existente.

| Caminho real | Símbolo/comportamento atual | Integração futura proposta |
| --- | --- | --- |
| [`prisma/schema.prisma`](../prisma/schema.prisma#L1) | Datasource MySQL; UUID nos principais modelos; `Order` tem `customerId`, `status` e histórico; não há modelos de webhook `[CÓDIGO: EV-CODE-001-A a EV-CODE-001-D]`. | Adicionar os quatro modelos, enums, relações em `Customer`/`User` e índices desta proposta por migration. |
| [`src/modules/orders/order.service.ts`](../src/modules/orders/order.service.ts#L126) | `OrderService.changeStatus` usa `$transaction`, atualiza estoque/status/histórico e não publica outbox `[CÓDIGO: EV-CODE-002-A a EV-CODE-002-D]`. | Injetar publisher/repository e chamar `publishWebhookEvent(tx, ...)` dentro do callback, depois do estado/histórico e antes do retorno; sem HTTP. Nome é proposta. |
| [`src/modules/orders/order.status.ts`](../src/modules/orders/order.status.ts#L3) | `canTransition`, `shouldDebitStock` e `shouldReplenishStock` definem transição/estoque `[CÓDIGO: EV-CODE-003-A a EV-CODE-003-C]`. | Reutilizar `OrderStatus` para validar inscrições; não duplicar enum de status. |
| [`src/app.ts`](../src/app.ts#L26) | `buildControllers` instancia repositories/services/controllers explicitamente e `buildApp` monta `/api/v1` `[CÓDIGO: EV-CODE-004-A, EV-CODE-004-B]`. | Compor `WebhookRepository`, `WebhookService`, publisher e `WebhookController`; passar dependências ao `OrderService`. Esses símbolos ainda não existem. |
| [`src/routes/index.ts`](../src/routes/index.ts#L13) | `Controllers` e `buildApiRouter` registram módulos explicitamente `[CÓDIGO: EV-CODE-004-B]`. | Acrescentar controller/router de webhook e montar as rotas propostas, incluindo o ramo admin. |
| [`src/middlewares/auth.middleware.ts`](../src/middlewares/auth.middleware.ts#L27) | `authenticate` popula `req.user`; `requireRole` suporta `ADMIN`/`OPERATOR` `[CÓDIGO: EV-CODE-005-A, EV-CODE-005-B]`. | Aplicar `authenticate` às sete rotas e `requireRole('ADMIN')` somente ao replay. `customerId` continua no path/body. |
| [`src/middlewares/validate.middleware.ts`](../src/middlewares/validate.middleware.ts#L11) | `validate` parseia body/query/params e converte `ZodError` em `ValidationError` `[CÓDIGO: EV-CODE-006-A]`. | Criar schemas de UUID, URL HTTPS, status, PATCH e replay; erros específicos de domínio exigem adaptação além do Zod genérico. |
| [`src/shared/errors/http-errors.ts`](../src/shared/errors/http-errors.ts#L1) | Classes herdam de `AppError`; validação usa `VALIDATION_ERROR` e ausência usa `NOT_FOUND` `[CÓDIGO: EV-CODE-006-B a EV-CODE-006-D]`. | Adicionar/exportar classes `WEBHOOK_*` da matriz futura sem alegar que já existem. |
| [`src/shared/logger/index.ts`](../src/shared/logger/index.ts#L4) | `createLogger` usa Pino/redaction; lista atual não cobre secrets ou assinatura `[CÓDIGO: EV-CODE-007-A a EV-CODE-007-C]`. | Reutilizar logger e ampliar redaction antes de qualquer log do worker. |
| [`src/server.ts`](../src/server.ts#L6) | `bootstrap` inicia HTTP e trata SIGINT/SIGTERM `[CÓDIGO: EV-CODE-008-A]`. | Espelhar lifecycle em entrypoint separado candidato `src/worker.ts`, com `PrismaClient` próprio e shutdown que pare polling, aguarde tentativa e desconecte. O caminho é proposta. |
| [`package.json`](../package.json#L10) | Há scripts da API/testes, sem script de worker nem tracing `[CÓDIGO: EV-CODE-008-B, EV-CODE-008-C]`. | Adicionar futuramente scripts de dev/build/start do worker e dependência de métricas escolhida; tracing somente após decisão. |
| [`tests/orders.test.ts`](../tests/orders.test.ts#L1) | Usa Vitest, Supertest e Prisma real; testa transição de status e rollback de estoque por efeitos observáveis `[CÓDIGO: EV-CODE-009-A]`. | Seguir o padrão para API e atomicidade; estender limpeza/factories para as novas tabelas, sem mocks na integração SQL. |

Os auxiliares atuais em [`tests/setup.ts`](../tests/setup.ts#L1) conectam/desconectam o
Prisma e limpam tabelas, e
[`tests/helpers/factories.ts`](../tests/helpers/factories.ts#L8) fornece `getTestApp` e
`bootstrapAuthenticatedUser` `[CÓDIGO: EV-CODE-009-B, EV-CODE-009-C]`. Testes do worker
devem adicionar servidor HTTP local controlado/fake apenas para o destino externo; a
persistência continua usando MySQL real nos testes de integração.

## Critérios de aceite técnicos

### Gates de decisão antes da implementação interoperável

1. Aprovar ou alterar a cardinalidade/modelo de fan-out
   `[PROPOSTA_DERIVADA: EV-PROP-FANOUT-001 a EV-PROP-FANOUT-006]`.
2. Reconciliar cinco tentativas versus cinco retries
   `[QUESTÃO_ABERTA: EV-TR-006-A a EV-TR-006-C]`.
3. Aprovar head-of-line e lease `[QUESTÃO_ABERTA: EV-AMB-001;
   PROPOSTA_DERIVADA: EV-PROP-ORDER-001, EV-PROP-FANOUT-006]`.
4. Fechar bytes, codificação, formato HMAC e operação do grace period
   `[QUESTÃO_ABERTA: EV-AMB-005, EV-AMB-006]`.
5. Fechar redirects e classificação HTTP/rede `[QUESTÃO_ABERTA: EV-AMB-003,
   EV-AMB-004]`.
6. Aprovar armazenamento cifrado, mitigação SSRF e dependência de métricas
   `[QUESTÃO_ABERTA: FDD-OPEN-SECRET-STORAGE-001, FDD-OPEN-SSRF-001]`.

### Matriz de testes e aceite

| ID | Cenário verificável | Nível |
| --- | --- | --- |
| FDD-AT-01 | Forçar falha na inserção da outbox em `changeStatus`; pedido, histórico e efeitos de estoque permanecem no estado anterior, sem linha parcial. | integração MySQL |
| FDD-AT-02 | Mudar para status sem endpoint inscrito; commit ocorre e a contagem de outbox permanece zero. | integração |
| FDD-AT-03 | Dois endpoints ativos, apenas um inscrito no novo status; criar exatamente uma unidade para o inscrito. | integração |
| FDD-AT-04 | Alterar pedido/endpoint depois do commit; payload e URL da unidade continuam iguais ao snapshot original. | integração |
| FDD-AT-05 | Dois endpoints inscritos; linhas têm `outboxId` distintos e o mesmo `eventId`/payload. | integração |
| FDD-AT-06 | Retentar e reprocessar DLQ; `eventId` e `outboxId` ficam estáveis, cada chamada recebe novo `deliveryId`, e replay incrementa o ciclo. | integração worker |
| FDD-AT-07 | Evento A entra em backoff e B do mesmo `orderId` chega depois; B não é claimado até A ter sucesso/DLQ, enquanto evento C de outro pedido avança. | integração/concorrência |
| FDD-AT-08 | Simular relógio para a tabela proposta; chamadas ficam elegíveis em `0/1m/6m/36m/2h36m/14h36m` e a falha final vai à DLQ. | unitário com fake clock |
| FDD-AT-09 | Destino excede dez segundos; tentativa registra `WEBHOOK_DELIVERY_TIMEOUT`, agenda retry e chega à DLQ ao esgotar a política aprovada. | integração HTTP |
| FDD-AT-10 | Vetores fixos comprovam que os bytes enviados produzem `X-Signature`; criação/rotação não loga secret e valida o comportamento aprovado durante 24 h. | unitário + integração de segurança |
| FDD-AT-11 | HTTP, URL malformada e destino SSRF proibido são recusados; corpo de 65.536 bytes é aceito e 65.537 é rejeitado sem truncamento e com rollback. | limites/segurança |
| FDD-AT-12 | Sem token, as sete rotas retornam 401; `ADMIN` e `OPERATOR` acessam CRUD; somente `ADMIN` recebe 202 no replay e `OPERATOR` recebe 403. | Supertest |
| FDD-AT-13 | Replay grava `replayedByUserId`, instante e motivo; duas requisições concorrentes produzem um 202 e um 409, sem duplicar ciclo. | integração/concorrência |
| FDD-AT-14 | Criar 101 tentativas mistas; histórico retorna exatamente as 100 mais recentes, ordenadas, com payload, response, duração e sem secrets/assinaturas. | integração API |
| FDD-AT-15 | Encerrar worker depois do envio e antes da finalização; após expirar lease, tentativa vira `UNKNOWN` e a unidade é reenviada com IDs estáveis. | integração/crash recovery |
| FDD-AT-16 | Cada linha da classificação aprovada produz `SUCCEEDED`, retry ou DLQ esperado; redirects não são seguidos quando essa proposta for aprovada. | testes parametrizados HTTP/rede |
| FDD-AT-17 | Métricas cobrem lag, tentativas, duração, respostas, retries, DLQ e lease; labels não contêm IDs/URL; logs correlacionam os três IDs e aplicam redaction. | observabilidade |
| FDD-AT-18 | Suíte existente de auth/orders continua passando após migration, composição e publicação. | regressão |

O aceite funcional requer também: polling observado a cada dois segundos, worker em
processo separado e operação com um único worker `[FECHADO: EV-TR-004-A a
EV-TR-004-C]`; payload sem items e quatro headers de webhook corretos
`[FECHADO: EV-TR-018-B a EV-TR-018-G]`; e revisão de segurança concluída antes do
deploy `[FECHADO: EV-TR-019-B]`.

## Riscos e mitigação

| Risco | Mitigação/decisão |
| --- | --- |
| Contratos abertos serem implementados como fatos | Gates explícitos acima; ADR/FDD atualizados antes de codificar retry, HMAC, grace period, classificação, redirects, lease e head-of-line. |
| Perda ou duplicação após crash | Outbox transacional, at-least-once, lease/reclaim proposto e IDs estáveis; consumidor deduplica por `X-Event-Id`. |
| Vazamento de secret/assinatura | Cifragem em repouso a aprovar, retorno one-shot, redaction ampliada, ausência em histórico/métricas/traces e dois dias de revisão de segurança. |
| SSRF ou redirect para infraestrutura interna | HTTPS, redirect manual proposto e política de resolução/bloqueio a aprovar antes do deploy. |
| Crescimento/carga no MySQL | Índices, lotes pequenos, métricas de lag/DLQ e single-worker inicial; retenção continua adiada e deve ser acompanhada operacionalmente. |
| Head-of-line atrasar eventos posteriores | Bloqueio somente por `orderId`, outros pedidos continuam; alertar idade por pedido e revisar proposta com dados reais. |
| Single-worker limitar disponibilidade/throughput | Monitorar lag e duração; escala multi-worker não é prometida nesta fase `[FECHADO: EV-TR-004-C, EV-TR-005-B]`. |
| Resposta do cliente conter dados sensíveis ou ser enorme | Sanitizar, propor limite de captura de 16 KiB e nunca registrar headers/secrets; política ainda em revisão. |
| Rotação tornar retries inválidos | Fechar versão/formato/grace period, adicionar vetores e testes atravessando 24 h antes do rollout. |
| Ausência de tracing dificultar diagnóstico | IDs persistidos, métricas e logs estruturados na fase inicial; tracing somente após dependência e política aprovadas. |

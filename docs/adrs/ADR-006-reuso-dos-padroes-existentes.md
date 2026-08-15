# ADR-006: Reuso dos padrões existentes

## Status

Aceito.

## Contexto

A reunião decidiu que webhooks seguirão a estrutura e a infraestrutura compartilhada
da aplicação: módulo com controller, service, repository, routes e schemas Zod, além de
Prisma, `AppError`, middleware centralizado e Pino
([transcrição, 09:27–09:30](../../TRANSCRICAO.md#L160-L180)). A aplicação também já
possui os middlewares de autenticação e autorização que este ADR reaproveita.

Esses padrões existem no código atual:

- o módulo de pedidos expõe [`OrderController`](../../src/modules/orders/order.controller.ts#L6),
  [`OrderService`](../../src/modules/orders/order.service.ts#L26),
  [`OrderRepository`](../../src/modules/orders/order.repository.ts#L18),
  [`buildOrderRouter`](../../src/modules/orders/order.routes.ts#L12) e
  [schemas Zod](../../src/modules/orders/order.schemas.ts#L4);
- [`buildControllers`](../../src/app.ts#L26) e
  [`buildApiRouter`](../../src/routes/index.ts#L21) fazem composição explícita;
- [`validate`](../../src/middlewares/validate.middleware.ts#L11),
  [`authenticate` e `requireRole`](../../src/middlewares/auth.middleware.ts#L27-L60),
  [`AppError`](../../src/shared/errors/app-error.ts#L3) e
  [`errorMiddleware`](../../src/middlewares/error.middleware.ts#L14) formam os padrões
  de validação, autorização e erro;
- [`createPrismaClient`](../../src/config/database.ts#L4) e
  [`createLogger`](../../src/shared/logger/index.ts#L13) fornecem persistência e logs.

## Decisão

Implementar webhooks como novo módulo da aplicação, preservando as mesmas separações de
controller, service, repository, routes e schemas. Reutilizar Prisma para persistência,
Zod e `validate` para entrada, `authenticate`/`requireRole` para acesso, a hierarquia de
`AppError` e o middleware centralizado para falhas, e Pino para logs estruturados. Os
símbolos existentes que sustentam essa decisão estão listados no contexto acima.

Erros específicos do domínio usarão o prefixo `WEBHOOK_`, conforme definido na reunião
([transcrição, 09:28–09:30](../../TRANSCRICAO.md#L170-L180)).

### Adaptações necessárias

- Compor controllers e rotas de webhooks nos pontos explícitos de montagem existentes;
  reuso não torna o novo módulo disponível automaticamente.
- Adicionar os modelos de persistência necessários ao schema Prisma; o schema atual
  contém pedidos e histórico, mas ainda não contém modelos de webhook, outbox ou DLQ
  ([`prisma/schema.prisma`](../../prisma/schema.prisma#L1-L138)).
- Ampliar a redaction do Pino antes de registrar dados do domínio. A lista atual cobre
  autorização, cookies, passwords e tokens, mas não cobre secrets de webhook nem
  `X-Signature` ([`redactPaths`](../../src/shared/logger/index.ts#L4-L11)).
- Criar um entrypoint próprio para o processo do worker e seu ciclo de shutdown. O
  padrão de bootstrap existente está em [`bootstrap`](../../src/server.ts#L6), mas o
  manifest atual só possui scripts da API
  ([`package.json`](../../package.json#L10-L20)). Um entrypoint próprio do worker foi
  sugerido na reunião; o caminho exato ainda é detalhe de implementação
  ([transcrição, 09:11](../../TRANSCRICAO.md#L70-L76) e
  [09:28](../../TRANSCRICAO.md#L162-L168)).
- Reutilizar Pino para eventos operacionais de entrega e definir métricas próprias de
  outbox, tentativas e DLQ. Tracing não é capacidade atual do projeto: não há dependência
  correspondente no manifest ([`package.json`](../../package.json#L25-L52)); eventual
  integração será proposta futura, não reuso já disponível.

## Alternativas Consideradas

- **Serviço separado com stack própria:** não escolhido porque duplicaria persistência,
  autenticação, erros, logs e operação para uma capacidade integrada ao fluxo de
  pedidos.
- **Implementação ad hoc dentro do módulo de pedidos:** não escolhida porque misturaria
  gerenciamento, entrega e segurança de webhooks às responsabilidades de pedidos.
- **Novas bibliotecas para ORM, validação ou logging:** não escolhidas; Prisma, Zod e
  Pino já atendem aos padrões que a reunião mandou reaproveitar.

## Consequências

### Positivas

- O novo domínio mantém convenções já reconhecidas pela equipe e pode compartilhar
  composição, validação, autenticação, erros, persistência e logging.
- Evita-se operar uma stack paralela apenas para webhooks.

### Negativas e trade-off

- O módulo continua acoplado às escolhas estruturais e ao processo de composição desta
  aplicação.
- O reuso exige alterações reais: novos modelos, composição, erros `WEBHOOK_*`,
  redaction, entrypoint do worker e instrumentação operacional.
- Métricas e tracing não surgem do reuso de Pino; precisam de decisões e implementação
  próprias.
- O trade-off aceito é favorecer consistência e menor custo cognitivo, assumindo as
  adaptações necessárias nos pontos de extensão existentes.

# RFC — Arquitetura de Webhooks de Status de Pedidos

## Metadados

- **Autor:** Gabriel Pupom
- **Status:** Em revisão
- **Data:** 2026-08-15
- **Revisores:** Larissa, Marcos, Bruno, Diego e Sofia

Este documento consolida a proposta arquitetural a partir dos ADRs aprovados e do
[Tracker de rastreabilidade](TRACKER.md). Decisões já fechadas
referenciam essas fontes; detalhes ainda não decididos permanecem explicitamente abertos
ou identificados como propostas derivadas para revisão.

## TL;DR

Propõe-se uma capacidade de webhooks **outbound** para comunicar mudanças de status de
pedidos aos clientes B2B, sem receber webhooks desses clientes. A referência de negócio
para a entrega é abaixo de dez segundos
([Tracker de rastreabilidade](TRACKER.md)). Para cumprir esse objetivo sem tornar a
mudança de status dependente da disponibilidade de um endpoint externo, o evento será
registrado em uma outbox no MySQL na mesma transação da alteração do pedido e do
histórico. A entrega HTTP ocorrerá depois, fora da transação, por um worker separado.
Essa direção é fundamentada em [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e
[ADR-005](adrs/ADR-005-worker-separado-com-polling.md).

A operação inicial usará um único worker com polling de dois segundos e entrega
at-least-once. Falhas retentáveis seguem uma política de backoff; depois de esgotada a
política aprovada, a entrega vai para uma DLQ separada e pode ser reprocessada
manualmente. As requisições serão protegidas por HMAC-SHA256, com uma secret por endpoint
e período de convivência de 24 horas na rotação. O novo domínio reutilizará os padrões
modulares, de persistência, validação, autenticação, erros e logs já adotados pela
aplicação. Essas escolhas têm como fontes [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md),
[ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md),
[ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) e
[ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md).

Esta RFC não transforma em contrato o número de chamadas, o ordering durante backoff, o
formato e a operação da assinatura, a classificação de falhas ou rate limiting. Esses
pontos requerem revisão antes de orientar o contrato outbound ou a implementação.

## Contexto e problema

Os clientes precisam ser avisados quando um pedido muda de status, mas a aplicação atual
não possui modelos de webhook, outbox, entrega ou DLQ, nem publica esse tipo de evento no
fluxo de status ([Tracker de rastreabilidade](TRACKER.md)). Ao mesmo tempo, a alteração de pedido e seu histórico já compartilham um
limite transacional no MySQL. Fazer uma chamada HTTP síncrona dentro desse limite faria o
resultado de uma operação de negócio depender da latência, disponibilidade e semântica de
rollback de sistemas externos. A outbox foi escolhida justamente para remover essa janela
de inconsistência entre a mudança confirmada e o evento durável
([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

O problema também é operacional. Endpoints de clientes podem falhar, responder depois do
timeout ou voltar a operar apenas mais tarde. A arquitetura precisa aceitar repetição,
evitar tentativas infinitas e preservar material suficiente para suporte quando uma
entrega sai do fluxo normal. A garantia aceita não é exactly-once: o cliente receberá um
identificador de evento e será responsável por deduplicar possíveis duplicatas
([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md),
[ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).

Por fim, a integração transmite dados de pedidos para fora da infraestrutura. Ela exige
autenticação da origem e integridade da mensagem, tratamento cuidadoso de secrets em
operação e uma estrutura que não crie uma segunda stack para uma capacidade integrada ao
domínio de pedidos. A proposta parte das convenções existentes em vez de introduzir novos
frameworks de persistência, validação ou logging
([ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md),
[ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

## Proposta técnica

### Consistência e desacoplamento

Quando uma mudança de status for confirmada, a proposta é persistir o respectivo evento
na outbox dentro da mesma transação que altera o pedido e registra o histórico. Assim, a
falha ao registrar o evento impede o commit conjunto, enquanto um commit bem-sucedido
deixa estado de negócio e intenção de entrega registrados juntos. O envio HTTP não
participa da transação; ele é uma responsabilidade assíncrona. Essa separação privilegia
consistência local e elimina o acoplamento direto entre o tempo de resposta do cliente e
a mudança de status ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

### Entrega, resiliência e semântica

Um processo separado da API consumirá a outbox, com polling a cada dois segundos e
seleção dos pendentes mais antigos. A primeira operação terá um único worker: isso reduz
a complexidade de concorrência nesta fase, mas não promete ordering global nem escala
multi-worker. A entrega será at-least-once, com um UUID em `X-Event-Id`; duplicatas são
uma consequência assumida e devem ser deduplicadas pelo cliente. Essa escolha evita a
coordenação distribuída exigida por exactly-once, mantendo um mecanismo recuperável
([ADR-005](adrs/ADR-005-worker-separado-com-polling.md),
[ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).

Para falhas classificadas como retentáveis, o worker aplicará backoff dentro de um limite.
Quando a política aprovada se esgotar, a entrega deixará o fluxo ativo e será registrada
em uma DLQ separada com dados de diagnóstico. O replay é manual por um endpoint
administrativo e deve preservar a auditabilidade da ação. O timeout de dez segundos é
conhecido como falha de entrega para retry; a classificação completa das demais respostas
e falhas permanece aberta e será definida em revisão posterior
([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md),
[Tracker de rastreabilidade](TRACKER.md)).

### Segurança e integração modular

Cada requisição outbound será assinada no corpo com HMAC-SHA256 e carregará a assinatura
em `X-Signature`. Cada endpoint terá uma secret própria, rotacionável, cuja versão anterior
permanece válida por 24 horas; endpoints devem usar HTTPS. O isolamento por endpoint
limita o alcance de um vazamento, enquanto a rotação evita uma troca abrupta de
credencial. A serialização assinada, o formato do header e o uso das secrets durante a
convivência não são inferidos aqui, porque não foram decididos
([ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md),
[Tracker de rastreabilidade](TRACKER.md)).

Webhooks serão um novo módulo que segue as fronteiras já usadas pela aplicação:
controller, service, repository, routes e schemas, com Prisma, Zod, autenticação,
autorização, `AppError`, middleware centralizado e Pino. Essa decisão reduz custo
cognitivo e operacional, mas exige os pontos de extensão adequados, modelos de
persistência próprios, redaction de secrets e assinatura nos logs, um ciclo de vida
independente para o worker e instrumentação operacional. Códigos específicos do domínio
usarão o prefixo `WEBHOOK_` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

## Alternativas consideradas

- **HTTP síncrono dentro da transação de mudança de status.** Parecia a rota mais direta,
  pois tentaria entregar imediatamente, mas o trade-off seria aceitar que a latência ou a
  indisponibilidade do cliente prolongasse e pudesse comprometer a transação de pedido.
  Também deixaria ambíguo quando fazer rollback após interação externa. Foi descartada em
  favor da atomicidade local e do desacoplamento da outbox
  ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

- **Redis Streams ou Redis Cluster.** Ofereceria um mecanismo assíncrono externo e uma
  possível base para escala futura, mas o trade-off seria operar infraestrutura adicional
  para um caso que pode reutilizar o MySQL existente. Para a capacidade e a equipe desta
  fase, esse custo foi considerado overengineering; por isso a alternativa foi descartada
  ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

- **Trigger/listener improvisado no MySQL.** Um trigger pode reagir a alterações no banco,
  mas não notifica diretamente um processo externo. O trade-off aparente de evitar polling
  viria com um mecanismo incompleto e frágil para acionar a entrega, além de misturar esse
  problema ao banco. Foi descartado em favor de um worker separado que consome a outbox
  de forma explícita ([ADR-005](adrs/ADR-005-worker-separado-com-polling.md)).

- **Exactly-once.** Reduziria a exposição do consumidor a duplicatas, mas exigiria
  coordenação entre plataforma e clientes e adicionaria complexidade desproporcional ao
  caso. O trade-off aceito é at-least-once com `X-Event-Id` e deduplicação no consumidor
  ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).

## Questões em aberto

1. **Cinco tentativas versus cinco retries.** A reunião mencionou um teto de cinco
   tentativas e também cinco intervalos de espera, mas não resolveu se a tentativa inicial
   faz parte desse total. A interpretação de tentativa imediata mais cinco retries,
   totalizando seis chamadas, é apenas uma proposta derivada para revisão; não é a regra
   fechada desta RFC ([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md)).

2. **Ordering durante backoff.** O single-worker e a seleção por antiguidade não impedem
   que um evento posterior do mesmo pedido seja entregue enquanto o anterior aguarda
   backoff. Head-of-line blocking por `order_id` é uma proposta derivada possível, com o
   custo de atrasar eventos posteriores, mas a decisão de ordering nesse cenário continua
   aberta ([ADR-005](adrs/ADR-005-worker-separado-com-polling.md)).

3. **Formato e rotação da assinatura.** Algoritmo, secret por endpoint e grace period
   estão fechados, mas faltam os bytes e a codificação assinados, a forma/versionamento de
   `X-Signature`, qual secret assina novas entregas durante as 24 horas e como o cliente
   distingue as duas. Nenhuma dessas escolhas deve ser presumida pelo contrato
   ([ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md)).

4. **Classificação de respostas HTTP e falhas de rede.** Fora o timeout de dez segundos,
   ainda não há política aprovada para respostas HTTP, redirects, DNS, TLS, conexão ou
   timeout quanto a retry, sucesso ou DLQ. A decisão precisa preceder o comportamento de
   entrega para evitar que cada caso seja tratado por convenção implícita
   ([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md)).

5. **Rate limiting futuro.** Não há requisito fechado de limitação de envio nesta fase.
   O comportamento deve ser observado e decidido posteriormente com base na operação e
   nos limites dos clientes; esta RFC não introduz quotas, filas adicionais ou regra de
   throttling sem essa decisão ([Tracker de rastreabilidade](TRACKER.md)).

## Impacto e riscos

A outbox, a DLQ e o polling transferem parte do custo para o MySQL: novas gravações,
consultas recorrentes, índices e retenção disputarão recursos com o fluxo transacional.
Esse é o custo aceito para obter atomicidade sem operar um broker. A política de
arquivamento ou limpeza ainda não tem prazo fechado e deve ser tratada como decisão
operacional futura, não como comportamento implícito
([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

Backoff e DLQ aumentam a resiliência, mas também podem atrasar a entrega e exigem suporte
para diagnósticos e replay. Como a garantia é at-least-once, o cliente precisa aplicar
deduplicação por `X-Event-Id`; repetir uma entrega é preferível a perdê-la diante de
incerteza de rede ([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md),
[ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).

Há risco de exposição de secrets ou da assinatura em logs e de integração incompatível
enquanto o formato criptográfico estiver aberto. A mitigação arquitetural já decidida é
secret por endpoint, HMAC e rotação; antes do deploy, a redaction deve abranger os novos
dados sensíveis e devem ser reservados dois dias úteis para revisão de segurança. A
necessidade de tracing continua uma possibilidade futura, não uma capacidade assumida
([ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md),
[ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md),
[Tracker de rastreabilidade](TRACKER.md)).

O single-worker simplifica a fase inicial, mas limita throughput e disponibilidade e não
cria garantia global de ordering. A equipe deve monitorar a meta de entrega e a
pressão no banco antes de propor escala horizontal ou rate limiting. O prazo estimado é
de três sprints; o risco de escopo cresce se as cinco questões abertas não forem
resolvidas durante a revisão arquitetural
([ADR-005](adrs/ADR-005-worker-separado-com-polling.md),
[Tracker de rastreabilidade](TRACKER.md)).

## Decisões relacionadas

- [ADR-001](adrs/ADR-001-outbox-no-mysql.md) — Outbox no MySQL.
- [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) — Retry com backoff e DLQ.
- [ADR-003](adrs/ADR-003-hmac-sha256-por-endpoint.md) — HMAC-SHA256 por endpoint.
- [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) — At-least-once com `X-Event-Id`.
- [ADR-005](adrs/ADR-005-worker-separado-com-polling.md) — Worker separado com polling.
- [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) — Reuso dos padrões existentes.

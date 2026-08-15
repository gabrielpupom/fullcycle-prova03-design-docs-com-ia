# PRD — Webhooks outbound de status de pedidos

Este PRD define o problema, o valor esperado e o comportamento observável da
feature. Ele não define rotas, payloads, modelos de dados, estados internos ou
algoritmos de entrega; esses detalhes pertencem ao RFC, aos ADRs e ao FDD.

**Convenção de evidência.** “Fato da reunião” identifica algo confirmado na
discussão e rastreado pelos IDs `EV-TR-*`. “Proposta derivada” identifica uma
forma de medir, priorizar ou reduzir risco que precisa de revisão; não a transforma
em decisão da reunião.

## Resumo e contexto da feature

O OMS passará a notificar sistemas de clientes B2B quando um pedido mudar de
status. Atlas Comercial, MaxDistribuição e Nova Cargo foram os clientes citados
na reunião. A capacidade é exclusivamente **outbound**: a plataforma envia a
notificação para os clientes, mas não recebe webhooks deles.

Hoje, esses clientes não têm uma notificação automática integrada ao ciclo de vida
do pedido. A feature cria uma capacidade administrável para que cada cliente
configure onde e sobre quais mudanças deseja ser avisado, sem tornar o fluxo de
pedido dependente da disponibilidade do sistema externo.

**Fatos da reunião:** clientes B2B e direção outbound
(`EV-TR-001-A`, `EV-TR-002-A`).

## Problema e motivação

Mudanças de status são relevantes para operações dos clientes, mas a ausência de
notificação automática obriga integrações a dependerem de consulta ou intervenção
posterior. Isso atrasa a reação a eventos de negócio e deixa a plataforma sem um
caminho controlado para lidar com destinatários temporariamente indisponíveis.

O produto precisa comunicar essas mudanças de forma próxima de tempo real, permitir
que o cliente escolha os status relevantes e preservar um caminho de suporte quando
a entrega não for concluída. Ao mesmo tempo, uma falha do destino não pode invalidar
uma mudança de status já solicitada no OMS.

**Fatos da reunião:** referência de tempo real, descarte da chamada síncrona e
necessidade de resiliência (`EV-TR-001-B`, `EV-TR-003-A`,
`EV-TR-007-A`).

## Público-alvo e cenários de uso

O público primário são as equipes e sistemas de integração dos clientes B2B. O
público operacional são as pessoas autenticadas que administram configurações e,
no caso de reprocessamento, administradores autorizados.

- Um cliente cadastra e mantém um destino de notificação para os status que importam
  à sua operação.
- Uma mudança de status compatível com a inscrição do cliente gera uma notificação
  outbound para o sistema dele.
- Se o destinatário estiver indisponível, a plataforma tenta recuperar a entrega e
  oferece histórico para diagnóstico.
- Um administrador reprocessa manualmente uma entrega que saiu do fluxo normal,
  deixando a ação auditável.
- O sistema cliente trata uma possível duplicata usando o identificador único do
  evento.

**Fatos da reunião:** gerenciamento autenticado, filtro por status, histórico,
replay administrativo e deduplicação pelo cliente (`EV-TR-012-C`,
`EV-TR-013-A`, `EV-TR-014-A`, `EV-TR-015-A`, `EV-TR-010-C`).

## Objetivos e métricas de sucesso

| Objetivo | Métrica ou evidência | Situação |
| --- | --- | --- |
| Notificar clientes B2B sobre mudanças relevantes de status | Clientes podem configurar seus destinos, receber notificações outbound e consultar o resultado das entregas. | Fato da reunião (`EV-TR-001-A`, `EV-TR-002-A`, `EV-TR-014-A`). |
| Oferecer experiência de tempo real | **Meta quantitativa da reunião:** abaixo de 10 segundos. | Fato da reunião (`EV-TR-001-B`). |
| Tornar a meta mensurável de modo controlável | Medir o tempo entre o commit da mudança de status e o início da primeira tentativa de entrega, excluindo retries. | Proposta derivada (`EV-PROP-SLI-001`). |
| Entregar a primeira fase | Três sprints foram a estimativa discutida. | Fato da reunião; é estimativa, não data contratual (`EV-TR-019-A`). |
| Reduzir risco antes da liberação | Reservar dois dias úteis para revisão de segurança antes do deploy. | Fato da reunião (`EV-TR-019-B`). |

A referência de menos de 10 segundos não cria, neste documento, percentil, janela de
observação, SLO ou meta de disponibilidade que não tenham sido discutidos. A definição
de medição acima é uma proposta para tornar a referência verificável e deve ser
revisada antes de servir como indicador operacional.

## Escopo

### Incluído nesta fase

- Notificações outbound de mudanças de status de pedidos para clientes B2B.
- Ciclo autenticado de cadastro, consulta, edição, remoção e rotação de secret das
  configurações de webhook.
- Seleção de status por configuração, publicação de notificações elegíveis,
  entrega assinada, recuperação por retry/DLQ, histórico e replay administrativo.
- Identificador único para que o cliente trate a semântica at-least-once.
- Revisão de segurança antes da liberação.

### Fora de escopo, descartado ou adiado

- Receber webhooks dos clientes.
- Avisos por email sobre falhas consecutivas, adiados para fase futura.
- Dashboard visual para o cliente, descartado nesta fase; os endpoints de
  gerenciamento continuam sendo a interface prevista.
- Arquivamento ou limpeza de entregas após aproximadamente 30 dias. A menção a
  “~30 dias” não fixa uma política de retenção e permanece adiada.

### Ponto aberto e adiado

**Rate limiting de envio** será observado e decidido em etapa posterior. Não há
quota, throttling, regra de fila nem critério de aceite de rate limiting para esta
fase.

**Rastreabilidade:** `EV-TR-002-A`, `EV-TR-016-A`, `EV-TR-016-B`,
`EV-TR-016-C` e `EV-TR-016-D`.

## Requisitos funcionais

### PRD-FR-01 — Cadastrar configuração de webhook

O produto deve permitir que uma pessoa autenticada cadastre uma configuração de
webhook para um cliente e informe o destino que receberá notificações. A plataforma
gera a secret associada e a apresenta no momento da criação.

Nesta fase, o CRUD de configuração é aceito para qualquer role autenticada; a
restrição adicional a `ADMIN` vale para o replay administrativo.

**Rastreabilidade:** fato da reunião `EV-TR-012-A`, `EV-TR-012-B` e
`EV-TR-012-C`; regra de role `EV-TR-015-A` e `EV-TR-015-C`.

### PRD-FR-02 — Listar configurações de webhook

O produto deve permitir que uma pessoa autenticada consulte as configurações de
webhook cadastradas para um cliente.

**PROPOSTA_DERIVADA — pendente de aprovação:** a listagem apresenta configurações
sanitizadas, sem expor a secret de operação. Esse comportamento está proposto no FDD,
mas não foi fechado pela reunião e não deve ser tratado como requisito aceito.

**Rastreabilidade:** fato da reunião sobre CRUD autenticado `EV-TR-012-C`;
proposta derivada `FDD-PROP-API-001`.

### PRD-FR-03 — Editar configuração de webhook

O produto deve permitir que uma pessoa autenticada atualize uma configuração de
webhook para manter o destino ou os status de interesse alinhados à operação do
cliente.

**Rastreabilidade:** fato da reunião sobre CRUD autenticado
`EV-TR-012-C`.

### PRD-FR-04 — Excluir configuração de webhook

O produto deve permitir que uma pessoa autenticada remova uma configuração de
webhook que não deva mais receber novas notificações.

**Rastreabilidade:** fato da reunião sobre CRUD autenticado
`EV-TR-012-C`.

### PRD-FR-05 — Rotacionar a secret por endpoint

O produto deve permitir a rotação da secret de cada configuração de webhook. A
secret anterior permanece válida em paralelo por 24 horas, conforme decidido na
reunião.

**Rastreabilidade:** fato da reunião `EV-TR-008-B` e `EV-TR-008-C`.

### PRD-FR-06 — Filtrar notificações pelos status inscritos

O produto deve permitir que cada configuração selecione os status de pedido de seu
interesse. Sem inscrição para o novo status, não deve haver publicação de
notificação na outbox para aquela configuração.

**Rastreabilidade:** fato da reunião `EV-TR-013-A`.

### PRD-FR-07 — Publicar a notificação elegível na outbox

Quando uma mudança de status confirmada tiver configuração inscrita, o produto deve
publicar a intenção de notificação na outbox, desacoplando a mudança de negócio da
entrega ao cliente.

**Rastreabilidade:** fato da reunião `EV-TR-003-B`, `EV-TR-003-C` e
`EV-TR-017-A`.

### PRD-FR-08 — Entregar webhook outbound assinado

O produto deve entregar a notificação ao destino configurado pelo cliente e assinar
o corpo com HMAC-SHA256 usando uma secret exclusiva daquele endpoint.

**Rastreabilidade:** fato da reunião `EV-TR-002-A`, `EV-TR-008-A` e
`EV-TR-008-B`.

### PRD-FR-09 — Recuperar falhas com retry e DLQ

O produto deve tentar recuperar entregas que a política aprovada considerar
retentáveis e encaminhar falhas permanentes ou política esgotada para uma DLQ
separada, preservando material para suporte e recuperação.

A quantidade exata de chamadas, os intervalos e a classificação detalhada das
falhas não são fechados por este requisito.

**Rastreabilidade:** fato da reunião `EV-TR-007-A`, `EV-TR-007-B`,
`EV-TR-018-A`; questões abertas `EV-TR-006-A`, `EV-TR-006-B`,
`EV-TR-006-C` e `EV-AMB-003`.

### PRD-FR-10 — Consultar histórico recente de entregas

O produto deve permitir consultar as últimas 100 entregas de uma configuração,
incluindo resultados de sucesso ou falha, payload, resposta e duração.

**PROPOSTA_DERIVADA — pendente de aprovação:** a consulta não expõe secret ou
assinatura. É um controle de segurança desejável, mas não foi fechado pela reunião e
não integra o comportamento aceito desta fase.

**Rastreabilidade:** fato da reunião `EV-TR-014-A`; proposta derivada
`FDD-PROP-API-001` e `PRD-NFR-08`.

### PRD-FR-11 — Reprocessar DLQ de forma administrativa e auditável

O produto deve permitir que uma pessoa com role `ADMIN` solicite o replay manual
de uma entrega na DLQ e registrar quem executou a ação para auditoria.

**Rastreabilidade:** fato da reunião `EV-TR-007-C`, `EV-TR-015-A` e
`EV-TR-015-B`.

### PRD-FR-12 — Permitir deduplicação pelo cliente

O produto deve identificar cada evento de entrega com UUID único para que o cliente
possa reconhecer e deduplicar mensagens repetidas.

**Rastreabilidade:** fato da reunião `EV-TR-010-A`, `EV-TR-010-B` e
`EV-TR-010-C`.

## Requisitos não funcionais

### PRD-NFR-01 — Atomicidade da mudança de status

A mudança de status, seu histórico e a publicação aplicável na outbox devem ter
resultado atômico: uma falha de publicação impede o commit conjunto.

**Rastreabilidade:** fato da reunião `EV-TR-003-C` e `EV-TR-017-A`.

### PRD-NFR-02 — Latência de primeira tentativa

A referência quantitativa de experiência é abaixo de 10 segundos. Para controlá-la,
propõe-se medir do commit da mudança de status ao início da primeira tentativa de
entrega e excluir retries. Essa proposta não inventa percentil, SLO ou garantia de
disponibilidade.

**Rastreabilidade:** fato da reunião `EV-TR-001-B`; proposta derivada
`EV-PROP-SLI-001`.

### PRD-NFR-03 — Transporte seguro

O destino de webhook deve usar HTTPS; destinos HTTP devem ser recusados na
configuração.

**Rastreabilidade:** fato da reunião `EV-TR-009-A`.

### PRD-NFR-04 — Limite e integridade do payload

O payload outbound não pode exceder 64 KB e não pode ser truncado para caber no
limite.

**Rastreabilidade:** fato da reunião `EV-TR-009-B`.

### PRD-NFR-05 — Tempo máximo da chamada outbound

Cada chamada outbound deve usar timeout de 10 segundos; timeout é uma falha tratada
pelo fluxo de recuperação.

**Rastreabilidade:** fato da reunião `EV-TR-018-A`.

### PRD-NFR-06 — Semântica de entrega

A entrega é at-least-once. Duplicatas podem chegar ao cliente e devem ser tratadas
por ele com o identificador único do evento.

**Rastreabilidade:** fato da reunião `EV-TR-010-A`, `EV-TR-010-B` e
`EV-TR-010-C`.

### PRD-NFR-07 — Identidade e snapshot da notificação

A notificação deve ter UUID e refletir um snapshot capturado quando ela entra na
outbox, evitando que tentativas posteriores alterem o fato de negócio comunicado.

**Rastreabilidade:** fato da reunião `EV-TR-020-A` e `EV-TR-020-B`.

### PRD-NFR-08 — Segurança de secrets e assinaturas

Cada endpoint deve ter secret própria, com rotação de 24 horas de convivência.

**PROPOSTA_DERIVADA — pendente de aprovação:** secret e assinatura não aparecem em
histórico, logs ou métricas. Esse é o controle de segurança desejável para cumprir a
decisão de secret por endpoint, mas não define armazenamento, formato de assinatura
nem comportamento durante a rotação e não deve ser tratado como requisito fechado.

**Rastreabilidade:** fato da reunião `EV-TR-008-A`, `EV-TR-008-B` e
`EV-TR-008-C`; proposta derivada baseada em `EV-CODE-007-C`.

### PRD-NFR-09 — Observabilidade operacional

O produto deve disponibilizar sinais operacionais para acompanhar a latência de
primeira tentativa, resultados, retries, DLQ e replays, com logs estruturados.

**PROPOSTA_DERIVADA (`EV-PROP-SLI-001`):** medir o intervalo entre o commit da
mudança e o início da primeira tentativa, excluindo retries. O conjunto exato de
métricas e o backend de observabilidade também dependem de aprovação; não são decisões
da reunião.

**QUESTÃO ABERTA (`EV-AMB-007`):** tracing não existe no projeto e sua dependência e
integração dependem de decisão futura; portanto, não é uma capacidade assumida nesta
fase. A não exposição de dados sensíveis em logs segue a proposta pendente de
`PRD-NFR-08`; não é um requisito fechado de observabilidade.

**Rastreabilidade:** proposta derivada `EV-PROP-SLI-001`; questão aberta
`EV-AMB-007`; fatos da reunião sobre retry, DLQ e replay `EV-TR-007-A` e
`EV-TR-007-C`.

## Decisões e trade-offs principais

- **Outbox e entrega assíncrona:** a mudança de status não espera uma chamada HTTP do
  cliente. O trade-off aceito é introduzir operação e monitoramento de entregas para
  preservar a consistência do fluxo de pedidos.
- **At-least-once em vez de exactly-once:** a plataforma pode repetir uma
  notificação; o cliente assume a deduplicação pelo identificador do evento. O
  trade-off evita coordenação distribuída mais complexa.
- **Secret por endpoint e HMAC:** limita o alcance de um eventual vazamento e permite
  rotação, ao custo de gestão de credenciais por cliente.
- **Operação inicial simples:** a reunião escolheu worker separado e single-worker.
  Isso reduz a complexidade inicial, mas não promete ordering global nem escala
  multi-worker nesta fase.

### Questões em aberto — não são critérios de aceitação

- A reunião não reconciliou cinco tentativas com cinco intervalos de espera; portanto,
  o número de chamadas e a agenda não são uma regra fechada.
- A classificação de respostas HTTP, redirects e falhas de rede ainda precisa de
  decisão antes de estabilizar o comportamento de recuperação.
- A codificação, os bytes assinados e a operação das duas secrets durante a rotação
  permanecem abertos.
- Há intenção de preservar a ordem por pedido na operação single-worker, mas ordering
  durante backoff não foi decidido. **Head-of-line blocking por pedido é somente uma
  proposta derivada**, com o custo de atrasar eventos posteriores daquele pedido.
- Rate limiting de envio permanece aberto e adiado.
- Tracing não existe no projeto; adotar uma dependência e integrá-la aos processos da
  API e do worker depende de decisão futura.

**Rastreabilidade:** `EV-TR-003-A`, `EV-TR-004-C`, `EV-TR-005-B`,
`EV-TR-006-A`, `EV-AMB-001`, `EV-AMB-003`, `EV-AMB-005`,
`EV-AMB-006`, `EV-AMB-007` e `EV-PROP-ORDER-001`.

## Dependências

- Os clientes precisam disponibilizar destino HTTPS, proteger sua cópia da secret e
  deduplicar eventos repetidos pelo identificador fornecido.
- A feature depende do fluxo confiável de mudança de status do OMS e da persistência
  que sustenta a publicação na outbox.
- A operação depende de monitoramento de entregas e de acesso administrativo para
  diagnosticar e reprocessar itens de DLQ.
- A liberação depende da revisão de segurança de dois dias úteis discutida na reunião.
- As decisões abertas sobre política de retry, interoperabilidade da assinatura e
  ordering sob backoff precisam ser revisadas antes de serem anunciadas como garantias
  ao cliente.

**Rastreabilidade:** `EV-TR-003-B`, `EV-TR-010-C`, `EV-TR-015-A`,
`EV-TR-019-B`, `EV-AMB-001`, `EV-AMB-003`, `EV-AMB-005` e
`EV-AMB-006`.

## Riscos e mitigação

As classificações de probabilidade e impacto abaixo são avaliação de produto
**proposta**, usada para priorização; não são fatos atribuídos à reunião.

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente indisponível ou lento | Média | Alta — pode atrasar ou impedir a entrega imediata | Retry para falhas tratadas pela política, DLQ, histórico e replay administrativo. A agenda e a classificação exatas continuam abertas. |
| Vazamento de secret ou assinatura | Baixa | Muito alto — pode comprometer a confiança na origem das notificações | Secret por endpoint, rotação com 24 horas, não exposição em canais operacionais e revisão de segurança antes do deploy. |
| Crescimento da outbox e pressão no banco | Média | Alta — pode elevar a latência e competir com o fluxo de pedidos | Índices adequados e monitoramento de volume, idade das pendências, retries e DLQ; a política de arquivamento segue adiada. |
| Ordering sob backoff | Média | Média — um evento posterior pode chegar antes de um anterior do mesmo pedido | Revisar a proposta de head-of-line por pedido. Ela preservaria a ordem ao custo de bloquear eventos posteriores e ainda não é decisão fechada. |
| Capacidade limitada do single-worker | Média | Média — maior volume pode aumentar o tempo até a tentativa | Monitorar a referência de latência e a carga antes de decidir escala ou rate limiting; nenhum dos dois é prometido nesta fase. |

**Rastreabilidade:** fatos de risco `EV-TR-004-C`, `EV-TR-005-B`,
`EV-TR-007-A`, `EV-TR-008-B`, `EV-TR-016-C`, `EV-TR-016-D` e
`EV-AMB-001`; priorização e ações de monitoramento são propostas derivadas.

## Critérios de aceitação

Os critérios abaixo verificam somente requisitos fechados ou propostas de medição
explicitamente identificadas. Questões abertas não foram convertidas em aceite.

- **PRD-AC-01 — Gestão autenticada:** uma pessoa autenticada consegue cadastrar,
  listar, editar, remover e rotacionar configurações de webhook; a secret é exibida
  na criação, e a rotação preserva a validade da secret anterior por 24 horas.
  Rastreia `PRD-FR-01` a `PRD-FR-05`.

  **Proposta derivada — pendente de aprovação:** a não exposição da secret na
  listagem é desejável e é rastreada por `PRD-FR-02`, mas fica fora deste critério
  fechado até aprovação.
- **PRD-AC-02 — Elegibilidade e publicação:** uma mudança de status com inscrição
  aplicável gera publicação na outbox; sem inscrição aplicável, não gera. Rastreia
  `PRD-FR-06`, `PRD-FR-07` e `PRD-NFR-01`.
- **PRD-AC-03 — Entrega segura:** a notificação é enviada somente para destino HTTPS,
  com HMAC-SHA256 por endpoint; payload acima de 64 KB não é truncado, e uma chamada
  que exceda 10 segundos é tratada como falha de entrega. Rastreia `PRD-FR-08` e
  `PRD-NFR-03` a `PRD-NFR-05`.
- **PRD-AC-04 — Resiliência e suporte:** uma falha segue o caminho de recuperação
  aprovado e, quando sai do fluxo normal, pode ser localizada na DLQ, consultada no
  histórico e reprocessada por `ADMIN` com auditoria. Rastreia `PRD-FR-09` a
  `PRD-FR-11`.
- **PRD-AC-05 — Semântica para o cliente:** a entrega fornece identificador UUID
  suficiente para o cliente deduplicar uma possível repetição; o snapshot comunicado
  permanece vinculado ao momento de publicação. Rastreia `PRD-FR-12`,
  `PRD-NFR-06` e `PRD-NFR-07`.
- **PRD-AC-06 — Histórico observável:** a consulta retorna no máximo as 100 entregas
  mais recentes com resultado, payload, resposta e duração. Rastreia
  `PRD-FR-10`.

  **Controle de segurança proposto — pendente de aprovação:** não expor secret ou
  assinatura no histórico é desejável e é rastreado por `PRD-NFR-08`, mas fica fora
  deste critério fechado até aprovação.
- **PRD-AC-07 — Evidência de latência:** em validação controlada, é registrado o
  intervalo proposto entre commit da mudança e início da primeira tentativa e o
  cenário validado fica abaixo de 10 segundos. Isso não estabelece percentil, SLO ou
  janela não decididos. Rastreia `PRD-NFR-02`.
- **PRD-AC-08 — Preparação operacional:** existem sinais para acompanhar primeira
  tentativa, resultados, retries, DLQ e replays, e a revisão de segurança de dois
  dias úteis é concluída antes do deploy. Rastreia `PRD-NFR-09` e
  `EV-TR-019-B`.

Não são critérios de aceitação nesta fase: a contagem/agenda final de retries, a
classificação completa de falhas, o formato interoperável da assinatura, a regra de
ordering durante backoff, head-of-line blocking, rate limiting e a retenção após
aproximadamente 30 dias.

## Estratégia de testes e validação

| Frente de validação | Cenários de produto | Resultado esperado |
| --- | --- | --- |
| Gestão de configuração | Criar, listar, editar, remover e rotacionar uma configuração autenticada. | O ciclo é utilizável para a pessoa autenticada. |
| Notificação elegível | Mudar status com e sem inscrição correspondente. | Só a mudança elegível é publicada e preparada para entrega. |
| Consistência | Induzir falha na publicação associada à mudança de status. | Não há confirmação parcial entre a mudança, o histórico e a publicação. |
| Entrega e segurança | Exercitar destino HTTPS, assinatura HMAC, limite de payload e timeout. | As restrições fechadas são respeitadas sem detalhar formatos que permanecem abertos. |
| Resiliência e suporte | Simular indisponibilidade, consultar histórico e executar replay administrativo. | Retry/DLQ/replay são observáveis e a auditoria é preservada, sem pressupor agenda ou classificação ainda abertas. |
| Semântica do consumidor | Repetir uma entrega para um cliente de teste. | O identificador único permite deduplicação pelo cliente. |
| Latência e operação | Coletar a medição proposta de primeira tentativa e sinais de resultados, retries e DLQ. | Há evidência controlável para comparar com a referência de menos de 10 segundos, sem percentis inventados. |
| Segurança antes do deploy | Revisar proteção de secrets, assinaturas e exposição operacional durante dois dias úteis. | Riscos sensíveis são avaliados antes da liberação. |

Os testes de interoperabilidade que dependem de decisão aberta devem permanecer como
gates de revisão, e não como testes que presumem uma regra ainda não aprovada.

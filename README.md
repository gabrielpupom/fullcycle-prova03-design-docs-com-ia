# Processo de produção dos design docs

## Sobre o desafio

Este desafio consistiu em transformar uma transcrição de reunião técnica e uma aplicação Node.js + TypeScript existente em um pacote rastreável de design docs para um sistema de webhooks de notificação de pedidos. O trabalho precisava separar visão de produto, proposta arquitetural, decisões, especificação de implementação e rastreabilidade, mantendo cada afirmação apoiada pela transcrição ou pelo código.

Além de produzir PRD, RFC, ADRs, FDD e Tracker, documentei o próprio processo e preservei o [enunciado original](docs/enunciado-da-prova.md). A aplicação serviu como fonte de contexto: o código não foi alterado e, antes da produção documental, o baseline foi validado com 15/15 testes usando MySQL local.

## Ferramentas de IA utilizadas

- **Codex** — ferramenta principal para leitura do repositório, estruturação do trabalho, produção dos documentos e revisão de consistência.
- **Luna** — usada na matriz de evidências, no Tracker e na redação deste README.
- **Terra** — usada no RFC, no PRD e nas revisões de rastreabilidade.
- **Sol** — usada nos ADRs, no FDD, na revisão independente da spec e nas revisões arquiteturais.

**Ferramentas auxiliares (não são IA):**

- `rg` — buscas dirigidas em transcrição, código, documentos, links e placeholders.
- Shell — execução dos validadores repetíveis e da validação local do baseline.
- Git — leitura do histórico, conferência do escopo público e registro dos commits.
- MySQL local — ambiente usado para validar os 15/15 testes do baseline antes da documentação.

## Workflow adotado

O enunciado foi preservado desde o início, enquanto o código e a transcrição foram lidos como fontes distintas. Primeiro houve brainstorming sobre escopo e fronteiras dos documentos; depois, uma spec estrutural organizou o catálogo de evidências, as propostas derivadas e o checklist. A matriz de evidências passou a funcionar como referência para não transformar ambiguidades em decisões.

A execução principal seguiu exatamente esta ordem: **matriz → ADRs → RFC → FDD → PRD → Tracker → README**. Cada tarefa recebeu uma revisão independente antes da seguinte: os ADRs fecharam as decisões arquiteturais, o RFC apresentou a proposta, o FDD detalhou a implementação, o PRD reorganizou a visão de produto e o Tracker conferiu a origem dos itens. A revisão final verificou a navegação, os links, os placeholders e a preservação da separação entre fatos, questões abertas e propostas derivadas.

## Prompts customizados

Os blocos abaixo são versões reais ou adaptadas dos prompts usados para dirigir a produção e as revisões.

**Prompt de extração e classificação de evidências**

```text
Leia TRANSCRICAO.md e os arquivos de código citados no enunciado. Extraia cada
afirmação atômica que possa alimentar PRD, RFC, ADRs, FDD ou Tracker. Para cada
linha, informe resumo, fonte e localização verificável: timestamp e falante para
TRANSCRICAO, ou caminho real do arquivo para CODIGO.

Classifique cada item somente como FECHADO, ABERTO, DESCARTADO, ADIADO, CODIGO
ou PROPOSTA_DERIVADA. Divida requisitos compostos sem perder o ID-base. Não
promova ambiguidades a decisões: mantenha separadas a contradição entre cinco
tentativas e cinco intervalos, o ordering durante backoff, fan-out/identidade,
classificação de respostas HTTP e rede, formato e rotação de HMAC e a ausência
atual de tracing. Confira também os símbolos reais de OrderService.changeStatus,
autenticação, autorização, erros e logger no código antes de atribuir uma fonte.
```

**Prompt de revisão independente contra enunciado, transcrição e código**

```text
Revise independentemente os documentos produzidos contra docs/enunciado-da-prova.md,
TRANSCRICAO.md e cada caminho de código citado. Para cada achado, devolva: documento,
trecho, evidência primária, classificação (fato fechado, questão aberta ou proposta
derivada) e correção objetiva.

Faça verificações dirigidas: não generalize UUID se o schema tiver exceções; não
chame de “tabela aprovada” uma agenda de retry em revisão; confira o orçamento
UNKNOWN, o ciclo DELETE/replay e o desempate causal por sequência em vez de UUID;
não trate listagem sem secret nem redaction como aceite fechado; valide timestamps,
fontes CODIGO e todos os caminhos compostos do Tracker. Marque como pendente tudo
que a reunião não decidiu e não aceite nomes de arquivos inexistentes.
```

## Iterações e ajustes

Foram **8 iterações principais**: as sete entregas de matriz, ADRs, RFC, FDD, PRD,
Tracker e README, mais uma onda final de correção da revisão global, além de uma etapa
preliminar de spec. A revisão independente do Sol encontrou, na primeira spec
estrutural, ambiguidades de retry/ordering e um checklist incompleto; a spec foi
corrigida com catálogo, propostas derivadas e checklist integral.

- **Matriz:** a revisão encontrou generalização incorreta de UUID e referências internas incompletas; o conteúdo foi corrigido em 1 rodada.
- **ADRs e RFC:** passaram na primeira revisão de tarefa.
- **FDD:** a revisão encontrou falsa “tabela aprovada”, orçamento `UNKNOWN`, ciclo DELETE/replay e desempate causal por UUID; tudo foi corrigido em 1 rodada, preservando o que permanecia aberto ou derivado.
- **PRD:** a revisão encontrou redaction e listagem sem secret tratadas como aceites fechados; a classificação foi corrigida em 1 rodada.
- **Tracker:** a revisão encontrou timestamps insuficientes, fonte `CODIGO` indevida, caminhos compostos insuficientes e redaction sem `PROPOSTA_DERIVADA`; os registros foram corrigidos em 1 rodada.
- **README:** a última iteração consolidou o histórico, separou ferramentas de IA das auxiliares e conferiu a navegação sem alterar os documentos ou a aplicação.
- **Revisão global final:** a oitava iteração acrescentou exemplos JSON aos sete
  contratos, corrigiu a semântica não retroativa do PATCH e separou tracing aberto do
  SLI proposto, com recálculo do Tracker.

## Como navegar a entrega

A ordem sugerida é: [README](README.md) → [PRD](docs/PRD.md) → [RFC](docs/RFC.md) → [ADRs](docs/adrs/) → [FDD](docs/FDD.md) → [Tracker](docs/TRACKER.md).

Os seis ADRs podem ser lidos individualmente: [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md), [ADR-002](docs/adrs/ADR-002-retry-com-backoff-e-dlq.md), [ADR-003](docs/adrs/ADR-003-hmac-sha256-por-endpoint.md), [ADR-004](docs/adrs/ADR-004-at-least-once-com-x-event-id.md), [ADR-005](docs/adrs/ADR-005-worker-separado-com-polling.md) e [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md). Para consultar a origem do trabalho, use o [enunciado preservado](docs/enunciado-da-prova.md).

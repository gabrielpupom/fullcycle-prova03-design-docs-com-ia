# ADR-003: HMAC-SHA256 por endpoint

## Status

Aceito, com formato da assinatura e operação durante a rotação em aberto.

## Contexto

Os webhooks enviam dados de pedidos para sistemas fora da infraestrutura da aplicação.
O cliente precisa verificar que o corpo não foi alterado e que o remetente possui a
secret compartilhada. A reunião definiu HMAC-SHA256, secret exclusiva por endpoint e
rotação com período de convivência
([transcrição, 09:19–09:22](../../TRANSCRICAO.md#L118-L134)).

## Decisão

Assinar o corpo de cada requisição outbound com HMAC-SHA256 e transportar a assinatura
no header `X-Signature`. Cada endpoint cadastrado terá sua própria secret; não haverá
secret global da plataforma. A secret será rotacionável, e a anterior permanecerá
válida em paralelo por 24 horas antes de expirar
([transcrição, 09:20–09:22](../../TRANSCRICAO.md#L120-L134)).

Esta decisão define algoritmo, escopo da chave e duração do grace period. Ela não define
a representação exata dos dados assinados nem qual secret produz cada assinatura
durante as 24 horas.

## Questões em aberto

- Quais bytes do JSON entram no HMAC, qual codificação é usada e se há alguma regra de
  serialização ou canonicalização.
- Qual é o formato de `X-Signature`, incluindo codificação, prefixo e eventual versão.
- Durante o grace period, qual secret assina novas entregas e como o cliente distingue
  assinaturas produzidas com a secret atual e a anterior.

Esses pontos não aparecem fechados na discussão que definiu o HMAC
([transcrição, 09:20–09:22](../../TRANSCRICAO.md#L120-L134)) e deverão ser resolvidos no
contrato outbound antes da implementação.

## Alternativas Consideradas

- **Sem assinatura:** descartada porque o cliente não teria um mecanismo no protocolo
  para verificar integridade e posse da secret pelo remetente.
- **Secret global da plataforma:** descartada explicitamente porque o vazamento de uma
  credencial comprometeria todos os endpoints
  ([transcrição, 09:21](../../TRANSCRICAO.md#L126-L130)).

## Consequências

### Positivas

- HMAC permite detectar alteração do corpo e autenticar o remetente como detentor da
  secret compartilhada.
- O isolamento por endpoint reduz o alcance do vazamento de uma secret.
- A convivência por 24 horas permite migração sem invalidar imediatamente a secret
  anterior.

### Negativas e trade-off

- A plataforma e os clientes precisam armazenar, rotacionar e ocultar material secreto.
- Sem fechar bytes, codificação, formato e comportamento do grace period, clientes
  independentes podem produzir verificações incompatíveis.
- O trade-off aceito é acrescentar gestão de chaves e complexidade de integração para
  obter autenticação e integridade por endpoint; os detalhes interoperáveis permanecem
  bloqueadores de contrato, não decisões implícitas.

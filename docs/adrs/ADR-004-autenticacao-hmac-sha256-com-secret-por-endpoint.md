# ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint

## Status

`[Revisado]`

## Contexto

Os webhooks expõem dados de pedidos para endpoints fora da nossa infraestrutura
([09:19] Sofia). O cliente precisa conseguir:

- validar que a requisição veio realmente da nossa plataforma;
- verificar que ninguém adulterou o payload no caminho.

Fatos relevantes citados na reunião:

- Já houve caso de cliente que vazou uma secret em log de aplicação ([09:22] Diego).
- Se houvesse uma única secret global, o vazamento de uma comprometeria todos os
  clientes ([09:21] Sofia).
- A validação de URL pode ser feita no schema Zod, seguindo o padrão de validação
  já usado no projeto (ex.:
  [src/modules/orders/order.schemas.ts](../../src/modules/orders/order.schemas.ts)
  e o middleware [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts)).

## Decisão

([09:22] Sofia — "Decidido: HMAC-SHA256 sobre o corpo do request, secret por
endpoint, suporte a rotação com grace period de 24h.")

- **Assinatura HMAC-SHA256** calculada sobre o corpo do request, enviada no header
  `X-Signature` ([09:20] Sofia). SHA-256 por ser o padrão de mercado, com
  biblioteca disponível em qualquer cliente sério ([09:20] Sofia).
- **Uma secret única por endpoint de webhook**, não uma secret global da
  plataforma ([09:21] Sofia). A tabela de configuração de webhook armazena
  `url` + `secret` + `customer_id` + estado ativo ([09:21] Bruno/Sofia).
- **Rotação de secret**: endpoint na API para o cliente pedir uma nova secret.
  Ao rotacionar, a secret antiga continua válida **em paralelo por 24 horas**
  (grace period), para o cliente migrar seus sistemas; depois disso, a antiga é
  invalidada ([09:21] Sofia).
- **TLS obrigatório**: a URL do webhook precisa ser `https`. Se o cliente
  cadastrar `http`, a criação é recusada com erro de validação no schema Zod
  ([09:23] Sofia).
- **Limite de tamanho de payload: 64 KB**, com erro (não truncamento) caso
  ultrapasse ([09:24] Diego/Larissa; ver
  [ADR-007](ADR-007-formato-de-payload-headers-e-limites.md)).

## Alternativas Consideradas

- **Secret global única da plataforma.** Descartada em [09:21] por Sofia:
  "se vaza uma, vaza tudo".
- **Truncar o payload quando exceder o limite** em vez de errar. Descartada em
  [09:23]–[09:24]: Sofia é "a favor de erra" — se o evento chegou a esse tamanho,
  "tem algo errado".
- **mTLS ou allowlist de IP em vez de HMAC** — alternativa plausível não discutida
  explicitamente: rejeitada implicitamente porque HMAC-SHA256 é o padrão que "todo
  cliente sério" já consegue implementar ([09:20] Sofia), enquanto mTLS impõe
  gestão de certificados a cada cliente B2B.
- **Rotação sem grace period (corte imediato).** Alternativa plausível: rejeitada
  em favor das 24h para não quebrar a integração do cliente durante a troca
  ([09:21] Sofia).

## Consequências

**Positivas**

- Autenticidade e integridade do payload verificáveis pelo cliente sem
  infraestrutura adicional.
- O raio de impacto de um vazamento de secret fica contido a um único endpoint.
- A rotação com 24h de sobreposição permite troca sem downtime do lado do cliente.
- A validação de `https` e do limite de 64 KB entra no schema Zod, sem novo
  componente arquitetural ([09:23]–[09:24]).

**Negativas / trade-offs**

- A secret precisa ser armazenada de forma recuperável para recalcular o HMAC no
  envio (não pode ser hash unidirecional), aumentando a superfície de proteção do
  dado em repouso — Sofia reservou ao menos dois dias úteis para revisar HMAC e
  geração de secret antes do deploy ([09:46]).
- Durante o grace period de 24h, duas secrets ficam simultaneamente válidas para o
  mesmo endpoint.
- A verificação da assinatura é responsabilidade do cliente; um cliente que não a
  implemente não ganha proteção.

## Rastreabilidade

- Transcrição: [09:19], [09:20], [09:21], [09:22], [09:23], [09:24], [09:46].
- Código: [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts),
  [src/modules/orders/order.schemas.ts](../../src/modules/orders/order.schemas.ts).

# ADR-007-formato-de-payload-headers-e-limites

## Status

`[Revisado]`

## Contexto

Definidas a arquitetura de entrega ([ADR-001](ADR-001-outbox-no-mysql.md) a
[ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)), restam decisões de
contrato do webhook: o que a `webhook_outbox` guarda, o corpo e os headers do
request HTTP, e os limites operacionais. Marcos documentará o contrato no portal
do desenvolvedor ([09:40] Marcos).

Restrição de filtragem: cada endpoint de webhook define a lista de status que quer
receber ([09:33] Marcos) — os status possíveis vêm do enum `OrderStatus`
([prisma/schema.prisma:16-23](../../prisma/schema.prisma#L16-L23):
`PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED`).

## Decisão

- **Snapshot na inserção.** A linha da `webhook_outbox` guarda o **payload já
  renderizado** no momento da mudança de status, não apenas o `order_id` para
  renderizar no envio. Assim, se o pedido mudar depois, o evento ainda reflete o
  estado de quando o status mudou ([09:52] Larissa; [09:52] Diego — "snapshot na
  inserção"; [09:52] Bruno — "Decidido").
- **Filtragem na inserção da outbox.** Se nenhum webhook do cliente quer aquele
  status, o evento nem é inserido, economizando linhas ([09:34] Bruno; [09:34]
  Diego — "Concordo").
- **Corpo do payload** (JSON) ([09:43] Diego): `event_id`, `event_type`
  (ex.: `"order.status_changed"`), `timestamp` em ISO 8601, `order_id`,
  `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da
  order como `total_cents`. **Sem `items`** — o cliente que quiser detalhes chama
  `GET /orders/:id` depois ([09:43] Diego; [09:44] Bruno — "mantém payload
  enxuto").
- **Headers do request** ([09:44] Diego; [09:44]–[09:45] Sofia/Diego):
  - `X-Event-Id`: o UUID do evento (ver
    [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md));
  - `X-Signature`: o HMAC-SHA256 (ver
    [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md));
  - `X-Timestamp`: timestamp do envio, para o cliente detectar replay attack se
    quiser;
  - `X-Webhook-Id`: o id do cadastro de webhook, para o cliente com vários
    endpoints saber qual caiu naquele envio (sugerido por Sofia em [09:44]);
  - `Content-Type: application/json`.
- **Timeout HTTP: 10 segundos** ([09:42] Diego) — ver
  [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md).
- **Limite de payload: 64 KB**, com erro caso ultrapasse ([09:24] Diego/Larissa).
  Tratado como requisito não funcional / validação, não como decisão arquitetural
  separada ([09:24] Larissa).
- Os campos da order no payload correspondem a colunas existentes do modelo
  `Order` ([prisma/schema.prisma:74-97](../../prisma/schema.prisma#L74-L97):
  `orderNumber`, `customerId`, `status`, `totalCents`).

## Alternativas Consideradas

- **Guardar só `order_id` na outbox e renderizar o payload na hora do envio.**
  Descartada em [09:51]–[09:52] por Larissa: se o pedido mudasse depois, o evento
  entregue não refletiria o estado do momento da transição ("tem caso esquisito").
- **Filtrar os status na hora de enviar** (em vez de na inserção). Descartada em
  [09:34] por Bruno: filtrar na inserção economiza linhas na tabela quando nenhum
  webhook do cliente quer aquele status.
- **Incluir os `items` do pedido no payload.** Descartada em [09:43] por Diego
  para não inflar o payload; detalhes ficam disponíveis via `GET /orders/:id`.
- **Truncar payloads acima de 64 KB.** Descartada em [09:23]–[09:24]: Sofia é a
  favor de erro, pois um evento desse tamanho indica que "tem algo errado".

## Consequências

**Positivas**

- Payload enxuto: menos banda, menos exposição de dados, entrega mais rápida.
- O snapshot torna o evento imutável e auditável, independente de mudanças
  posteriores no pedido.
- Filtrar na inserção mantém a `webhook_outbox` menor e o worker mais leve.
- Headers `X-*` dão ao cliente o necessário para dedup, verificação de assinatura,
  detecção de replay e roteamento entre múltiplos endpoints.

**Negativas / trade-offs**

- O payload é um snapshot: dados que mudam depois da transição (ex.: correções no
  pedido) não são propagados — o cliente precisa buscar o estado atual via
  `GET /orders/:id` quando isso importar.
- Filtrar na inserção acopla a escrita da outbox à configuração de webhooks do
  cliente: mudanças na lista de status assinada só valem para eventos futuros.
- O contrato de payload/headers vira uma interface pública versionável; alterações
  exigem coordenação via portal do desenvolvedor ([09:40] Marcos).

## Rastreabilidade

- Transcrição: [09:23], [09:24], [09:33], [09:34], [09:40], [09:42], [09:43],
  [09:44], [09:45], [09:51], [09:52].
- Código: [prisma/schema.prisma](../../prisma/schema.prisma),
  [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts).

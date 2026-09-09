# ADR-005-entrega-at-least-once-com-x-event-id

## Status

`[Revisão]`

## Contexto

Com o padrão Outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e um worker que faz
polling, marca como entregue e retenta em caso de falha
([ADR-002](ADR-002-worker-em-processo-separado-com-polling.md),
[ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md)), é possível que uma
chamada HTTP tenha sucesso no cliente mas a confirmação se perca, levando o worker
a reenviar o mesmo evento ([09:24] Diego).

É preciso definir a garantia de entrega e como o cliente distingue duplicatas
([09:25] Bruno).

## Decisão

([09:26] Larissa — "At-least-once com X-Event-Id pra dedup do lado do cliente.
Decisão.")

- A entrega é **at-least-once**: o cliente pode receber o mesmo evento mais de uma
  vez e precisa estar preparado para isso ([09:24] Diego).
- Cada evento carrega um **`event_id` (UUID)** gerado no momento em que entra na
  `webhook_outbox`, único por evento ([09:25] Diego). O projeto já usa UUID como
  identificador padrão (`uuid` em [package.json](../../package.json);
  `@default(uuid())` em [prisma/schema.prisma](../../prisma/schema.prisma);
  `uuidv4()` em
  [src/middlewares/request-logger.middleware.ts:6](../../src/middlewares/request-logger.middleware.ts#L6)).
- O `event_id` é enviado no header **`X-Event-Id`**. O cliente deduplica por esse
  identificador do lado dele ([09:25] Diego).
- Marcos documentará esse comportamento de forma destacada no portal do
  desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

- **Garantia exactly-once.** Descartada em [09:25] por Diego: exigiria coordenação
  entre os dois lados e complexidade muito maior; at-least-once com `event_id`
  "resolve 99% dos casos", e é o padrão adotado por Stripe e GitHub.
- **Deduplicação no lado da plataforma** (não expor o problema ao cliente) —
  alternativa plausível: rejeitada implicitamente pela mesma razão, já que
  confirmar entrega exatamente uma vez exigiria coordenação com o receptor
  ([09:25] Sofia observou que "isso joga responsabilidade pro cliente"; Diego
  respondeu que é o padrão de mercado).

## Consequências

**Positivas**

- Simplicidade: o worker pode reenviar livremente em caso de dúvida, sem risco de
  perder eventos.
- Alinhado a integrações que os clientes B2B provavelmente já conhecem (Stripe,
  GitHub).
- O `event_id` também serve como chave de correlação em logs e na
  `webhook_dead_letter`.

**Negativas / trade-offs**

- A responsabilidade de deduplicação recai sobre o cliente ([09:25] Sofia); um
  cliente que não a implemente pode processar o mesmo pedido duas vezes.
- Exige documentação clara e visível no portal do desenvolvedor ([09:26] Marcos).
- Não há garantia de entrega única nem, isoladamente, de ordenação — a ordenação
  por `order_id` vem da restrição de single-worker
  ([ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).

## Rastreabilidade

- Transcrição: [09:24], [09:25], [09:26].
- Código: [package.json](../../package.json),
  [prisma/schema.prisma](../../prisma/schema.prisma),
  [src/middlewares/request-logger.middleware.ts](../../src/middlewares/request-logger.middleware.ts).

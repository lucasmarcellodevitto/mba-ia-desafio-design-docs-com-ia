# ADR-001-outbox-no-mysql

## Status

`[Revisado]`

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser
notificados quando o status dos pedidos deles muda na plataforma, substituindo o
polling atual sobre `GET /orders` ([09:00] Marcos). O requisito de latência é
"abaixo de 10 segundos" ([09:02] Marcos).

A mudança de status hoje acontece em `OrderService.changeStatus`
([src/modules/orders/order.service.ts:126-179](../../src/modules/orders/order.service.ts#L126-L179)),
dentro de uma única transação (`this.prisma.$transaction`) que já:

- atualiza `orders` (`tx.order.update`);
- insere em `order_status_history` (`tx.orderStatusHistory.create`);
- debita/repõe `stockQuantity` dos produtos (`debitStock` / `replenishStock`).

Restrições levantadas na reunião:

- Fazer a chamada HTTP do webhook de forma síncrona dentro dessa transação faria
  um cliente lento travar a mudança de status de outros pedidos ([09:04] Bruno).
- Se o cliente estivesse fora do ar, não é aceitável dar rollback na mudança de
  status ([09:04] Bruno).
- O time é pequeno e não quer subir infraestrutura nova só para isso ([09:07] Diego).
- Não pode existir o caso de "status mudou e evento não saiu" — se o evento não
  for registrado, a transação inteira deve falhar ([09:40] Bruno; [09:41] Diego).

O banco atual é MySQL ([prisma/schema.prisma:6](../../prisma/schema.prisma#L6)).

## Decisão

Adotar o **padrão Outbox** sobre o MySQL já existente ([09:08] Larissa —
"Tá decidido então: outbox em MySQL").

- Criar a tabela `webhook_outbox`. Na mesma transação SQL que atualiza `orders` e
  `order_status_history`, inserir uma linha com o evento ([09:06] Diego).
- Se a inserção na outbox falhar, a transação inteira sofre rollback e o evento
  "some junto" com a mudança de status ([09:06] Diego; [09:40] Bruno).
- A linha da outbox terá índice no campo de status
  (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`; um worker lê
  apenas os pendentes em batch pequeno, processa e marca como entregue
  ([09:08] Diego).
- Chave primária da outbox: **UUID**, seguindo o padrão do restante do projeto
  (todos os modelos usam `@id @default(uuid()) @db.Char(36)` —
  [prisma/schema.prisma:26](../../prisma/schema.prisma#L26), [09:51] Larissa).
- A integração com o service de pedidos será feita por uma função
  `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da
  transação corrente, chamada de dentro de `changeStatus` — sem injetar um
  repository inteiro no `OrderService` ([09:41] Bruno; [09:41] Diego).

O processamento e a entrega dos eventos são responsabilidade do worker
(ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).

## Alternativas Consideradas

- **Disparo HTTP síncrono dentro de `changeStatus`.** Descartada em [09:06] por
  Diego ("Síncrono está fora de questão"), com concordância de Larissa em [09:04].
  Motivo: a transação de mudança de status já é pesada; um HTTP call no meio dela
  faz um cliente lento travar mudanças de status de outros pedidos, e a
  indisponibilidade do cliente exigiria rollback do negócio.
- **Fila em Redis Streams / Redis Cluster.** Descartada em [09:07] por Diego
  ("Subir Redis Cluster pra isso é overengineering"); alternativa levantada por
  Larissa em [09:07]. Motivo: exige subir infraestrutura nova para um time
  pequeno; a outbox no MySQL existente resolve.
- **Trigger de banco escrevendo em um sistema externo.** Discutida no contexto de
  notificação do worker e descartada em [09:09] por Diego: trigger de MySQL só
  executa SQL, não notifica processo externo.

## Consequências

**Positivas**

- Consistência transacional garantida: se a mudança de status commitou, o evento
  foi registrado; se deu rollback, o evento não existe ([09:06] Diego).
- Nenhuma infraestrutura nova: usa o MySQL e o Prisma já configurados
  ([src/config/database.ts](../../src/config/database.ts)).
- A carga de entrega (retry, timeout, clientes lentos) fica isolada do caminho
  crítico de escrita de pedidos.

**Negativas / trade-offs**

- A tabela `webhook_outbox` cresce e precisa de uma rotina de arquivamento das
  linhas já entregues; isso foi explicitamente colocado **fora do escopo desta
  feature** ([09:08] Diego).
- Introduz latência inerente entre a mudança de status e a entrega, dependente do
  ciclo de polling do worker (ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- É necessário alterar `OrderService.changeStatus`
  ([src/modules/orders/order.service.ts:126-179](../../src/modules/orders/order.service.ts#L126-L179))
  para chamar `publishWebhookEvent(tx, ...)` dentro da transação — ponto de
  acoplamento entre o módulo de pedidos e o de webhooks.

## Rastreabilidade

- Transcrição: [09:00], [09:02], [09:04], [09:06], [09:07], [09:08], [09:09],
  [09:40], [09:41], [09:51].
- Código: [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts),
  [prisma/schema.prisma](../../prisma/schema.prisma),
  [src/config/database.ts](../../src/config/database.ts).

# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Revisado |
| **Data** | 2026-09-09 |
| **Autor** | Larissa (Tech Lead) |
| **Revisores** | Marcos, Bruno, Diego, Sofia |
| **Documentos relacionados** | [RFC.md](RFC.md) · [ADR-001](adrs/ADR-001-outbox-no-mysql.md) · [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) · [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) · [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) · [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) · [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) · [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md) |
| **Fonte de requisitos** | [TRANSCRICAO.md](../TRANSCRICAO.md) (timestamps `[hh:mm]`) |

> Este documento descreve **como implementar**. O "o que e por quê" está no [RFC](RFC.md)
> e nos ADRs. Toda afirmação é rastreável à transcrição ou ao código.

---

## 1. Contexto e motivação técnica

Este documento detalha o "como construir" da feature de Webhooks de Notificação de
Pedidos, cuja proposta de arquitetura está descrita em [RFC.md](RFC.md) e cujas decisões
estão registradas nas 7 ADRs em [docs/adrs/](adrs/). O FDD assume que o leitor já conhece
a proposta técnica (padrão Outbox, worker dedicado, *polling* de 2 s, retry com backoff,
DLQ, HMAC-SHA256, entrega at-least-once) e foca em modelo de dados, contratos, fluxos,
integração com código real e critérios de aceite acionáveis para implementação.

O sistema hoje **não possui nenhum mecanismo de notificação externa, fila de eventos ou
processamento assíncrono**. A mudança de status de pedido é tratada inteiramente dentro de
`OrderService.changeStatus()`
([src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts), linhas
126‑179), em uma única transação `this.prisma.$transaction(...)`. Essa transação hoje:
(1) busca o pedido com seus itens (`tx.order.findUnique` com `include: { items: true }`);
(2) rejeita transição para o mesmo status e valida a transição via `canTransition()`
(importado de [order.status.ts](../src/modules/orders/order.status.ts)), lançando
`ConflictError` / `InvalidStatusTransitionError`;
(3) debita ou repõe estoque via `debitStock()` / `replenishStock()`, conforme
`shouldDebitStock()` / `shouldReplenishStock()`;
(4) atualiza `order.status` (`tx.order.update`);
(5) insere um registro em `order_status_history` (`tx.orderStatusHistory.create`);
(6) recarrega o pedido com relações (`items`, `history`, `customer`) para retorno.
A feature de webhooks se insere como um **passo adicional dentro dessa mesma transação**
(detalhado em §5.1 e §12.1).

Motivação para o mecanismo assíncrono, conforme registrado na reunião
([TRANSCRICAO.md](../TRANSCRICAO.md)): não se pode fazer a chamada HTTP de forma síncrona
dentro dessa transação — um cliente lento travaria a mudança de status de outros pedidos e
a indisponibilidade do cliente forçaria *rollback* do negócio ([09:04] Bruno) →
[ADR-001](adrs/ADR-001-outbox-no-mysql.md); não se quer introduzir infraestrutura nova
como Redis, dado o tamanho do time ([09:07] Diego); e o MySQL não oferece `LISTEN/NOTIFY`,
o que leva à abordagem de *polling* ([09:09] Diego) →
[ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md). Os clientes B2B (Atlas
Comercial, MaxDistribuição, Nova Cargo) consomem status hoje por *polling* em
`GET /orders`, o que é lento e caro, e esperam latência abaixo de 10 s ([09:00]‑[09:02]
Marcos). A solução reaproveita ao máximo os padrões da codebase ([09:30] Larissa) →
[ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

---

## 2. Objetivos técnicos

1. Registrar o evento de mudança de status **na mesma transação** de `changeStatus`, sem
   possibilidade de "status mudou e evento não saiu" ([09:40] Bruno) — [ADR-001](adrs/ADR-001-outbox-no-mysql.md).
2. Entregar o evento ao endpoint do cliente em **≤ 10 s** no caminho feliz
   (≤ 2 s de latência de *polling* + tempo de rede) ([09:10] Larissa) — [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).
3. Tolerar indisponibilidade do cliente com **5 tentativas** em backoff
   `1m/5m/30m/2h/12h` e, após o esgotamento, **DLQ** ([09:17] Larissa) — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).
4. Permitir ao cliente **autenticar e verificar integridade** via `X-Signature`
   (HMAC‑SHA256), com *secret* por endpoint e rotação com *grace period* de 24 h
   ([09:22] Sofia) — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md).
5. Garantir **at-least-once** com `X-Event-Id` para dedup no cliente ([09:26] Larissa) —
   [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).
6. Entregar o CRUD de configuração, histórico de entregas e *replay* administrativo
   ([09:31]‑[09:36]).
7. **Nenhuma dependência npm nova** — usar `crypto` e `fetch` nativos do Node ≥ 20
   ([package.json](../package.json) → `engines.node >= 20`) — [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

---

## 3. Escopo e exclusões

O escopo de produto e a lista completa de exclusões estão no [PRD §5](PRD.md). Aqui, o
recorte de **implementação**.

### Artefatos a construir / alterar

- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_delivery_attempts`,
  `webhook_dead_letter` (§4).
- Módulo `src/modules/webhooks` (controller / service / repository / routes / schemas /
  errors / worker) — [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).
- Nova *entry-point* `src/worker.ts` + script `npm run worker` ([09:11] Larissa) —
  [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- Extensão de [`OrderService.changeStatus`](../src/modules/orders/order.service.ts) via
  `publishWebhookEvent(tx, …)` ([09:41] Bruno) — §12.
- Endpoints HTTP do §6.

### Limites que afetam a implementação

- **Single-worker apenas** — sem particionamento nem *lock* para múltiplos workers
  ([09:13]); a ordenação garantida é só por `order_id`.
- **Sem rotina de arquivamento** da outbox nesta entrega ([09:08]).
- **Sem canal de *fallback*** (e-mail/alerta ao cliente) — a recuperação da DLQ é manual
  ([09:37]).
- **Sem *rate limiting*** de saída ([09:39]).
- **CRUD de configuração** aberto a qualquer papel autenticado; só o *replay* exige ADMIN
  ([09:37]).
- **At-least-once**, não *exactly-once* ([09:25]).

---

## 4. Modelo de dados

Novas tabelas em [prisma/schema.prisma](../prisma/schema.prisma), seguindo o padrão do
arquivo: PK `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa), `@@map` snake
case, índices explícitos. Gerar migração com `npx prisma migrate dev`
([package.json](../package.json) → `db:migrate`).

```prisma
enum WebhookOutboxStatus {
  PENDING      // aguardando processamento
  PROCESSING   // reservado por um worker
  DELIVERED    // entregue com 2xx
  FAILED       // esgotou as tentativas -> copiado para webhook_dead_letter
}

model WebhookEndpoint {
  id               String        @id @default(uuid()) @db.Char(36)
  customerId       String        @db.Char(36)
  url              String        @db.VarChar(2048)          // sempre https (ADR-004)
  secretCurrent    String        @db.VarChar(255)           // usada para assinar
  secretPrevious   String?       @db.VarChar(255)           // válida durante o grace period
  secretRotatesAt  DateTime?                                // fim do grace period da secretPrevious
  subscribedStatuses Json                                   // string[] de OrderStatus (ADR-007)
  active           Boolean       @default(true)
  createdAt        DateTime      @default(now())
  updatedAt        DateTime      @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id             String              @id @default(uuid()) @db.Char(36)   // == X-Event-Id (ADR-005)
  endpointId     String              @db.Char(36)
  eventType      String              @db.VarChar(64)                     // "order.status_changed"
  orderId        String              @db.Char(36)
  requestId      String?             @db.Char(36)                        // X-Request-Id de origem, p/ correlação (§9.3)
  payload        Json                                                    // snapshot renderizado (ADR-007)
  status         WebhookOutboxStatus @default(PENDING)
  attempts       Int                 @default(0)
  nextAttemptAt  DateTime            @default(now())                     // due time p/ o worker
  lockedAt       DateTime?
  lastError      String?             @db.VarChar(500)
  createdAt      DateTime            @default(now())
  deliveredAt    DateTime?

  endpoint WebhookEndpoint @relation(fields: [endpointId], references: [id])

  @@index([status, nextAttemptAt])   // varredura do worker ([09:08] Diego)
  @@index([orderId])                 // ordering implícita por order (ADR-002)
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDeliveryAttempt {
  id                String   @id @default(uuid()) @db.Char(36)
  outboxId          String   @db.Char(36)
  endpointId        String   @db.Char(36)
  attemptNumber     Int
  requestUrl        String   @db.VarChar(2048)
  responseStatus    Int?                                    // null = timeout / erro de rede
  responseBodySnippet String? @db.VarChar(2048)
  durationMs        Int
  error             String?  @db.VarChar(500)
  createdAt         DateTime @default(now())

  @@index([outboxId])
  @@index([endpointId, createdAt])   // GET /webhooks/:id/deliveries ([09:34] Marcos)
  @@map("webhook_delivery_attempts")
}

model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @unique @db.Char(36)               // evita replay duplicado
  endpointId   String   @db.Char(36)
  payload      Json
  failureReason String  @db.VarChar(500)                   // ADR-003
  replayedAt   DateTime?
  replayedById String?  @db.Char(36)                       // auditoria do replay ([09:36] Sofia)
  createdAt    DateTime @default(now())

  @@index([endpointId])
  @@map("webhook_dead_letter")
}
```

Adicionar `webhookEndpoints WebhookEndpoint[]` ao `model Customer`
([prisma/schema.prisma](../prisma/schema.prisma) linhas 40‑54).

**Configuração** — novas variáveis em [src/config/env.ts](../src/config/env.ts)
(mesmo padrão `z.coerce` / `.default(...)`) e em [.env.example](../.env.example):

| Variável | Default | Origem |
| --- | --- | --- |
| `WEBHOOK_WORKER_POLL_INTERVAL_MS` | `2000` | [09:09]/[09:10] — [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| `WEBHOOK_OUTBOX_BATCH_SIZE` | `20` | [09:08] Diego — "batch pequeno" |
| `WEBHOOK_HTTP_TIMEOUT_MS` | `10000` | [09:42] Diego — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) |
| `WEBHOOK_MAX_ATTEMPTS` | `5` | [09:17] Larissa — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) |
| `WEBHOOK_BACKOFF_SCHEDULE_MIN` | `1,5,30,120,720` | [09:17] Diego (1m/5m/30m/2h/12h) |
| `WEBHOOK_MAX_PAYLOAD_BYTES` | `65536` | [09:24] Diego — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| `WEBHOOK_SECRET_GRACE_PERIOD_HOURS` | `24` | [09:21] Sofia — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| `WEBHOOK_LOCK_TIMEOUT_MS` | `60000` | recuperação de linha `PROCESSING` órfã (§8) |

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro da transação de pedidos)

Origem: [09:06] Diego, [09:34] Bruno, [09:41] Bruno/Diego, [09:52] snapshot —
[ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md).

1. Em [`OrderService.changeStatus`](../src/modules/orders/order.service.ts), após obter
   `refreshed` (linha 169) e antes do `return refreshed!`, chamar a função livre:
   ```ts
   await publishWebhookEvent(tx, { order: refreshed!, fromStatus: from, toStatus: to });
   ```
2. `publishWebhookEvent(tx, input)` (arquivo `src/modules/webhooks/webhook.publisher.ts`):
   a. `endpoints = await tx.webhookEndpoint.findMany({ where: { customerId, active: true } })`.
   b. Filtra os endpoints cujo `subscribedStatuses` contém `toStatus`. **Se nenhum, retorna
      sem inserir nada** ([09:34] Bruno — "economiza linha na tabela").
   c. Para cada endpoint, renderiza o `payload` (snapshot, §6.10) e insere uma linha em
      `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`,
      `id` = UUID (será o `X-Event-Id`).
3. `publishWebhookEvent` **recebe o `tx`** (`Prisma.TransactionClient`) e **não** injeta um
   repository no `OrderService` ([09:41] Diego — "função pura recebendo o tx").
4. Se qualquer `INSERT` falhar, a exceção propaga e o `$transaction` reverte tudo —
   mudança de status + evento ([09:41] Diego — "Se ficar fora da transação, perde a
   garantia toda").

### 5.2 Processamento pelo worker

Origem: [09:09]‑[09:11] Diego/Larissa, [09:30] Bruno —
[ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).

Processo iniciado por `src/worker.ts` (análogo a [src/server.ts](../src/server.ts)):
instancia seu **próprio `PrismaClient`** via `createPrismaClient()`
([src/config/database.ts](../src/config/database.ts)), monta o `WebhookProcessor` e entra
no loop. `SIGINT`/`SIGTERM` → para o loop, aguarda o batch corrente, `prisma.$disconnect()`.

Loop (`setInterval` de `WEBHOOK_WORKER_POLL_INTERVAL_MS`, sem sobreposição de execuções):

1. **Reserva de batch** (transação curta):
   ```sql
   -- seleciona candidatos
   SELECT id FROM webhook_outbox
     WHERE status = 'PENDING' AND nextAttemptAt <= NOW()
     ORDER BY createdAt ASC
     LIMIT :WEBHOOK_OUTBOX_BATCH_SIZE
     FOR UPDATE SKIP LOCKED;
   -- marca como PROCESSING
   UPDATE webhook_outbox SET status='PROCESSING', lockedAt=NOW() WHERE id IN (...);
   ```
   `ORDER BY createdAt ASC` garante ordenação por `order_id` enquanto for *single-worker*
   ([09:12] Diego). `FOR UPDATE SKIP LOCKED` já deixa o caminho pronto para múltiplos
   workers (fora do escopo, [09:13]).
2. Para cada linha reservada, **entrega HTTP** (§5.3).
3. Resultado:
   - **2xx** → `status = DELIVERED`, `deliveredAt = NOW()`; grava `WebhookDeliveryAttempt`.
   - **falha** e `attempts + 1 < WEBHOOK_MAX_ATTEMPTS` → `status = PENDING`,
     `attempts++`, `nextAttemptAt = NOW() + backoff(attempts)`, `lastError` preenchido;
     grava `WebhookDeliveryAttempt`.
   - **falha** e `attempts + 1 >= WEBHOOK_MAX_ATTEMPTS` → §5.4 (DLQ).

### 5.3 Entrega HTTP (uma tentativa)

Origem: [09:20] Sofia (HMAC), [09:42] Diego (timeout), [09:44] Diego (headers) —
[ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md),
[ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md).

1. `body = JSON.stringify(outbox.payload)`.
2. `sentAt = new Date().toISOString()`.
3. **Salvaguarda de tamanho:** se `body` exceder `WEBHOOK_MAX_PAYLOAD_BYTES` (64 KB),
   **não envia**: falha com `WEBHOOK_PAYLOAD_TOO_LARGE` e o evento vai direto para a DLQ,
   sem novas tentativas ([09:23]‑[09:24] Sofia/Larissa). O payload é montado a partir dos
   campos fixos do §6.10, então na prática isso não ocorre.
4. `signature = "sha256=" + hmacSha256Hex(endpoint.secretCurrent, body)` (módulo `crypto`).
5. `POST endpoint.url` com headers do §6.9 e `AbortController` armado em
   `WEBHOOK_HTTP_TIMEOUT_MS` (`fetch` nativo).
6. Sucesso = status **200‑299**. Qualquer outra resposta, *timeout*, DNS ou erro de
   conexão = falha; registrar `responseStatus` (ou `null`) e `error`.
7. Guardar um trecho da resposta (limitado, ex.: 2 KB) em `responseBodySnippet` para
   diagnóstico.

`backoff(attempts)` = `WEBHOOK_BACKOFF_SCHEDULE_MIN[attempts - 1]` minutos. A progressão é
fixa (`1/5/30/120/720`), sem randomização — exatamente os valores definidos em [09:17].

### 5.4 Retry e DLQ

Origem: [09:15]‑[09:19] Diego/Larissa, [09:35]‑[09:36] —
[ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).

**Retry:** enquanto `attempts < WEBHOOK_MAX_ATTEMPTS`, a linha volta a `PENDING` com
`nextAttemptAt` no futuro. Janela total ≈ 1 + 5 + 30 + 120 + 720 min ≈ **14h36** entre a
1ª falha e a última tentativa ([09:17] Diego).

**DLQ (transação única):**
1. `INSERT webhook_dead_letter { outboxId, endpointId, payload, failureReason }`
   (`failureReason` = `lastError` truncado — ex.: `HTTP 503`, `timeout after 10000ms`).
2. `UPDATE webhook_outbox SET status='FAILED' WHERE id = :outboxId`.
3. Log `warn` `webhook.dead_letter` com `event_id`, `endpoint_id`, `order_id`, `attempts`.

Sem retry automático após a DLQ e **sem alerta proativo ao cliente** — e-mail está fora de
escopo ([09:37] Larissa).

### 5.5 Replay manual da DLQ

Origem: [09:18] Diego, [09:36] Sofia/Larissa — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role **ADMIN**, §6.14):
1. Carrega `webhook_dead_letter` por `id`; 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` se ausente.
2. Se `replayedAt != null` → 409 `WEBHOOK_ALREADY_REPLAYED`.
3. Transação: `UPDATE webhook_outbox SET status='PENDING', attempts=0, nextAttemptAt=NOW(),
   lastError=NULL WHERE id = dead_letter.outboxId` **+** `UPDATE webhook_dead_letter SET
   replayedAt=NOW(), replayedById = :userId`.
4. Log `info` `webhook.replay` com `dead_letter_id`, `outbox_id`, `replayed_by` (auditoria,
   [09:36] Sofia).
5. `202 Accepted`.

### 5.6 Rotação de secret

Origem: [09:21] Sofia — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md).

`POST /api/v1/webhooks/:id/rotate-secret`:
1. `secretPrevious = secretCurrent`; `secretCurrent = generateSecret()` (bytes aleatórios
   via `crypto`; parâmetros finais na revisão de segurança — [09:46] Sofia);
   `secretRotatesAt = NOW() + WEBHOOK_SECRET_GRACE_PERIOD_HOURS`.
2. Retorna a **nova** secret em texto puro (única oportunidade — igual à criação, [09:31]).
3. Requisito ([09:21] Sofia — "a antiga fica válida por 24 horas em paralelo"): durante as
   24 h seguintes à rotação, uma entrega deve poder ser validada pelo cliente **com a
   secret nova ou com a anterior**, para ele migrar sem downtime. A estratégia exata de
   assinatura na janela de sobreposição (assinar com a anterior até `secretRotatesAt` e
   então trocar, ou enviar duas assinaturas) fica para a revisão de segurança
   ([09:46] Sofia). `secretPrevious` guarda a secret anterior até o fim da janela.
4. Job de limpeza (no próprio loop do worker): `secretPrevious = NULL` quando
   `secretRotatesAt < NOW()`.

---

## 6. Contratos públicos

Base: `/api/v1`. Todas as rotas passam por `authenticate`
([src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts)). Validação
via `validate({ body, query, params })`
([src/middlewares/validate.middleware.ts](../src/middlewares/validate.middleware.ts)).
Formato de erro conforme [src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts):
`{ "error": { "code", "message", "details?" } }`.

### 6.1 `POST /api/v1/webhooks` — criar endpoint

`customerId` vem **no body**, não do JWT ([09:32] Larissa).

Request:
```json
{
  "customerId": "0e6f1c2a-...-b1",
  "url": "https://hooks.atlascomercial.com/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```
Response `201`:
```json
{
  "id": "b4a1...",
  "customerId": "0e6f1c2a-...-b1",
  "url": "https://hooks.atlascomercial.com/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "<secret gerada — exibida apenas nesta resposta>",
  "createdAt": "2026-09-09T12:00:00.000Z"
}
```
Semântica: `secret` é gerada pela plataforma e **devolvida só nesta resposta**
([09:31] Marcos). `url` precisa ser `https` (senão `400 WEBHOOK_INVALID_URL`, [09:23]
Sofia). `subscribedStatuses` ⊆ enum `OrderStatus`
([prisma/schema.prisma](../prisma/schema.prisma) linhas 16‑23); vazio → `400
WEBHOOK_INVALID_EVENT_FILTER`. `customerId` inexistente → `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### 6.2 `GET /api/v1/webhooks` — listar endpoints de um customer

Query: `customerId` (obrigatório), `page` (default `1`), `pageSize` (default `20`, máx
`100`). Lista endpoints de um customer ([09:33] Bruno). Formato de
[src/shared/http/response.ts](../src/shared/http/response.ts). **`secret` nunca retorna.**

Response `200`:
```json
{
  "data": [
    {
      "id": "b4a1c9e0-1111-2222-3333-444455556666",
      "customerId": "0e6f1c2a-...-b1",
      "url": "https://hooks.atlascomercial.com/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-09T12:00:00.000Z",
      "updatedAt": "2026-09-09T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Status: `200` sempre (lista vazia se não houver endpoints); `400 VALIDATION_ERROR` se
`customerId` ausente (via `validate`).

### 6.3 `GET /api/v1/webhooks/:id` — detalhe de um endpoint

Response `200`:
```json
{
  "id": "b4a1c9e0-1111-2222-3333-444455556666",
  "customerId": "0e6f1c2a-...-b1",
  "url": "https://hooks.atlascomercial.com/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotatesAt": null,
  "createdAt": "2026-09-09T12:00:00.000Z",
  "updatedAt": "2026-09-09T12:00:00.000Z"
}
```
Status: `200`; `404` se não existe:
```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook endpoint not found" } }
```

### 6.4 `PATCH /api/v1/webhooks/:id` — editar endpoint

Edita `url`, `subscribedStatuses` e/ou `active` ([09:33] Bruno). **Não** edita `secret`
(usar §6.6). Todos os campos são opcionais; ao menos um deve vir.

Request:
```json
{
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```
Response `200` (objeto completo, mesmo shape do §6.3, já atualizado).
Status: `200`; `400 WEBHOOK_INVALID_URL` se `url` não for `https` ([09:23] Sofia);
`400 WEBHOOK_INVALID_EVENT_FILTER` se `subscribedStatuses` inválido; `404 WEBHOOK_NOT_FOUND`.

### 6.5 `DELETE /api/v1/webhooks/:id` — remover endpoint

Sem corpo de request. Response `204` **sem corpo** ([09:33] Bruno). `404 WEBHOOK_NOT_FOUND`
se não existe.

### 6.6 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Sem corpo de request. Fluxo em §5.6 ([09:21] Sofia).

Response `200`:
```json
{
  "id": "b4a1c9e0-1111-2222-3333-444455556666",
  "secret": "<nova secret — exibida apenas nesta resposta>",
  "secretRotatesAt": "2026-09-10T12:00:00.000Z"
}
```
Status: `200`; `404 WEBHOOK_NOT_FOUND`.
A `secret` retorna **apenas nesta resposta** (igual à criação).

### 6.7 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Query: `page`, `pageSize`. Últimas tentativas do endpoint, mais recentes primeiro
([09:34] Marcos).

Response `200`:
```json
{
  "data": [
    {
      "id": "d1e2f3a4-...-01",
      "outboxId": "6f9619ff-8b86-d011-b42d-00cf4fc964ff",
      "attemptNumber": 1,
      "requestUrl": "https://hooks.atlascomercial.com/orders",
      "responseStatus": 200,
      "responseBodySnippet": "{\"received\":true}",
      "durationMs": 143,
      "error": null,
      "createdAt": "2026-09-09T12:00:02.100Z"
    },
    {
      "id": "d1e2f3a4-...-00",
      "outboxId": "1c0b...-ff",
      "attemptNumber": 3,
      "requestUrl": "https://hooks.atlascomercial.com/orders",
      "responseStatus": null,
      "responseBodySnippet": null,
      "durationMs": 10000,
      "error": "timeout after 10000ms",
      "createdAt": "2026-09-09T11:20:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```
Status: `200`; `404 WEBHOOK_NOT_FOUND` se o endpoint não existe.

### 6.8 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar item da DLQ

Role **ADMIN** obrigatória (§6.14). Sem corpo de request. Fluxo em §5.5.

Response `202`:
```json
{
  "deadLetterId": "aa11bb22-...-cc",
  "outboxId": "6f9619ff-8b86-d011-b42d-00cf4fc964ff",
  "status": "PENDING",
  "replayedAt": "2026-09-10T09:00:00.000Z"
}
```
Status: `202` (reenfileirado); `403 FORBIDDEN` se o papel não for ADMIN
([09:36] Sofia):
```json
{ "error": { "code": "FORBIDDEN", "message": "Insufficient permissions" } }
```
`404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_ALREADY_REPLAYED` se já reprocessado:
```json
{ "error": { "code": "WEBHOOK_ALREADY_REPLAYED", "message": "Dead-letter entry already replayed" } }
```

### 6.9 Headers do request enviado ao cliente

Origem: [09:44]‑[09:45] Diego/Sofia — [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md).

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (`webhook_outbox.id`) — chave de dedup ([09:25] Diego) |
| `X-Webhook-Id` | `webhook_endpoints.id` — para o cliente com vários endpoints ([09:44] Sofia) |
| `X-Timestamp` | ISO 8601 do envio — permite ao cliente detectar *replay attack* ([09:44] Diego) |
| `X-Signature` | `sha256=<hmac_hex>` — HMAC-SHA256 de `secretCurrent` sobre o corpo bruto ([09:20] Sofia) |

### 6.10 Payload do evento (exemplo)

Origem: [09:43] Diego — [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md).
**Snapshot** no momento da transição ([09:52]). Sem `items` — o cliente busca detalhes em
`GET /orders/:id` ([09:43] Diego).

```json
{
  "event_id": "6f9619ff-8b86-d011-b42d-00cf4fc964ff",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-09T12:00:00.000Z",
  "order_id": "1a2b3c4d-...-99",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "0e6f1c2a-...-b1",
  "total_cents": 154900
}
```

### 6.11 Interpretação da resposta do **cliente**

Regra única ([09:15] Diego; [09:42] Diego): **`200‑299` = entregue** (`DELIVERED`).
**Qualquer outra resposta, *timeout* (10 s) ou erro de rede = falha**, e o evento entra no
ciclo de retry (§5.4). Não há, nesta fase, tratamento diferenciado por código de status
(ex.: parar de tentar em `4xx`) — possível melhoria futura.

### 6.12 Verificação da assinatura (documentação para o cliente — portal do dev, [09:26] Marcos)

```
esperado = "sha256=" + HMAC_SHA256(secret, corpo_bruto_do_request)  // hex
comparar em tempo constante com o header X-Signature
durante 24h após rotação, aceitar assinatura com a secret nova OU a anterior
```

### 6.13 Idempotência (cliente)

`X-Event-Id` é estável por evento e reenviado em todas as tentativas ([09:25] Diego). O
cliente deve persistir os `event_id` já processados e ignorar repetidos —
[ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).

### 6.14 Autorização

- CRUD de configuração, `deliveries`, `rotate-secret`: **qualquer papel autenticado** por
  enquanto ([09:37] Sofia).
- `replay` de DLQ: **`requireRole('ADMIN')`**, reaproveitando
  [src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts) linhas 49‑61
  ([09:36] Larissa). Padrão de uso idêntico ao de
  [src/modules/users/user.routes.ts](../src/modules/users/user.routes.ts) linhas 12‑18.

---

## 7. Matriz de erros previstos (`WEBHOOK_*`)

Todas as classes estendem `AppError`
([src/shared/errors/app-error.ts](../src/shared/errors/app-error.ts)) e ficam em
`src/modules/webhooks/webhook.errors.ts`, seguindo o padrão de
[src/shared/errors/http-errors.ts](../src/shared/errors/http-errors.ts) (prefixo por
domínio — [09:29] Larissa). O [error.middleware.ts](../src/middlewares/error.middleware.ts)
já serializa `AppError` sem alteração ([09:29] Bruno).

### 7.1 Erros de API (HTTP)

| Código | HTTP | Quando | Classe |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | endpoint inexistente em GET/PATCH/DELETE/deliveries/rotate | `WebhookNotFoundError extends NotFoundError` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` do body não existe | `extends NotFoundError` |
| `WEBHOOK_INVALID_URL` | 400 | `url` não é `https` ou é malformada ([09:23] Sofia) | `extends ValidationError` |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | `subscribedStatuses` vazio ou com valor fora de `OrderStatus` | `extends ValidationError` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | operação que exige secret sem que exista (estado inconsistente) ([09:29] Bruno) | `extends ValidationError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | replay de item de DLQ inexistente | `extends NotFoundError` |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | replay de item de DLQ já reprocessado | `extends ConflictError` |

Erros genéricos continuam vindo das classes atuais: `UNAUTHORIZED`, `FORBIDDEN`
(`requireRole`), `VALIDATION_ERROR` (Zod, via `validate`).

### 7.2 Motivos de falha do worker (não-HTTP — gravados em `webhook_dead_letter.failureReason` / `lastError`)

| Rótulo | Quando |
| --- | --- |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | resposta com status ≥ 300 (inclui o status: `HTTP 503`) |
| `WEBHOOK_DELIVERY_TIMEOUT` | sem resposta em `WEBHOOK_HTTP_TIMEOUT_MS` ([09:42]) |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | DNS / conexão recusada / TLS |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | `body` > `WEBHOOK_MAX_PAYLOAD_BYTES` (64 KB), verificado antes do envio (§5.3) — vai direto para a DLQ ([09:23]‑[09:24]) |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | motivo final ao mover para a DLQ ([09:17]) |

---

## 8. Estratégias de resiliência

Origem: [09:15]‑[09:19], [09:42] — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).

| Mecanismo | Definição |
| --- | --- |
| **Timeout HTTP** | `WEBHOOK_HTTP_TIMEOUT_MS = 10000` via `AbortController` ([09:42] Diego). |
| **Retry** | até `WEBHOOK_MAX_ATTEMPTS = 5` ([09:16] Diego rejeitou 3). |
| **Backoff** | progressão fixa `1m/5m/30m/2h/12h` via `nextAttemptAt` ([09:17] Diego). |
| **DLQ** | tabela dedicada `webhook_dead_letter` ([09:18] Diego), com replay manual (§5.5). |
| **Fallback** | **não há** fallback automático de canal (e-mail fora de escopo — [09:37]). O "fallback" operacional é o replay administrativo. |
| **Transação atômica** | evento só existe se `changeStatus` comitou ([09:41]). |
| **Lock / linhas órfãs** | linha em `PROCESSING` com `lockedAt` mais antigo que `WEBHOOK_LOCK_TIMEOUT_MS` volta a `PENDING` no início do loop (recuperação de crash do worker). |
| **Idempotência de entrega** | at-least-once + `X-Event-Id` ([09:25]) — o cliente deduplica. |
| **Isolamento de falha** | worker em processo separado: crash do worker não afeta a API; restart da API não mata o worker ([09:11] Diego). |
| **Limite de payload** | 64 KB, erro (não trunca) ([09:24] Sofia/Diego). |
| **Graceful shutdown** | `SIGTERM` → termina o batch atual, não pega novos, `$disconnect` (padrão de [src/server.ts](../src/server.ts) linhas 13‑21). |

---

## 9. Observabilidade

Reutiliza o logger Pino único ([src/shared/logger/index.ts](../src/shared/logger/index.ts)),
que **já redige** `*.token` / `*.secret`-like paths — **adicionar** `*.secret`,
`*.secretCurrent`, `*.secretPrevious`, `req.headers.x-signature` à lista `redactPaths`
(linhas 4‑11). Nenhuma lib nova ([09:29] Bruno).

### 9.1 Logs estruturados (eventos nomeados, estilo `http_request` do projeto)

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook.event_published` | info | `event_id`, `endpoint_id`, `order_id`, `event_type`, `to_status` |
| `webhook.delivery_attempt` | info | `event_id`, `endpoint_id`, `attempt`, `response_status`, `duration_ms`, `outcome` |
| `webhook.delivery_retry_scheduled` | warn | `event_id`, `attempt`, `next_attempt_at`, `last_error` |
| `webhook.dead_letter` | warn | `event_id`, `endpoint_id`, `order_id`, `attempts`, `failure_reason` |
| `webhook.replay` | info | `dead_letter_id`, `outbox_id`, `replayed_by` |
| `webhook.secret_rotated` | info | `endpoint_id`, `rotates_at` (nunca a secret) |
| `webhook.worker_tick` | debug | `batch_size`, `picked`, `delivered`, `failed`, `tick_ms` |

### 9.2 Métricas (expor via log agregável nesta fase; endpoint `/metrics` fica para depois)

| Métrica | Tipo | Labels |
| --- | --- | --- |
| `webhook_events_published_total` | counter | `to_status` |
| `webhook_deliveries_total` | counter | `outcome` = `delivered\|retry\|dead_letter` |
| `webhook_delivery_duration_ms` | histogram | — |
| `webhook_outbox_pending` | gauge | (idade do item mais antigo `PENDING`) |
| `webhook_dead_letter_total` | counter | `failure_reason` |
| `webhook_replay_total` | counter | — |
| `webhook_worker_tick_duration_ms` | histogram | — |

Alerta sugerido: `webhook_outbox_pending` (idade) > 30 s por 2 min → o worker está parado
ou atrasado (o SLA é 10 s, [09:02]).

### 9.3 Tracing / correlação

Não há APM nem biblioteca de tracing no projeto, e nenhuma é adicionada. A correlação
ponta a ponta usa o **`event_id`** (= `X-Event-Id`), presente em todos os logs de API e do
worker para um mesmo evento. Para amarrar o evento ao request HTTP que originou a
transição, a `webhook_outbox` tem a coluna opcional `requestId` (§4), alimentada com o
`X-Request-Id` que
[request-logger.middleware.ts](../src/middlewares/request-logger.middleware.ts) (linhas
6‑8) já gera por request.

---

## 10. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 ([package.json](../package.json) `engines`). `crypto` (HMAC,
  secrets) e `fetch` + `AbortController` (HTTP com timeout) são nativos → **zero pacotes
  novos** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).
- **Banco:** mesmo MySQL / mesma `DATABASE_URL` ([09:30] Bruno). Nova migração Prisma
  aditiva (só `CREATE TABLE` + `ALTER TABLE ADD FKs`), sem alterar tabelas existentes →
  compatível com dados atuais. `prisma/migrations/` ganha uma pasta nova.
- **Prisma Client:** o worker cria uma instância separada via
  [src/config/database.ts](../src/config/database.ts) (`PrismaClient` é por processo —
  [09:30] Bruno).
- **API:** mudança **retrocompatível** — `POST/PATCH /orders`, `PATCH /orders/:id/status`
  mantêm contrato; só ganham o efeito colateral de enfileirar eventos.
- **Deploy:** novo processo `npm run worker` a ser adicionado ao orquestrador
  (`docker-compose.yml` / infra). A API **não** deve iniciar o worker.
- **Tooling:** ESLint/Prettier/Vitest atuais cobrem o novo módulo sem config nova
  (`eslint . --ext .ts`, `vitest run`).
- **Migração de dados:** nenhuma (tabelas nascem vazias).
- **Rollback:** desligar o processo worker e remover o registro do router de webhooks
  interrompe a feature; as tabelas podem permanecer vazias sem impacto.

---

## 11. Critérios de aceite técnicos

**Outbox / transação** — [ADR-001](adrs/ADR-001-outbox-no-mysql.md)
1. `changeStatus` que comita grava exatamente uma linha `webhook_outbox` por endpoint
   ativo inscrito no `toStatus`; nenhuma para status não inscrito ([09:34]).
2. Se `publishWebhookEvent` lança, `orders` / `order_status_history` / estoque **não**
   mudam (teste: forçar erro no insert da outbox e checar rollback) ([09:41]).
3. `webhook_outbox.payload` é snapshot: alterar o pedido depois não muda o payload
   gravado ([09:52]).

**Worker / polling** — [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)
4. Com o worker ativo e cliente respondendo `200`, o evento fica `DELIVERED` em ≤ 4 s
   (2 s de tick + margem) ([09:10]).
5. `npm run worker` sobe processo independente; matar a API não interrompe entregas.
6. Dois eventos do mesmo `order_id` criados em sequência são entregues em ordem de
   `createdAt` ([09:12]).

**Retry / DLQ** — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)
7. Cliente retornando `503` gera 5 `webhook_delivery_attempts` com `nextAttemptAt`
   respeitando `1/5/30/120/720` min, e na 5ª cria `webhook_dead_letter` + `status=FAILED`.
8. Cliente que não responde é abortado em 10 s e a tentativa conta como falha ([09:42]).
9. `POST /admin/webhooks/dead-letter/:id/replay` como `OPERATOR` → `403`; como `ADMIN` →
   `202`, recria `PENDING`, grava `replayedById`; segundo replay → `409
   WEBHOOK_ALREADY_REPLAYED` ([09:36]).

**Segurança** — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
10. `url` `http://` → `400 WEBHOOK_INVALID_URL`; `https://` aceito ([09:23]).
11. `X-Signature` recebido pelo cliente-teste confere com
    `HMAC_SHA256(secret, corpo_bruto)` em hex ([09:20]).
12. Após `rotate-secret`, envios assinam com a nova secret; `secretPrevious` é limpa após
    24 h ([09:21]).
13. `secret` aparece **apenas** nas respostas de criação e de `rotate-secret`; nunca em
    GET/list, nunca em log (`redact`).

**Entrega / idempotência** — [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
14. `X-Event-Id` é o mesmo em todas as tentativas de um evento e igual a
    `webhook_outbox.id` ([09:25]).

**Contratos / erros** — [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md), §7
15. Payload contém exatamente os campos do §6.10 (sem `items`) ([09:43]).
16. Headers do §6.9 presentes em todo request de entrega ([09:44]‑[09:45]).
17. `GET /webhooks/:id/deliveries` pagina e retorna sucesso/falha, status, `durationMs`,
    trecho da resposta ([09:34]).
18. Todo erro do módulo responde `{ error: { code: "WEBHOOK_...", message } }` via o
    middleware atual, sem alterá-lo ([09:29]).

**Padrões** — [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
19. `src/modules/webhooks` tem `controller`/`service`/`repository`/`routes`/`schemas` e é
    registrado em [src/routes/index.ts](../src/routes/index.ts) ([09:27]‑[09:28]).
20. `npm run lint` e `npm test` passam; testes de integração seguem o estilo de
    [tests/orders.test.ts](../tests/orders.test.ts) (supertest + factories).

---

## 12. Integração com o sistema existente

Esta seção nomeia os arquivos reais que serão **criados** ou **alterados** e como o módulo
de webhooks se conecta a cada um.

### 12.1 [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts) — **alterar**

- Em `changeStatus` (linhas 126‑179), dentro do `this.prisma.$transaction`, após o
  `tx.orderStatusHistory.create` (linha 159) e o `refreshed` (linha 169), inserir:
  `await publishWebhookEvent(tx, { order: refreshed!, fromStatus: from, toStatus: to })` (função livre, não método).
- `publishWebhookEvent` é uma **função** importada de
  `src/modules/webhooks/webhook.publisher.ts`, recebendo o `Prisma.TransactionClient`
  (`tx`) já usado no arquivo (alias `TxClient`, linha 24). **Não** injetar
  `WebhookRepository` no construtor do `OrderService` ([09:41] Diego — "função pura
  recebendo o tx"; [09:41] Bruno — "publishWebhookEvent(tx, order, fromStatus, toStatus)").
- `create` (linha 50) **não** dispara webhook (o requisito é mudança de status; a criação
  já entra como `PENDING` sem transição observável pelos clientes B2B — [09:31]/[09:33]).

### 12.2 [src/shared/errors/](../src/shared/errors/) (`app-error.ts`, `http-errors.ts`, `index.ts`) — **reutilizar / estender**

- `src/modules/webhooks/webhook.errors.ts` define `WebhookNotFoundError`,
  `WebhookInvalidUrlError`, `WebhookAlreadyReplayedError`, etc., **estendendo**
  `AppError` / `NotFoundError` / `ConflictError` / `ValidationError` de
  [src/shared/errors/http-errors.ts](../src/shared/errors/http-errors.ts) — exatamente o
  padrão de `InsufficientStockError` / `InvalidStatusTransitionError` (linhas 45‑63).
- Códigos `SCREAMING_SNAKE_CASE` com prefixo `WEBHOOK_` ([09:28] Bruno; [09:29] Larissa).
- Nada muda em [src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts):
  ele já trata `instanceof AppError` (linhas 15‑24), `ZodError` e Prisma ([09:29] Bruno).
- Opcional: re-exportar os novos erros; não é obrigatório tocar
  [src/shared/errors/index.ts](../src/shared/errors/index.ts) já que o módulo importa do
  próprio arquivo.

### 12.3 [src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts) — **reutilizar**

- Rotas de webhook usam `authenticate` (linha 27) como todas as outras.
- A rota de replay usa `requireRole('ADMIN')` (linhas 49‑61), montada no router igual a
  [src/modules/users/user.routes.ts](../src/modules/users/user.routes.ts) (linhas 12‑18):
  `router.post('/dead-letter/:id/replay', authenticate, requireRole('ADMIN'), validate({...}), controller.replay)`.
- `req.user.id` (linha 42) alimenta `webhook_dead_letter.replayedById` para auditoria
  ([09:36] Sofia).

### 12.4 [src/routes/index.ts](../src/routes/index.ts) e [src/app.ts](../src/app.ts) — **alterar**

- [src/routes/index.ts](../src/routes/index.ts): adicionar `webhooks: WebhookController` ao
  type `Controllers` (linhas 13‑19) e `router.use('/webhooks', buildWebhookRouter(...))` +
  `router.use('/admin/webhooks', buildWebhookAdminRouter(...))` (linhas 24‑28).
- [src/app.ts](../src/app.ts): em `buildControllers` (linhas 26‑53) instanciar
  `WebhookRepository` → `WebhookService` → `WebhookController` com o `prisma` recebido,
  seguindo o *wiring* manual já usado para `orders` (linhas 42‑44), e incluí-lo no objeto
  retornado (linhas 46‑52).

### 12.5 [src/config/database.ts](../src/config/database.ts) e novo `src/worker.ts` — **reutilizar / criar**

- `src/worker.ts` (novo): *entry-point* espelhando [src/server.ts](../src/server.ts) —
  importa `createPrismaClient` de [src/config/database.ts](../src/config/database.ts)
  (linha 4), instancia **seu próprio** `PrismaClient` ([09:30] Bruno), monta o
  `WebhookProcessor` e registra os *handlers* de `SIGINT`/`SIGTERM` como em
  [src/server.ts](../src/server.ts) (linhas 13‑21).
- [package.json](../package.json): novo script `"worker": "tsx watch --env-file=.env
  src/worker.ts"` (dev) e `"worker:start": "node --env-file=.env dist/worker.js"` (prod),
  espelhando `dev` / `start` (linhas 11‑14) ([09:11] Larissa — `npm run worker`).

### 12.6 [prisma/schema.prisma](../prisma/schema.prisma) — **alterar**

- Adicionar os 4 modelos + enum do §4 e a relação reversa em `model Customer`
  (linhas 40‑54). Rodar `npm run db:migrate` (linha 15 de [package.json](../package.json))
  para gerar `prisma/migrations/<timestamp>_webhooks/`.

### 12.7 [src/config/env.ts](../src/config/env.ts) e [.env.example](../.env.example) — **alterar**

- Acrescentar ao `envSchema` (linhas 3‑10) as variáveis do §4 com `z.coerce.number()` /
  `.default(...)`, no mesmo estilo de `PORT` / `JWT_EXPIRES_IN`.
- Replicar os defaults em [.env.example](../.env.example) (hoje linhas 1‑15).

### 12.8 [src/shared/logger/index.ts](../src/shared/logger/index.ts) — **alterar (mínimo)**

- Incluir `*.secret`, `*.secretCurrent`, `*.secretPrevious`,
  `req.headers['x-signature']` em `redactPaths` (linhas 4‑11) — a *secret* nunca pode
  vazar em log ([09:22] Diego relata caso real de vazamento de secret em log de cliente).

### 12.9 [tests/](../tests/) — **criar**

- `tests/webhooks.test.ts` e `tests/webhook-worker.test.ts` usando `getTestApp` /
  `bootstrapAuthenticatedUser` / `createTestCustomer` de
  [tests/helpers/factories.ts](../tests/helpers/factories.ts), no estilo de
  [tests/orders.test.ts](../tests/orders.test.ts). Um servidor HTTP *stub* local
  (`http.createServer`) recebe as entregas e permite asserção sobre headers, assinatura e
  retries.

---

## 13. Riscos e mitigação

| Risco | Origem | Impacto | Mitigação |
| --- | --- | --- | --- |
| `publishWebhookEvent` lento/pesado dentro da transação de `changeStatus` degrada a escrita de pedidos | [09:04] Bruno | Latência em todo o fluxo de pedidos | Só `SELECT` de endpoints + `INSERT`s simples; nenhum I/O externo; índice `webhook_endpoints(customerId, active)`; medir p95 de `changeStatus` antes/depois |
| Worker parado silenciosamente (crash, deadlock) | [09:11] Diego | Eventos param de sair, cliente volta a ficar "pendurado" | Alerta em `webhook_outbox_pending` (idade > 30 s); `webhook.worker_tick` em nível debug; recuperação de linhas `PROCESSING` órfãs (§8) |
| Crescimento indefinido de `webhook_outbox` / `webhook_delivery_attempts` | [09:08] Diego (retention fora de escopo) | Tabelas grandes, *scan* lento | Índice `(status, nextAttemptAt)`; abrir tarefa de *retention* (30 dias) como follow-up explícito |
| *Secret* vaza em log ou em resposta de listagem | [09:22] Diego (caso real) | Cliente pode ser falsificado | `redact` no Pino (§12.8); `secret` só em `POST`/`rotate-secret`; revisão de segurança da Sofia antes do deploy ([09:46]) |
| Single-worker vira gargalo de throughput | [09:12]‑[09:13] Diego | Latência acima de 10 s sob pico | `FOR UPDATE SKIP LOCKED` já preparado; abrir follow-up de particionamento por `order_id` |
| Cliente não implementa dedup por `X-Event-Id` | [09:25] Sofia | Pedido processado em duplicidade no cliente | Documentação destacada no portal do dev ([09:26] Marcos); `X-Event-Id` estável e explícito |
| Retry por até ~15 h mantém eventos "vivos" e pode entregar informação obsoleta | [09:17] | Cliente recebe transição antiga muito depois | Payload é snapshot com `timestamp` do evento; cliente compara com o estado atual via `GET /orders/:id` ([09:43]) |
| Resposta não-2xx do cliente (inclusive erro permanente) é sempre retentada até a 5ª tentativa | §6.11 | Tentativas desperdiçadas | Aceito nesta fase; as 5 tentativas limitam o desperdício; refinamento por código de status é melhoria futura |
| Janela de revisão de segurança no fim das 3 sprints atrasa o deploy | [09:46]‑[09:47] | Risco de prazo (fim de novembro) | Reservar 2 dias úteis para a Sofia; entregar HMAC + geração de secret já na 1ª sprint para revisão antecipada |

---

## 14. Sequência de implementação sugerida

1. **Migração + modelos** (§4, §12.6) e `env` (§12.7).
2. **Erros `WEBHOOK_*`** (§7, §12.2) + **schemas Zod**.
3. **Módulo CRUD** (`repository`/`service`/`controller`/`routes`) + *wiring* (§12.4) +
   `rotate-secret` (§5.6). Entregar HMAC/geração de secret aqui para revisão da Sofia.
4. **`publishWebhookEvent` + integração no `order.service`** (§5.1, §12.1) com testes de
   rollback.
5. **`src/worker.ts` + `WebhookProcessor`** (§5.2‑5.4, §12.5): polling, entrega, retry,
   DLQ.
6. **Endpoints `deliveries` e `admin/replay`** (§5.5, §6.7, §6.8).
7. **Observabilidade** (§9) e **testes de integração ponta a ponta** (§12.9).
8. **Revisão de segurança da Sofia** e ajustes ([09:46]).

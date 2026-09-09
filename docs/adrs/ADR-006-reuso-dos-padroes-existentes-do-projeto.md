# ADR-006-reuso-dos-padroes-existentes-do-projeto

## Status

`[Revisão]`

## Contexto

O time é pequeno ([09:07] Diego) e o prazo é de três sprints, incluindo a revisão
de segurança ([09:47] Larissa). A codebase já tem convenções claras e
consolidadas ([09:27] Bruno). Introduzir estruturas paralelas para o módulo de
webhooks aumentaria custo de manutenção e curva de revisão.

Padrões existentes identificados no código:

- **Módulos por domínio** em `src/modules/<dominio>` com `controller`, `service`,
  `repository`, `routes` e `schemas` — ex.:
  [src/modules/orders/](../../src/modules/orders/) e
  [src/routes/index.ts:21-31](../../src/routes/index.ts#L21-L31).
- **Erros**: classe base `AppError`
  ([src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts)) e
  classes específicas com código em `SCREAMING_SNAKE_CASE` como
  `INSUFFICIENT_STOCK` / `INVALID_STATUS_TRANSITION`
  ([src/shared/errors/http-errors.ts:45-63](../../src/shared/errors/http-errors.ts#L45-L63)).
- **Middleware de erro centralizado** que já trata `AppError`, `ZodError` e
  `Prisma.PrismaClientKnownRequestError`
  ([src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts)).
- **Logger Pino** único, com redação de campos sensíveis
  ([src/shared/logger/index.ts](../../src/shared/logger/index.ts)); `pino` e
  `pino-http` em [package.json](../../package.json).
- **Validação com Zod** via `validate({ body, query, params })`
  ([src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts)).
- **Autorização por papel** com `requireRole`
  ([src/middlewares/auth.middleware.ts:49-61](../../src/middlewares/auth.middleware.ts#L49-L61)).
- **Composition root** com injeção manual de dependências em
  [src/app.ts:26-53](../../src/app.ts#L26-L53).
- **`PrismaClient`** criado por `createPrismaClient`
  ([src/config/database.ts](../../src/config/database.ts)).

## Decisão

([09:30] Larissa — "Decisão: reuso máximo do que já existe. AppError, Pino, error
middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro.
Webhook fica como módulo igual aos outros.")

- **Novo módulo `src/modules/webhooks`** com `controller`, `service`,
  `repository`, `routes` e `schemas`, registrado em `src/routes/index.ts` como os
  demais ([09:27]–[09:28] Bruno; [09:28] Diego — "Faz.").
- A lógica de processamento do worker fica em um arquivo do módulo
  (`webhook.worker.ts` ou `webhook.processor.ts`); a entry-point `src/worker.ts`
  apenas inicializa ([09:28] Bruno; ver
  [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- **Erros do módulo** estendem `AppError`, com **todos os códigos prefixados por
  `WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`,
  `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno; [09:29] Larissa — "Prefixo WEBHOOK_
  pra tudo do módulo"). Nenhuma mudança no middleware de erro é necessária, pois
  ele já trata `AppError` ([09:29] Bruno).
- **Logger**: usa o Pino já existente; nada novo é adicionado ([09:29] Bruno).
- **Validação**: schemas Zod + `validate(...)`, incluindo a recusa de URL `http`
  (ver [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)).
- **Autorização**: o endpoint de replay da DLQ reaproveita `requireRole('ADMIN')`
  ([09:36] Larissa); o CRUD de configuração de webhook aceita qualquer papel
  autenticado por enquanto ([09:37] Sofia).
- **Persistência**: mesmo MySQL via Prisma. O worker instancia um `PrismaClient`
  próprio (mesma `DATABASE_URL`), porque `PrismaClient` é por processo
  ([09:30] Bruno).
- **Integração com pedidos**: função `publishWebhookEvent(tx, order, fromStatus,
  toStatus)` chamada de dentro de `OrderService.changeStatus`
  ([src/modules/orders/order.service.ts:126-179](../../src/modules/orders/order.service.ts#L126-L179)),
  recebendo o `tx` da transação — sem injetar o repository de webhooks no
  `OrderService` ([09:41] Bruno/Diego).

## Alternativas Consideradas

- **Criar uma estrutura/biblioteca dedicada para webhooks** (logger próprio,
  hierarquia de erros própria, camada de acesso a dados separada). Descartada em
  [09:30] por Larissa ("reuso máximo do que já existe") e em [09:07] pelo
  argumento geral de time pequeno / evitar overengineering.
- **Injetar o `WebhookRepository` inteiro no `OrderService`.** Descartada em
  [09:41] por Diego, em favor de uma função pura que recebe o `tx`
  ("Não precisa injetar repository inteiro").
- **Compartilhar a mesma instância de `PrismaClient` entre API e worker.**
  Inviável por serem processos distintos; decisão de instância separada em
  [09:30] por Bruno.
- **Endurecer desde já os papéis no CRUD de configuração de webhook.** Adiada em
  [09:37] por Sofia ("Por enquanto sim. Mais pra frente a gente pode endurecer").

## Consequências

**Positivas**

- Curva de revisão baixa: revisores reconhecem imediatamente a estrutura.
- Nenhuma dependência nova em [package.json](../../package.json).
- Erros do módulo entram automaticamente no formato de resposta padronizado do
  [error.middleware.ts](../../src/middlewares/error.middleware.ts).
- Consistência de observabilidade: um único formato de log Pino, com redação de
  segredos já configurada.

**Negativas / trade-offs**

- O módulo de webhooks fica acoplado às convenções atuais; uma futura mudança
  transversal (ex.: troca de ORM ou de framework de erros) o afeta junto.
- O padrão de módulo é orientado a request/response HTTP; o worker (loop de
  polling) não encaixa 100% nesse molde e exige o arquivo `webhook.worker.ts`
  fora do fluxo controller→service→repository.
- Reaproveitar o composition root manual de [src/app.ts](../../src/app.ts)
  significa que o `src/worker.ts` precisará de um wiring de dependências próprio e
  análogo, mantido em paralelo.

## Rastreabilidade

- Transcrição: [09:07], [09:27], [09:28], [09:29], [09:30], [09:36], [09:37],
  [09:41].
- Código: [src/modules/orders/](../../src/modules/orders/),
  [src/routes/index.ts](../../src/routes/index.ts),
  [src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts),
  [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts),
  [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts),
  [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts),
  [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts),
  [src/shared/logger/index.ts](../../src/shared/logger/index.ts),
  [src/config/database.ts](../../src/config/database.ts),
  [src/app.ts](../../src/app.ts),
  [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts).

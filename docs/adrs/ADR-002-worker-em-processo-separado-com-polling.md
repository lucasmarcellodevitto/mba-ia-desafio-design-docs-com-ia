# ADR-002-worker-em-processo-separado-com-polling

## Status

`[Revisão]`

## Contexto

O padrão Outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) apenas registra os eventos
na tabela `webhook_outbox`. É preciso um componente que leia esses eventos e
dispare as chamadas HTTP para os clientes.

Restrições e requisitos levantados na reunião:

- Latência aceitável pelos clientes: "abaixo de 10 segundos" ([09:02] Marcos).
- MySQL não tem mecanismo nativo de `LISTEN/NOTIFY` como o Postgres; uma trigger
  de banco só executa SQL e não consegue acordar um processo externo ([09:09] Diego).
- O componente de processamento não pode viver dentro da instância da API: se a
  API reinicia, o worker é perdido ([09:11] Diego).
- Hoje a API sobe por uma única entry-point,
  [src/server.ts](../../src/server.ts), que instancia o Prisma
  ([src/config/database.ts:10](../../src/config/database.ts#L10)) e chama
  `buildApp` ([src/app.ts:55](../../src/app.ts#L55)).
- O `PrismaClient` é por processo ([09:30] Bruno).

## Decisão

- **Worker em processo separado**, com uma nova entry-point `src/worker.ts` e um
  script `npm run worker`, análogo ao que já existe em `src/server.ts`
  ([09:11] Larissa). Mesmo banco, mesma `DATABASE_URL`, mesma stack — apenas
  processo diferente ([09:11] Diego).
- **Polling em loop, a cada 2 segundos**: busca os eventos pendentes mais antigos,
  processa, marca ([09:09] Diego; [09:10] Larissa — "Vamos registrar isso como uma
  decisão. Worker em polling, 2s"). Marcos confirmou que 2 segundos atende
  ([09:10]).
- A latência mínima de 2 segundos no pior caso é **explicitamente aceita**
  ([09:10] Larissa).
- O worker abre um **`PrismaClient` próprio** (nova instância, mesma
  `DATABASE_URL`), porque é outro processo Node ([09:30] Bruno). Reutiliza a
  função `createPrismaClient` de
  [src/config/database.ts:4](../../src/config/database.ts#L4).
- A lógica de processamento fica em um arquivo dentro do módulo de webhooks
  (ex.: `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`),
  mantendo `src/worker.ts` apenas como entry-point ([09:28] Bruno).
- **Enquanto houver um único worker**, os eventos são processados em ordem de
  `created_at` da outbox, o que dá ordenação por `order_id` para o cliente. Não há
  garantia de ordenação global; isso é registrado como **limitação conhecida**
  ([09:12]–[09:13] Diego; [09:13] Larissa). Os clientes nunca pediram ordenação
  global ([09:14] Marcos).

## Alternativas Consideradas

- **Trigger de banco para notificar o worker (em vez de polling).** Descartada em
  [09:09] por Diego; alternativa levantada por Bruno em [09:09]. Motivo: MySQL não
  tem `LISTEN/NOTIFY`; a trigger só executa SQL e teria que "improvisar" escrita
  em arquivo ou chamada a endpoint, o que ficaria frágil. Polling de 2s já atende
  o requisito de <10s.
- **Rodar o processamento dentro da própria instância da API.** Descartada em
  [09:11] por Diego: se a API reinicia, perde-se o worker.
- **Múltiplos workers em paralelo desde já** (particionamento por `order_id` ou
  lock pessimista). Adiada em [09:13] por Diego ("problema do futuro, não agora"),
  porque quebraria a ordenação implícita por `order_id`.
- **Intervalo de polling menor (sub-segundo)** — alternativa plausível não
  discutida explicitamente: rejeitada implicitamente pelo custo de carga no MySQL
  sem benefício, já que 2s satisfaz o SLA de 10s informado por Marcos ([09:02]).

## Consequências

**Positivas**

- Sobrevive a restart da API; deploy e escala do worker são independentes.
- Implementação simples: um loop de polling, sem broker nem dependências novas.
- Ordenação por `order_id` "de graça" enquanto for single-worker.

**Negativas / trade-offs**

- Adiciona até ~2 segundos de latência no pior caso, além do tempo da chamada HTTP
  ([09:10] Larissa).
- Polling constante gera carga fixa no MySQL mesmo sem eventos pendentes.
- Single-worker é um ponto único de processamento e um gargalo de throughput;
  escalar horizontalmente exigirá particionamento por `order_id` ou lock
  pessimista, trabalho adiado ([09:13] Diego).
- Nova entry-point e novo script de operação (`npm run worker`) para manter,
  monitorar e implantar.

## Rastreabilidade

- Transcrição: [09:02], [09:09], [09:10], [09:11], [09:12], [09:13], [09:14],
  [09:28], [09:30].
- Código: [src/server.ts](../../src/server.ts),
  [src/config/database.ts](../../src/config/database.ts),
  [src/app.ts](../../src/app.ts),
  [package.json](../../package.json) (bloco `scripts`).

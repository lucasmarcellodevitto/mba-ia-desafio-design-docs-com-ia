# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) — responsável por abrir o doc de design da feature ([09:50]) |
| **Status** | Em revisão |
| **Data** | 2026-09-09 |
| **Origem** | Reunião técnica de quinta-feira, 09:00 (~55 min) — ver [TRANSCRICAO.md](../TRANSCRICAO.md) |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno — time de Pedidos), Diego (Engenheiro Sênior — time de Plataforma), Sofia (Engenheira de Segurança) |


---

## Resumo executivo (TL;DR)

Propomos entregar notificações **outbound** de mudança de status de pedidos para clientes
B2B via webhooks HTTP. Quando o status de um pedido muda, um evento é gravado numa tabela
**outbox** dentro da **mesma transação** que atualiza o pedido; um **worker em processo
separado** faz *polling* dessa tabela a cada 2 segundos e entrega os eventos por HTTP, com
**retry em backoff exponencial** (5 tentativas) e **dead-letter queue** para falhas
permanentes. As requisições são autenticadas com **HMAC-SHA256** e *secret* única por
endpoint. A garantia de entrega é **at-least-once**, com `X-Event-Id` para deduplicação no
cliente. A feature reaproveita ao máximo os padrões já existentes na codebase (módulos em
`src/modules`, `AppError`, Pino, middleware de erro, validação Zod, `requireRole`).
Estimativa: **três sprints**, com revisão de segurança incluída ao final.

---

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para
ser notificados em tempo real quando o status dos pedidos deles muda na plataforma
([09:00] Marcos). Hoje eles fazem *polling* sobre `GET /orders`, o que deixa a integração
lenta e cara. A Atlas sinalizou risco de migração para um concorrente caso a entrega não
ocorra até o fim do trimestre ([09:00] Marcos). "Tempo real", para eles, significa
**latência abaixo de 10 segundos**; o essencial é não depender de atualização manual
([09:02] Marcos).

O escopo é **apenas outbound** — a plataforma envia, os clientes recebem ([09:02] Marcos;
[09:03] Sofia).

Restrições técnicas relevantes:

- A transação de mudança de status já é pesada (atualiza o pedido, insere no histórico,
  ajusta estoque). Um HTTP *call* síncrono no meio dela travaria mudanças de status de
  outros pedidos quando um cliente estivesse lento, e a indisponibilidade do cliente
  exigiria *rollback* do negócio — inaceitável ([09:04] Bruno).
- O time é pequeno; subir infraestrutura nova (ex.: Redis Cluster) para isso é
  considerado *overengineering* ([09:07] Diego).
- O banco é MySQL, que não possui `LISTEN/NOTIFY` nativo ([09:09] Diego).
- Os eventos carregam dados de pedidos e trafegam para fora da nossa infraestrutura,
  exigindo autenticidade e integridade verificáveis pelo cliente ([09:19] Sofia).
- Prazo de negócio: a Atlas quer a entrega para o fim de novembro ([09:45] Marcos).

---

## Proposta técnica

### Visão geral

1. **Padrão Outbox no MySQL.** Na mesma transação SQL que muda o status do pedido,
   gravamos o evento (já renderizado — *snapshot*) numa tabela `webhook_outbox`. Se a
   transação principal sofre *rollback*, o evento desaparece junto; se comita, o evento
   está garantidamente registrado ([09:06]–[09:08] Diego; [09:52] snapshot).

2. **Worker em processo separado.** Uma nova *entry-point* (`src/worker.ts` + `npm run
   worker`), no mesmo banco e mesma stack, mas fora do processo da API. Faz *polling* dos
   eventos pendentes a cada **2 segundos** e dispara as chamadas HTTP. A latência mínima de
   2 segundos no pior caso é aceita ([09:10]–[09:11]). Enquanto for *single-worker*, os
   eventos são processados em ordem de criação, garantindo ordenação **por `order_id`**
   (não global) ([09:12]–[09:13]).

3. **Retry e DLQ.** Falha de entrega dispara *retry* com **backoff exponencial**
   (1m / 5m / 30m / 2h / 12h), até **5 tentativas**; depois disso, o evento vai para uma
   tabela `webhook_dead_letter`. Há um endpoint administrativo de *replay* manual,
   restrito a `ADMIN` e auditado ([09:15]–[09:19], [09:36]). *Timeout* HTTP de 10 segundos
   ([09:42]).

4. **Segurança.** Assinatura **HMAC-SHA256** sobre o corpo do request (`X-Signature`),
   **secret única por endpoint** (não global), com **rotação** via API e *grace period* de
   24h para a secret antiga. URL do webhook obrigatoriamente `https` ([09:20]–[09:23]).

5. **Garantia de entrega at-least-once.** O cliente pode receber o mesmo evento mais de
   uma vez e deduplica pelo `X-Event-Id` (UUID gerado na inserção do evento). Não
   perseguimos *exactly-once* — é o padrão de mercado (Stripe, GitHub) ([09:24]–[09:26]).

6. **Reuso dos padrões do projeto.** Webhook entra como um módulo em
   `src/modules/webhooks`, com erros estendendo `AppError` e códigos prefixados por
   `WEBHOOK_`, logger Pino existente, middleware de erro atual, validação Zod e
   `requireRole` já disponíveis. Nenhuma dependência nova ([09:27]–[09:30]).

### Superfície funcional (resumo)

CRUD de configuração de webhook por cliente (criar com *secret* devolvida na criação,
editar, remover, listar), com filtro dos status desejados aplicado **na inserção** da
outbox; endpoint de histórico de entregas; endpoint admin de *replay* de DLQ. O
`customer_id` é passado no corpo/rota, **não** derivado do JWT (o JWT atual é do usuário
operador) ([09:31]–[09:37]). O detalhamento de rotas, campos e contrato fica no FDD.

### Estimativa

Três sprints, incluindo a revisão de segurança da Sofia ao final (reservar ao menos dois
dias úteis para revisão de HMAC e geração de *secret* antes do deploy) ([09:46]–[09:47]).

---

## Alternativas consideradas

| Alternativa | Discutida em | Trade-off que levou ao descarte |
| --- | --- | --- |
| **Disparo HTTP síncrono** dentro do service de pedidos, no `changeStatus` | [09:04]–[09:06] (Bruno, Diego) | A transação de status já é pesada; um cliente lento travaria mudanças de status de outros pedidos, e cliente offline exigiria *rollback* da mudança de status. Sem isolamento entre o caminho crítico de escrita e a entrega. |
| **Fila em Redis Streams / Redis Cluster** | [09:07] (Diego; alternativa de Larissa) | Exige subir e operar infraestrutura nova para um time pequeno. A outbox no MySQL existente resolve sem custo operacional adicional — *overengineering*. |
| **Trigger de banco** para acordar o worker (em vez de *polling*) | [09:09] (Diego; sugestão de Bruno) | MySQL não tem `LISTEN/NOTIFY`; a trigger só executa SQL e teria que "improvisar" (arquivo, endpoint), ficando frágil. *Polling* de 2s já cumpre o SLA de <10s. |
| **3 tentativas de retry** (mais agressivo) | [09:16] (Diego; proposta de Bruno) | Com backoff, cobriria só ~30 min e mataria eventos de clientes com indisponibilidade matinal; já houve cliente com 2h de manutenção planejada. 5 tentativas cobrem ~15h. |
| **DLQ como flag `failed` na própria outbox** (sem tabela separada) | [09:17]–[09:18] (Diego; opção de Larissa) | Polui a leitura da outbox principal; a tabela dedicada serve melhor como trilha de evidência e base de reprocessamento. |
| **Garantia exactly-once** | [09:25] (Diego; observação de Sofia) | Exigiria coordenação entre os dois lados e complexidade muito maior. *At-least-once* + `event_id` resolve 99% dos casos, ao custo de empurrar a deduplicação para o cliente. |
| **Secret global da plataforma** | [09:21] (Sofia) | "Se vaza uma, vaza tudo." *Secret* por endpoint contém o raio de impacto de um vazamento. |

---

## Questões em aberto

1. **Escalar para múltiplos workers em paralelo.** Adiado — "problema do futuro, não
   agora" ([09:13] Diego). Hoje assumimos *single-worker*; particionamento por `order_id`
   ou *lock* pessimista fica para quando a escala exigir. A ordenação garantida é apenas
   por `order_id` e enquanto houver um único worker.

2. **Rate limiting de envio para o cliente.** Se um cliente tem 50 pedidos mudando de
   status em um minuto, hoje enviamos 50 chamadas. Decisão: "observar e decidir depois" —
   implementar se virar problema ([09:38]–[09:39]).

3. **Aviso proativo ao cliente sobre webhook com falha** (ex.: e-mail após N falhas
   consecutivas). Fora do escopo desta fase; possível próxima fase, após medir o impacto
   ([09:37] Larissa).

4. **Endurecimento de papéis no CRUD de configuração de webhook.** Por ora qualquer papel
   autenticado pode operar o CRUD; "mais pra frente a gente pode endurecer" ([09:37] Sofia).
   O endpoint de *replay* de DLQ já nasce restrito a `ADMIN`.

5. **Arquivamento das linhas já entregues da outbox** (ex.: após 30 dias). Mencionado como
   necessário, mas explicitamente fora do escopo desta feature ([09:08] Diego).

---

## Impacto e riscos

**Impacto**

- **Alteração no caminho crítico de pedidos:** `OrderService.changeStatus` passará a
  gravar o evento na outbox dentro da transação existente. Se a inserção falhar, a mudança
  de status inteira sofre *rollback* ([09:40]–[09:41]). É o ponto de integração mais
  sensível.
- **Novo processo operacional:** o worker (`npm run worker`) precisa ser implantado,
  monitorado e reiniciado de forma independente da API.
- **Novas tabelas** (`webhook_outbox`, `webhook_dead_letter`, configuração de webhook,
  histórico de entregas) e **novo módulo** `src/modules/webhooks`.
- **Contrato público** de payload e headers, a ser documentado por Marcos no portal do
  desenvolvedor ([09:40]).

**Riscos**

- **Latência acumulada:** até ~2s de *polling* + tempo da chamada HTTP; em cenário de
  *retry*, um evento pode levar até ~15h para ser considerado falha permanente
  ([09:17]). Aceito pelo grupo, mas é um risco de percepção para o cliente.
- **Single-worker como gargalo e ponto único** de processamento; escala e ordenação
  global ficam limitadas até o trabalho adiado de particionamento.
- **Gestão de *secret*:** a secret precisa ser recuperável para recalcular o HMAC no
  envio, ampliando a superfície de proteção de dado em repouso. Mitigação: revisão de
  segurança dedicada da Sofia antes do deploy ([09:46]).
- **Deduplicação delegada ao cliente:** um cliente que não implemente a checagem de
  `X-Event-Id` pode processar o mesmo pedido duas vezes ([09:25] Sofia).
- **Crescimento da outbox** sem rotina de arquivamento definida nesta fase ([09:08]).
- **Prazo apertado** (fim de novembro / três sprints) com dependência da janela de
  revisão de segurança no fim ([09:45]–[09:47]).

---

## Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox sobre o MySQL existente, na mesma transação de `changeStatus` |
| [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado (`src/worker.ts`) com *polling* de 2s |
| [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com *secret* por endpoint e rotação com *grace period* de 24h |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` para deduplicação no cliente |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto (módulos, `AppError`, Pino, Zod, `requireRole`) |
| [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md) | Formato de payload (*snapshot*), headers `X-*`, *timeout* de 10s e limite de 64 KB |


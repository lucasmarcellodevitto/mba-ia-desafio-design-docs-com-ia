# ADR-003-retry-com-backoff-e-dead-letter-queue

## Status

`[Revisado]`

## Contexto

O worker ([ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)) dispara
chamadas HTTP para endpoints fora da nossa infraestrutura. O cliente pode estar
offline no momento do envio ([09:14] Larissa). Já houve caso de cliente com
indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

É preciso decidir:

- o que fazer quando a entrega falha;
- quantas vezes retentar e com qual espaçamento;
- o que acontece com o evento após o esgotamento das tentativas;
- como reprocessar manualmente um evento definitivamente falho.

O projeto já tem um mecanismo de autorização por papel:
`requireRole('ADMIN' | 'OPERATOR')` em
[src/middlewares/auth.middleware.ts:49-61](../../src/middlewares/auth.middleware.ts#L49-L61),
e os papéis são definidos no enum `UserRole`
([prisma/schema.prisma:11-14](../../prisma/schema.prisma#L11-L14)).

## Decisão

- **Retry com backoff exponencial**, com teto de **5 tentativas**; após o teto, o
  evento é considerado falha permanente ([09:15] Diego; [09:17] Larissa —
  "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").
- **Progressão do backoff:** 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas —
  aproximadamente 15 horas entre a primeira falha e a última tentativa
  ([09:17] Diego). Marcos considerou aceitável ([09:17]).
- **Timeout da chamada HTTP: 10 segundos.** Cliente que não responde em 10s é
  tratado como falha e marcado para retry ([09:42] Diego; [09:42] Sofia).
- **Dead-letter queue em tabela separada `webhook_dead_letter`**, contendo a
  payload, o motivo da falha e o timestamp ([09:18] Diego). Mantém a leitura da
  `webhook_outbox` principal limpa e serve como evidência para debug e
  reprocessamento.
- **Reprocessamento manual** via endpoint administrativo
  `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na
  `webhook_outbox` como pendente ([09:18] Diego; [09:35] Diego).
- O endpoint de replay **exige papel `ADMIN`**, reaproveitando o `requireRole`
  existente, e deve **registrar em log quem executou o replay**, para auditoria
  ([09:36] Sofia; [09:36] Larissa — "Decidido, role ADMIN obrigatório no replay e
  a gente reaproveita o requireRole que já existe").

## Alternativas Consideradas

- **3 tentativas de retry** (mais agressivo). Descartada em [09:16] por Diego
  ("3 é pouco"); proposta por Bruno em [09:16]. Motivo: com backoff, 3 tentativas
  cobririam apenas ~30 minutos e matariam eventos de clientes com indisponibilidade
  de manhã; já houve cliente com 2h de indisponibilidade planejada.
- **Retry indefinido com backoff.** Descartada em [09:15] por Diego: o evento
  ficaria pendurado para sempre se o cliente sumisse.
- **DLQ como flag `failed` na própria `webhook_outbox`** (sem tabela separada).
  Descartada em [09:18] por Diego; opção levantada por Larissa em [09:17]. Motivo:
  tabela separada mantém a outbox principal enxuta e serve melhor como trilha de
  evidência e reprocessamento.
- **Replay acessível a qualquer papel autenticado** — alternativa implícita
  (o CRUD de configuração de webhook seguirá esse modelo, [09:37] Sofia).
  Descartada para o replay em [09:36] por Sofia: "mexer em fila de entrega de
  notificação não é coisa de operador".

## Consequências

**Positivas**

- Tolera indisponibilidades reais de clientes (janela de ~15h) sem intervenção.
- Eventos irrecuperáveis não poluem a outbox e ficam auditáveis na
  `webhook_dead_letter`.
- Operação de recuperação é explícita, restrita a `ADMIN` e auditada.

**Negativas / trade-offs**

- Um evento pode levar até ~15 horas para ser considerado definitivamente falho —
  aceito por Marcos ([09:17]), mas significa entrega potencialmente muito atrasada.
- Retries ocupam ciclos do worker e mantêm linhas "vivas" na outbox por longos
  períodos.
- O reprocessamento é **manual**: não há re-tentativa automática após a DLQ, nem
  alerta proativo ao cliente sobre webhooks com problema — o aviso por e-mail foi
  colocado fora do escopo desta fase ([09:37] Larissa).
- Nova tabela (`webhook_dead_letter`) e novo endpoint administrativo para manter.

## Rastreabilidade

- Transcrição: [09:14], [09:15], [09:16], [09:17], [09:18], [09:35], [09:36],
  [09:37], [09:42].
- Código: [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts),
  [prisma/schema.prisma](../../prisma/schema.prisma).

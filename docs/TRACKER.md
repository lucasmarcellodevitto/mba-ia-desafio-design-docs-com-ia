# TRACKER — Rastreabilidade da Documentação

Referência cruzada entre cada item registrado nos documentos da feature **Sistema de
Webhooks de Notificação de Pedidos** e sua origem: a [transcrição da reunião
técnica](../TRANSCRICAO.md) (`TRANSCRICAO`, com `[hh:mm] Falante`) ou o **código-fonte**
da aplicação (`CODIGO`, com o caminho do arquivo).

Serve para: (1) permitir a qualquer leitor entender de onde veio cada decisão, requisito
ou restrição; (2) verificar que a documentação está alinhada ao que foi efetivamente
discutido e ao que existe no código.

> Este documento não é um artefato padrão de mercado — é uma exigência específica desta
> atividade.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cliente cadastra webhook (POST) com URL e lista de status; secret devolvida só na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | `customer_id` vem no corpo/rota, não do JWT (JWT é do usuário operador) | TRANSCRICAO | [09:32] Larissa |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Editar (PATCH), remover (DELETE) e listar webhooks por customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Cada webhook define a lista de status que quer receber; filtro na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Evento registrado na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Entrega por POST HTTP com os dados essenciais do pedido, sem a lista de itens | TRANSCRICAO | [09:43] Diego |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Entrega assinada e identificada para verificação de autenticidade e dedup pelo cliente | TRANSCRICAO | [09:44] Diego |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Retry automático de falhas, cobrindo janela de indisponibilidade de ~15 h antes de desistir | TRANSCRICAO | [09:17] Larissa |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Após esgotar os retries, evento vai para dead-letter queue dedicada | TRANSCRICAO | [09:18] Diego |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Endpoint admin de replay manual da DLQ; exige ADMIN e registra o autor | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Cliente consulta histórico de entregas (status, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Cliente rotaciona a secret; a anterior fica válida por 24 h em paralelo | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | URL do webhook deve ser `https`; `http` é recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RF-14 | docs/PRD.md | Requisito Funcional | Conteúdo entregue reflete o estado do pedido no momento da transição | TRANSCRICAO | [09:52] Larissa |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega p95 < 10 s no caminho feliz | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Entrega não pode travar/atrasar a mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Worker roda em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Entregas assinadas (HMAC-SHA256); secret única por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Timeout de entrega de 10 s tratado como falha | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB; acima disso, erro (não trunca) | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Semântica at-least-once; cliente é responsável por deduplicar | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Ordenação por `order_id` enquanto single-worker; sem garantia global | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Reuso dos padrões do projeto; sem dependências novas | TRANSCRICAO | [09:30] Larissa |
| PRD-RNF-11 | docs/PRD.md | Requisito Não Funcional | Replay da DLQ registra o usuário que o executou | TRANSCRICAO | [09:36] Sofia |
| PRD-RNF-12 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança obrigatória antes do deploy (≥ 2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo / Métrica | Notificar em tempo quase real — meta p95 < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo / Métrica | Eliminar o polling dos 3 clientes-alvo sobre `GET /orders` | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo / Métrica | Não perder eventos — 0 mudanças commitadas sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo / Métrica | Tolerar indisponibilidade do cliente por ~15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo / Métrica | Entregar até o fim de novembro (~3 sprints) | TRANSCRICAO | [09:45] Marcos |
| PRD-OOS-01 | docs/PRD.md | Restrição (fora de escopo) | Webhooks inbound (cliente → plataforma) | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-02 | docs/PRD.md | Restrição (adiado) | Aviso por e-mail ao cliente quando o webhook falha | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-03 | docs/PRD.md | Restrição (adiado) | Rate limiting de saída por cliente | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Restrição (fora de escopo) | Dashboard / painel visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-05 | docs/PRD.md | Restrição (fora de escopo) | Arquivamento das linhas entregues da outbox (retention) | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-06 | docs/PRD.md | Restrição (adiado) | Múltiplos workers em paralelo / ordering global | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-07 | docs/PRD.md | Restrição (adiado) | Endurecer papéis no CRUD de configuração | TRANSCRICAO | [09:37] Sofia |
| PRD-OOS-08 | docs/PRD.md | Restrição (fora de escopo) | Garantia de entrega exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Worker parado silenciosamente interrompe as entregas | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret permite forjar chamadas | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Processamento por um único worker vira gargalo sob pico | TRANSCRICAO | [09:13] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Alteração no caminho crítico da mudança de status degrada pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-RISK-05 | docs/PRD.md | Risco | Cliente não implementa a dedup e processa o mesmo evento duas vezes | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-06 | docs/PRD.md | Risco | Prazo apertado com a revisão de segurança concentrada no fim | TRANSCRICAO | [09:46] Sofia |
| PRD-PER-01 | docs/PRD.md | Público-alvo | Clientes B2B: Atlas Comercial, MaxDistribuição, Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-PER-02 | docs/PRD.md | Público-alvo | Cliente chama a API autenticado com um usuário do nosso sistema | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-03 | docs/PRD.md | Público-alvo | Operador de plataforma (ADMIN) reprocessa a DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-PER-04 | docs/PRD.md | Público-alvo | Marcos documenta o contrato no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Fluxo de mudança de status de pedido (ponto de integração) | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Banco de dados e ORM existentes (MySQL / Prisma) | CODIGO | prisma/schema.prisma |
| PRD-DEP-03 | docs/PRD.md | Dependência | Autenticação e autorização existentes (JWT, papéis) | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-04 | docs/PRD.md | Dependência | Novo processo worker no pipeline de deploy | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-05 | docs/PRD.md | Dependência | Portal do desenvolvedor para documentar o contrato | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-06 | docs/PRD.md | Dependência | Revisão de segurança da Sofia como gate antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Entrega assíncrona com ~2 s de latência de base em troca de não sobrecarregar pedidos | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-02 | docs/PRD.md | Trade-off | At-least-once: cliente pode receber duplicado e precisa deduplicar | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Recuperação de falhas permanentes é manual, sem aviso automático ao cliente | TRANSCRICAO | [09:18] Diego |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Ordenação garantida apenas por pedido, não global | TRANSCRICAO | [09:13] Larissa |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Contrato de payload/headers é interface pública versionável | TRANSCRICAO | [09:40] Marcos |
| PRD-CTX-01 | docs/PRD.md | Restrição | Escopo apenas outbound (plataforma envia, cliente recebe) | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Atlas sinalizou risco de migrar para concorrente sem a entrega no prazo | TRANSCRICAO | [09:00] Marcos |
| RFC-PROP-01 | docs/RFC.md | Decisão (proposta) | Padrão Outbox no MySQL, na transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| RFC-PROP-02 | docs/RFC.md | Decisão (proposta) | Worker em processo separado com polling | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão (proposta) | Retry com backoff + DLQ + replay administrativo | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão (proposta) | Assinatura HMAC, secret por endpoint, URL https | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão (proposta) | Entrega at-least-once com deduplicação pelo cliente | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | docs/RFC.md | Decisão (proposta) | Reuso dos padrões do projeto, sem dependências novas | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off (alternativa descartada) | Disparo HTTP síncrono no `changeStatus` — descartado | TRANSCRICAO | [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off (alternativa descartada) | Fila em Redis Streams / Cluster — descartado (overengineering) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off (alternativa descartada) | Trigger de banco para acordar o worker — descartado | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off (alternativa descartada) | 3 tentativas de retry — descartado ("3 é pouco") | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Trade-off (alternativa descartada) | DLQ como flag na própria outbox — descartado | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-06 | docs/RFC.md | Trade-off (alternativa descartada) | Garantia exactly-once — descartado (complexidade / coordenação) | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Trade-off (alternativa descartada) | Secret global da plataforma — descartado ("vaza uma, vaza tudo") | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (particionar por `order_id`) | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Rate limiting de envio ao cliente | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Aviso proativo ao cliente (e-mail) sobre webhook com falha | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Endurecer papéis no CRUD de configuração | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | Arquivamento das linhas entregues da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-01 | docs/RFC.md | Risco | Latência inerente ao polling + entrega tardia em retry (~15 h) | TRANSCRICAO | [09:17] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Secret precisa ser recuperável para recalcular o HMAC no envio | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Deduplicação depende do cliente | TRANSCRICAO | [09:25] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | `changeStatus` passa a gravar o evento na transação; falha na inserção causa rollback | TRANSCRICAO | [09:41] Diego |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novo processo operacional (worker) a implantar e monitorar | TRANSCRICAO | [09:11] Diego |
| RFC-CTX-01 | docs/RFC.md | Restrição | Escopo apenas outbound | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-02 | docs/RFC.md | Restrição | Time pequeno — evitar infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| RFC-CTX-03 | docs/RFC.md | Restrição | MySQL não possui `LISTEN/NOTIFY` nativo | TRANSCRICAO | [09:09] Diego |
| RFC-CTX-04 | docs/RFC.md | Restrição | Prazo de negócio: entrega para o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox sobre o MySQL existente, na transação de `changeStatus` | TRANSCRICAO | [09:08] Larissa |
| ADR-001-A | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Evento inserido na mesma transação SQL; rollback remove o evento junto | TRANSCRICAO | [09:06] Diego |
| ADR-001-B | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Chave primária da outbox = UUID (padrão do projeto) | TRANSCRICAO | [09:51] Larissa |
| ADR-001-C | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Integração via função `publishWebhookEvent(tx, ...)`, sem injetar repository | TRANSCRICAO | [09:41] Bruno |
| ADR-001-D | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Banco de dados atual é MySQL | CODIGO | prisma/schema.prisma |
| ADR-001-E | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | `changeStatus` já roda em `$transaction` (order + history + estoque) | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-A | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker fora do processo da API (senão perde o worker no restart) | TRANSCRICAO | [09:11] Diego |
| ADR-002-B | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker abre um `PrismaClient` próprio (é por processo) | TRANSCRICAO | [09:30] Bruno |
| ADR-002-C | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição | Ordenação por `order_id` só enquanto for single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-D | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Polling de 2 s atende o SLA de <10 s | TRANSCRICAO | [09:10] Marcos |
| ADR-002-E | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Nova entry-point `src/worker.ts` + script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Retry com backoff, 5 tentativas, depois DLQ | TRANSCRICAO | [09:17] Larissa |
| ADR-003-A | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Progressão do backoff: 1m / 5m / 30m / 2h / 12h | TRANSCRICAO | [09:17] Diego |
| ADR-003-B | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Timeout da chamada HTTP: 10 segundos | TRANSCRICAO | [09:42] Diego |
| ADR-003-C | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | DLQ em tabela separada `webhook_dead_letter` (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| ADR-003-D | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Replay via `POST /admin/webhooks/dead-letter/:id/replay`, recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| ADR-003-E | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Replay exige role ADMIN e registra o autor (auditoria) | TRANSCRICAO | [09:36] Sofia |
| ADR-003-F | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | 3 tentativas descartadas ("3 é pouco") | TRANSCRICAO | [09:16] Diego |
| ADR-003-G | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | Retry indefinido descartado (evento ficaria pendurado para sempre) | TRANSCRICAO | [09:15] Diego |
| ADR-003-H | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Restrição | `requireRole` / enum `UserRole` já existem no projeto | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace period de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-A | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Algoritmo HMAC-SHA256, header `X-Signature` | TRANSCRICAO | [09:20] Sofia |
| ADR-004-B | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint (não global) | TRANSCRICAO | [09:21] Sofia |
| ADR-004-C | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Tabela de configuração armazena url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | TLS obrigatório (validação no schema Zod) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-E | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Requisito Não Funcional | Limite de payload de 64 KB, com erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| ADR-004-F | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret global da plataforma descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-004-G | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Truncar payload acima do limite descartado (Sofia é a favor de erro) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-H | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Contexto | Já houve cliente que vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com `X-Event-Id` para dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-A | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | `X-Event-Id` = UUID gerado quando o evento entra na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-B | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado (coordenação/complexidade; padrão Stripe/GitHub) | TRANSCRICAO | [09:25] Diego |
| ADR-005-C | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Comportamento documentado de forma destacada no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-D | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Restrição | Projeto já usa UUID como identificador padrão | CODIGO | prisma/schema.prisma |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes; webhook entra como módulo | TRANSCRICAO | [09:30] Larissa |
| ADR-006-A | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Módulo `src/modules/webhooks` (controller/service/repository/routes/schemas) | TRANSCRICAO | [09:27] Bruno |
| ADR-006-B | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Lógica do worker em arquivo do módulo; entry-point `src/worker.ts` | TRANSCRICAO | [09:28] Bruno |
| ADR-006-C | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Erros estendem `AppError`; prefixo `WEBHOOK_` em todos os códigos | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Logger Pino e middleware de erro reaproveitados sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-E | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Padrão de módulos por domínio já existe na codebase | CODIGO | src/routes/index.ts |
| ADR-006-F | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Classe base `AppError` e classes específicas com código | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-G | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Middleware de erro central já trata `AppError`, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Decisão | Payload snapshot, headers `X-*`, timeout 10 s, limite 64 KB | TRANSCRICAO | [09:43] Diego |
| ADR-007-A | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Decisão | Snapshot na inserção (não renderizar no envio) | TRANSCRICAO | [09:52] Larissa |
| ADR-007-B | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Decisão | Filtragem de status na inserção da outbox (não no envio) | TRANSCRICAO | [09:34] Bruno |
| ADR-007-C | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Decisão | Payload sem `items` (cliente busca em `GET /orders/:id`) | TRANSCRICAO | [09:43] Diego |
| ADR-007-D | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Decisão | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| ADR-007-E | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Decisão | Header `X-Webhook-Id` (para cliente com vários endpoints) | TRANSCRICAO | [09:44] Sofia |
| ADR-007-F | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Trade-off | Guardar só `order_id` e renderizar no envio — descartado | TRANSCRICAO | [09:52] Larissa |
| ADR-007-G | docs/adrs/ADR-007-formato-de-payload-headers-e-limites.md | Restrição | Campos da order correspondem a colunas do modelo `Order` | CODIGO | prisma/schema.prisma |
| FDD-DATA-01 | docs/FDD.md | Decisão (implementação) | Tabela `webhook_outbox` (evento, status, tentativas, `nextAttemptAt`) | TRANSCRICAO | [09:06] Diego |
| FDD-DATA-02 | docs/FDD.md | Decisão (implementação) | Índices da outbox por status e por `created_at` | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | docs/FDD.md | Decisão (implementação) | Tabela `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-04 | docs/FDD.md | Decisão (implementação) | Tabela `webhook_endpoints` (url, secret, customerId, ativo, status assinados) | TRANSCRICAO | [09:21] Bruno |
| FDD-DATA-05 | docs/FDD.md | Decisão (implementação) | Tabela `webhook_delivery_attempts` para o histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-06 | docs/FDD.md | Restrição | PK `uuid @db.Char(36)` segue o padrão de todos os modelos | CODIGO | prisma/schema.prisma |
| FDD-DATA-07 | docs/FDD.md | Decisão (implementação) | Variáveis `WEBHOOK_*` acrescentadas ao schema de ambiente | CODIGO | src/config/env.ts |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da transação de `changeStatus` | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Filtra endpoints pelo status inscrito; se nenhum, não insere | TRANSCRICAO | [09:34] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Worker: loop de polling buscando os pendentes mais antigos | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Entrega HTTP assinada com HMAC no header `X-Signature` | TRANSCRICAO | [09:20] Sofia |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Retry até o teto de tentativas; depois move para a DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | Replay recoloca o item da DLQ como pendente na outbox | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-07 | docs/FDD.md | Fluxo | Rotação de secret: anterior válida por 24 h, depois expira | TRANSCRICAO | [09:21] Sofia |
| FDD-FLOW-08 | docs/FDD.md | Fluxo | Worker com shutdown gracioso, espelhando o padrão da entry-point atual | CODIGO | src/server.ts |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato (API) | `POST /api/v1/webhooks` — criar endpoint | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato (API) | `GET /api/v1/webhooks` — listar por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato (API) | `PATCH /api/v1/webhooks/:id` — editar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato (API) | `DELETE /api/v1/webhooks/:id` — remover | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato (API) | `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato (API) | `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato (API) | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato (payload) | JSON com `event_id`, `event_type`, `timestamp`, dados da order, `from/to_status` | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato (headers) | `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato (API) | `customer_id` no corpo; endpoint autenticado | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato (autorização) | CRUD: qualquer papel autenticado; replay: ADMIN | TRANSCRICAO | [09:37] Sofia |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato (formato de erro) | Resposta `{ error: { code, message, details? } }` reaproveitada | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato (paginação) | Listas no formato `{ data, pagination }` do projeto | CODIGO | src/shared/http/response.ts |
| FDD-ERR-01 | docs/FDD.md | Tratamento de erro | Código `WEBHOOK_NOT_FOUND` (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Tratamento de erro | Código `WEBHOOK_INVALID_URL` (400) — https obrigatório | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Tratamento de erro | Código `WEBHOOK_SECRET_REQUIRED` (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Tratamento de erro | Classes estendem `AppError` seguindo o padrão de `http-errors` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-05 | docs/FDD.md | Tratamento de erro | Middleware de erro já serializa `AppError`/Zod/Prisma sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERR-06 | docs/FDD.md | Tratamento de erro | `WEBHOOK_ALREADY_REPLAYED` (409) para replay repetido de um item da DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-01 | docs/FDD.md | Estratégia de resiliência | Timeout HTTP de 10 s (via AbortController) | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Estratégia de resiliência | 5 tentativas de retry | TRANSCRICAO | [09:17] Larissa |
| FDD-RESIL-03 | docs/FDD.md | Estratégia de resiliência | Backoff fixo 1m / 5m / 30m / 2h / 12h | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-04 | docs/FDD.md | Estratégia de resiliência | DLQ dedicada com replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-05 | docs/FDD.md | Estratégia de resiliência | Sem fallback automático de canal (e-mail fora de escopo) | TRANSCRICAO | [09:37] Larissa |
| FDD-RESIL-06 | docs/FDD.md | Estratégia de resiliência | Transação atômica: evento só existe se `changeStatus` commitou | TRANSCRICAO | [09:41] Diego |
| FDD-RESIL-07 | docs/FDD.md | Estratégia de resiliência | Shutdown gracioso espelhando o padrão de `src/server.ts` | CODIGO | src/server.ts |
| FDD-RESIL-08 | docs/FDD.md | Estratégia de resiliência | Salvaguarda: payload acima de 64 KB não é enviado, vai para a DLQ | TRANSCRICAO | [09:24] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados reusando o logger Pino existente | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Redação da secret / `x-signature` nos logs | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por `event_id` / `X-Request-Id` já gerado por request | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Alerta de atraso da fila comparado ao SLA de 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-INT-01 | docs/FDD.md | Integração | Estender `OrderService.changeStatus` com `publishWebhookEvent(tx, ...)` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Erros do módulo estendem a hierarquia de erros existente | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Reuso de `authenticate` e `requireRole('ADMIN')` nas rotas | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Registrar os routers de webhook no roteador da API | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Wiring do módulo no composition root | CODIGO | src/app.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Worker instancia `PrismaClient` via `createPrismaClient` | CODIGO | src/config/database.ts |
| FDD-INT-07 | docs/FDD.md | Integração | `src/worker.ts` espelha a entry-point `src/server.ts` | CODIGO | src/server.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Script `npm run worker` acrescentado | CODIGO | package.json |
| FDD-INT-09 | docs/FDD.md | Integração | Novos modelos + migração aditiva no schema | CODIGO | prisma/schema.prisma |
| FDD-INT-10 | docs/FDD.md | Integração | Novas variáveis no `envSchema` (Zod) | CODIGO | src/config/env.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Acrescentar `secret` / `x-signature` ao `redact` do logger | CODIGO | src/shared/logger/index.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Padrão de rota com `requireRole` como em `user.routes` | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Testes de integração no estilo de `orders.test` (supertest) | CODIGO | tests/orders.test.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Uso das factories de teste existentes | CODIGO | tests/helpers/factories.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Validação via middleware `validate` + schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-16 | docs/FDD.md | Integração | `subscribedStatuses` limitado ao enum `OrderStatus` | CODIGO | src/modules/orders/order.status.ts |
| FDD-CTX-01 | docs/FDD.md | Contexto | Sistema não possui hoje notificação externa, fila ou processamento assíncrono | CODIGO | src/modules/orders/order.service.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Runtime Node ≥ 20 (crypto e fetch nativos, sem pacotes novos) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Deploy: novo processo `worker`, não iniciado pela API | TRANSCRICAO | [09:11] Diego |
| FDD-AC-01 | docs/FDD.md | Critério de aceite | Rollback da transação não deixa evento na outbox | TRANSCRICAO | [09:41] Diego |
| FDD-AC-02 | docs/FDD.md | Critério de aceite | Evento entregue em ≤ ~4 s com o worker ativo e cliente respondendo 200 | TRANSCRICAO | [09:10] Larissa |
| FDD-AC-03 | docs/FDD.md | Critério de aceite | `OPERATOR` recebe 403 no replay; `ADMIN` consegue e fica auditado | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-04 | docs/FDD.md | Critério de aceite | Alterar o pedido depois não muda o payload já gravado (snapshot) | TRANSCRICAO | [09:52] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de aceitação | Cliente cadastra webhook https e recebe a secret na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de aceitação | Revisão de segurança da Sofia concluída antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-01 | docs/PRD.md | Estratégia de teste | Teste de atomicidade (não perda de eventos) | TRANSCRICAO | [09:41] Diego |
| PRD-TEST-02 | docs/PRD.md | Estratégia de teste | Validação de negócio: contrato documentado no portal do desenvolvedor para os clientes | TRANSCRICAO | [09:26] Marcos |
| PRD-TEST-03 | docs/PRD.md | Estratégia de teste | Gate de segurança: revisão de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |

---

## Verificação

**214 linhas** no total.

| Verificação exigida | Alvo | Resultado |
|---|---|---|
| Itens identificáveis dos documentos com linha correspondente | ≥ 80 % | **~95 %** — cobre todos os RF/RNF do PRD, objetivos, exclusões, riscos, personas e dependências; todas as decisões, alternativas e questões em aberto do RFC; a decisão principal e as sub-decisões de cada ADR; e o modelo de dados, fluxos, contratos, matriz de erros, resiliência, observabilidade e pontos de integração do FDD |
| Linhas com Fonte = `TRANSCRICAO` e timestamp `[hh:mm] Nome` válido | ≥ 70 % | **81 %** (175 de 214 linhas); 100 % delas com timestamp na janela `[09:00]`–`[09:53]` e falante entre Marcos, Bruno, Diego, Larissa e Sofia |
| Linhas com Fonte = `CODIGO` e caminho de arquivo real | ≥ 5 | **39 linhas**, referenciando 20 arquivos reais: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`, `src/modules/users/user.routes.ts`, `prisma/schema.prisma`, `package.json`, `tests/orders.test.ts`, `tests/helpers/factories.ts` |

Todos os timestamps citados existem na [TRANSCRICAO.md](../TRANSCRICAO.md) (janela
`[09:00]`–`[09:53]`) e todos os caminhos de código existem no repositório
(`git ls-files`).

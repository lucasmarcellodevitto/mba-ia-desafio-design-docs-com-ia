# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Em revisão |
| **Data** | 2026-09-09 |
| **Autor** | Larissa (Tech Lead) — abre o doc de design da feature ([09:50]) |
| **Product Manager** | Marcos |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. de Segurança) |
| **Fonte de requisitos** | [TRANSCRICAO.md](../TRANSCRICAO.md) (timestamps `[hh:mm]`) e código-fonte da aplicação |
| **Documentos relacionados** | [RFC.md](RFC.md) · [FDD.md](FDD.md) · ADRs em [docs/adrs/](adrs/) |

> Toda informação deste PRD é rastreável à transcrição da reunião técnica ou ao código.
> O "como construir" está no [FDD](FDD.md); a proposta de arquitetura, no [RFC](RFC.md);
> as decisões, nas [ADRs](adrs/).

---

## 1. Resumo e contexto da feature

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram
formalmente para ser notificados quando o status de um pedido deles muda na nossa
plataforma ([09:00] Marcos). Hoje eles descobrem mudanças fazendo *polling* no
`GET /orders` de tempos em tempos, o que torna a integração lenta e cara para eles
([09:00] Marcos).

A feature entrega **webhooks HTTP outbound**: quando o status de um pedido muda, a
plataforma envia uma requisição assinada para uma URL cadastrada pelo cliente, em poucos
segundos. O cliente cadastra e gerencia seus endpoints pela nossa API, escolhe quais
status quer receber, consulta o histórico de entregas e roda com uma *secret* própria
para validar a autenticidade das chamadas.

Arquitetura resumida (detalhe em [RFC.md](RFC.md) / [FDD.md](FDD.md)): padrão **Outbox**
gravado na mesma transação da mudança de status, **worker em processo separado** fazendo
*polling*, **retry com backoff + dead-letter queue**, assinatura **HMAC-SHA256** e entrega
**at-least-once**.

---

## 2. Problema e motivação

- **Integração ineficiente:** os clientes fazem *polling* repetido em `GET /orders` para
  detectar mudanças, o que é "lento e caro" para eles ([09:00] Marcos).
- **Experiência ruim:** a integração "fica pendurada" e exige que o cliente "fique
  atualizando manualmente" ([09:02] Marcos).
- **Risco comercial concreto:** a Atlas sinalizou que, se a entrega não sair até o fim do
  trimestre, pode migrar para um concorrente ([09:00] Marcos).
- **Restrição técnica:** a mudança de status roda numa transação já pesada
  (`OrderService.changeStatus` — [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts),
  linhas 126‑179 — atualiza pedido, histórico e estoque). Uma chamada HTTP síncrona ali
  travaria mudanças de status de outros pedidos com um cliente lento, e a indisponibilidade
  do cliente forçaria *rollback* do negócio ([09:04] Bruno).

---

## 3. Público-alvo e cenários de uso

### Personas

| Persona | Descrição | Origem |
| --- | --- | --- |
| **Cliente B2B integrador** | Sistemas de Atlas Comercial, MaxDistribuição e Nova Cargo que consomem status de pedidos. Cadastram e operam webhooks pela API. | [09:00] Marcos |
| **Usuário que representa o cliente** | Conta de usuário do nosso sistema (JWT) que o cliente usa para chamar a API de configuração de webhooks. | [09:32] Marcos |
| **Operador de plataforma (ADMIN)** | Time interno que reprocessa entregas que caíram na DLQ. | [09:36] Sofia |
| **Product / Dev Rel (Marcos)** | Documenta o contrato de webhook no portal do desenvolvedor. | [09:26], [09:40] Marcos |

### Cenários de uso

1. **Cadastro:** o cliente chama `POST /webhooks` com a URL (`https`) e a lista de status
   que quer ouvir; recebe de volta a *secret* (exibida só nessa resposta) ([09:31] Marcos).
2. **Notificação em tempo quase real:** o pedido do cliente passa a `SHIPPED`; em poucos
   segundos ele recebe um `POST` assinado no endpoint dele ([09:02], [09:10]).
3. **Cliente temporariamente offline:** a entrega falha; a plataforma retenta com backoff
   por até ~15 h; quando o cliente volta, recebe o evento ([09:15]‑[09:17]).
4. **Falha permanente + recuperação:** esgotadas as tentativas, o evento vai para a DLQ;
   um operador ADMIN dispara o *replay* manual ([09:18], [09:35]‑[09:36]).
5. **Rotação de *secret*:** o cliente pede uma nova *secret* pela API; a antiga continua
   válida por 24 h para ele migrar ([09:21] Sofia).
6. **Auditoria:** o cliente consulta `GET /webhooks/:id/deliveries` para ver as últimas
   entregas (sucesso/falha, payload, resposta, tempo) ([09:34] Marcos).
7. **Filtro de eventos:** o cliente edita o webhook para ouvir só `SHIPPED` e `DELIVERED`;
   os demais status deixam de gerar evento para ele ([09:33]‑[09:34]).

---

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| **O1** | Notificar o cliente em tempo quase real | Tempo entre o *commit* da mudança de status e a resposta `2xx` do cliente (p95) | **< 10 s** | [09:02] Marcos ("qualquer coisa abaixo de 10 segundos já é tempo real") |
| **O2** | Eliminar o *polling* dos 3 clientes sobre `GET /orders` | Nº de clientes-alvo com ≥ 1 endpoint de webhook ativo | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o fim de novembro | [09:00] Marcos; prazo [09:45] Marcos |
| **O3** | Não perder eventos de mudança de status | Nº de mudanças de status commitadas sem a linha correspondente na `webhook_outbox` | **0** | [09:40] Bruno; [09:41] Diego |
| **O4** | Tolerar indisponibilidade temporária do cliente | Janela de indisponibilidade do cliente coberta pelo ciclo de *retry* antes de desistir | **≈ 15 h** entre a 1ª falha e a última tentativa | [09:16]‑[09:17] Diego |
| **O5** | Entregar a feature no prazo comercial | Data de disponibilização em produção | **Fim de novembro** (≈ 3 sprints, com revisão de segurança) | [09:45]‑[09:47] |

---

## 5. Escopo

### 5.1 Incluído

- Webhooks **outbound** para o evento `order.status_changed` ([09:02] Marcos; [09:43] Diego).
- CRUD de configuração de webhook por cliente: criar, editar, listar por customer, remover
  ([09:31]‑[09:33]).
- *Secret* gerada pela plataforma e devolvida apenas na criação ([09:31] Marcos).
- Filtro por lista de status que cada webhook deseja receber ([09:33]‑[09:34]).
- Assinatura das entregas e *secret* única por endpoint, com rotação (janela de 24 h)
  ([09:20]‑[09:22] Sofia) — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md).
- TLS obrigatório (URL `https`; `http` recusado) ([09:23] Sofia).
- Entrega assíncrona (sem bloquear o fluxo de pedidos), com latência dentro do SLA de 10 s
  ([09:06]‑[09:10]) — [ADR-001](adrs/ADR-001-outbox-no-mysql.md),
  [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- Retry automático de falhas transitórias e *dead-letter queue* para falhas permanentes
  ([09:15]‑[09:18]) — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).
- *Replay* manual da DLQ pelo time interno, restrito a `ADMIN` e auditado ([09:35]‑[09:36]).
- Histórico de entregas consultável pelo cliente ([09:34] Marcos).
- Entrega at-least-once com deduplicação pelo cliente
  ([09:24]‑[09:26]) — [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).
- Documentação do contrato no portal do desenvolvedor ([09:26], [09:40] Marcos).

### 5.2 Fora de escopo

Itens levantados na reunião e explicitamente **descartados** ou **adiados**:

| Item | Situação | Origem |
| --- | --- | --- |
| **Webhooks *inbound*** (cliente → plataforma) | Descartado — "só saindo da gente pra eles" | [09:02] Marcos |
| **Aviso ao cliente (e-mail) quando o webhook falha repetidamente** | Adiado — "fora de escopo dessa fase. Talvez próxima fase" | [09:37] Larissa |
| **Rate limiting de envio de saída por cliente** | Adiado — "observar e decidir depois" | [09:39] Larissa |
| **Dashboard / painel visual para o cliente** | Descartado desta entrega — "projeto separado do time de frontend" | [09:40] Larissa |
| **Arquivamento das linhas entregues da outbox (retention ~30 dias)** | Fora do escopo desta feature | [09:08] Diego |
| **Múltiplos workers em paralelo / garantia de ordering global** | Adiado — "problema do futuro, não agora" | [09:13] Diego |
| **Endurecer papéis no CRUD de configuração** (hoje qualquer papel autenticado) | Adiado — "mais pra frente a gente pode endurecer" | [09:37] Sofia |
| **Garantia de entrega *exactly-once*** | Descartado — at-least-once + `event_id` resolve, exactly-once é muito mais complexo | [09:25] Diego |

---

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **RF-01** | O cliente cadastra um webhook via `POST`, informando a URL de destino e a lista de status que deseja receber; a plataforma gera a *secret* e a devolve **apenas** na resposta de criação. | [09:31] Marcos |
| **RF-02** | O `customer_id` do webhook é informado no corpo/rota da requisição, **não** derivado do JWT (o JWT atual é do usuário operador). O endpoint é autenticado. | [09:32] Larissa; [09:32] Bruno |
| **RF-03** | O cliente edita um webhook (`PATCH`), remove um webhook (`DELETE`) e lista os webhooks de um customer (`GET`). | [09:33] Bruno |
| **RF-04** | Cada webhook define uma **lista de status** que quer ouvir (ex.: só `SHIPPED` e `DELIVERED`); o filtro é aplicado **no momento da inserção** na outbox — se nenhum webhook do customer quer aquele status, o evento não é gravado. | [09:33] Marcos; [09:34] Bruno |
| **RF-05** | Quando o status de um pedido muda, um evento é registrado **na mesma transação** que atualiza o pedido; não pode haver caso de "status mudou e evento não saiu". | [09:40] Bruno; [09:41] Diego |
| **RF-06** | A plataforma entrega o evento por `POST` HTTP para a URL do webhook, com um payload JSON contendo os dados essenciais do pedido (identificação, transição de status, cliente, valor total) e **sem** a lista de itens — o cliente busca detalhes em `GET /orders/:id` quando precisar. Campos exatos: [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md) / [FDD §6.10](FDD.md). | [09:43] Diego |
| **RF-07** | Cada entrega é assinada e identificada de forma que o cliente possa verificar a autenticidade/integridade da chamada e deduplicar recebimentos repetidos. Headers exatos: [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md) / [FDD §6.9](FDD.md). | [09:44]‑[09:45] Diego/Sofia |
| **RF-08** | Se a entrega falhar (resposta de erro, timeout ou indisponibilidade do cliente), a plataforma retenta automaticamente, com espaçamento crescente, cobrindo uma janela de indisponibilidade de ~15 h antes de desistir. Progressão e nº de tentativas: [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md). | [09:15]‑[09:17] Diego/Larissa |
| **RF-09** | Após esgotar as tentativas de retry, o evento é movido para uma **dead-letter queue** dedicada, com payload, motivo da falha e timestamp. | [09:18] Diego |
| **RF-10** | Um endpoint administrativo permite **reprocessar** manualmente um item da DLQ (recolocando-o como pendente na outbox). Exige papel **`ADMIN`** e registra em log quem executou o *replay*. | [09:35] Diego; [09:36] Sofia/Larissa |
| **RF-11** | O cliente consulta o **histórico de entregas** de um webhook (`GET /webhooks/:id/deliveries`): sucesso/falha, payload, resposta e tempo de resposta das últimas entregas (~100). | [09:34] Marcos |
| **RF-12** | O cliente **rotaciona a *secret*** de um webhook via API; ao rotacionar, a *secret* anterior permanece válida por **24 h** em paralelo, e depois é invalidada. | [09:21] Sofia |
| **RF-13** | A URL do webhook precisa ser **`https`**; uma URL `http` é recusada com erro de validação. | [09:23] Sofia |
| **RF-14** | O conteúdo entregue reflete o estado do pedido **no momento da transição de status** — não o estado atual, caso o pedido mude depois. | [09:52] Larissa/Diego/Bruno |

Detalhamento de contratos, payloads de exemplo, status codes e matriz de erros: [FDD.md §6 e §7](FDD.md).

---

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **RNF-01** | **Latência de entrega:** p95 do tempo entre a mudança de status e a entrega ao cliente **< 10 s** no caminho feliz (worst case de ~2 s por conta do *polling*). | [09:02] Marcos; [09:10] Larissa |
| **RNF-02** | **Não bloqueio do fluxo de pedidos:** a entrega do webhook não pode travar nem atrasar a mudança de status de outros pedidos (entrega assíncrona, fora da transação de negócio). | [09:04] Bruno — [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **RNF-03** | **Isolamento de processo:** o worker roda como processo separado da API; reinício da API não derruba o worker. | [09:11] Diego — [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| **RNF-04** | **Autenticidade e integridade:** toda entrega assinada com HMAC-SHA256; *secret* única por endpoint (vazar uma não compromete as demais). | [09:20]‑[09:21] Sofia — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| **RNF-05** | **Transporte seguro:** TLS obrigatório (`https`). | [09:23] Sofia |
| **RNF-06** | **Timeout de entrega:** cliente que não responde em **10 s** é tratado como falha. | [09:42] Diego |
| **RNF-07** | **Tamanho de payload:** limite de **64 KB**; acima disso, erro (não trunca) e o evento não é enviado. | [09:24] Diego/Sofia/Larissa |
| **RNF-08** | **Semântica de entrega:** at-least-once; o cliente pode receber o mesmo evento mais de uma vez e é responsável por deduplicar (cada evento tem um identificador único, estável entre tentativas). | [09:24]‑[09:26] Diego — [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| **RNF-09** | **Ordenação:** por `order_id` enquanto houver um único worker; **sem** garantia de ordenação global (limitação conhecida e documentada). | [09:12]‑[09:13] Diego |
| **RNF-10** | **Reuso e simplicidade:** a feature segue os padrões de engenharia já estabelecidos no projeto e **não adiciona dependências novas**. | [09:27]‑[09:30] — [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| **RNF-11** | **Auditoria:** o *replay* da DLQ registra qual usuário o executou. | [09:36] Sofia |
| **RNF-12** | **Revisão de segurança obrigatória** antes do deploy (≥ 2 dias úteis reservados para revisar HMAC e geração de *secret*). | [09:46] Sofia |

---

## 8. Decisões e trade-offs principais

O registro de cada decisão arquitetural está nas [ADRs](adrs/); a proposta consolidada e
as alternativas descartadas estão no [RFC](RFC.md). Do ponto de vista **de produto**, os
trade-offs que afetam a experiência do cliente, o escopo ou a operação são:

- **Entrega assíncrona com ~2 s de latência de base** (para não sobrecarregar o fluxo de
  pedidos nem exigir infraestrutura nova) — dentro do SLA de 10 s —
  [ADR-001](adrs/ADR-001-outbox-no-mysql.md),
  [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- **At-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa
  deduplicar; não oferecemos *exactly-once* —
  [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).
- **Recuperação de falhas permanentes é manual** (reprocessamento pelo time interno), sem
  aviso automático ao cliente nesta fase —
  [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).
- **Ordenação garantida apenas por pedido** (não global) e enquanto houver um único
  worker — [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- **O contrato de payload/headers é uma interface pública versionável**, documentada no
  portal do desenvolvedor —
  [ADR-007](adrs/ADR-007-formato-de-payload-headers-e-limites.md).

---

## 9. Dependências

| Dependência | Natureza | Origem |
| --- | --- | --- |
| **Fluxo de mudança de status de pedido** | Passa a registrar o evento na mesma transação de negócio — ponto de integração mais sensível ([FDD §12.1](FDD.md)) | [09:40]‑[09:41] |
| **Banco de dados e ORM existentes** (MySQL/Prisma) | Novas tabelas via migração aditiva; o worker usa conexão própria | [09:30] Bruno |
| **Autenticação e autorização existentes** (JWT, papéis) | Reuso; o *replay* da DLQ exige papel ADMIN | [09:36] Larissa |
| **Runtime da aplicação** (Node ≥ 20) | Assinatura e chamadas HTTP com timeout usando recursos nativos — sem dependências novas ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)) | [09:30] |
| **Pipeline de deploy** | Precisa subir e monitorar um novo processo (worker) separado da API | [09:11] Diego |
| **Portal do desenvolvedor** | Marcos documenta o contrato de webhook para os clientes | [09:26], [09:40] Marcos |
| **Revisão de segurança (Sofia)** | *Gate* obrigatório antes do deploy: HMAC e geração de *secret* | [09:46]‑[09:47] |

---

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação | Origem |
| --- | --- | --- | --- | --- |
| **Worker parado silenciosamente** (crash/deadlock) — eventos deixam de sair e o cliente volta a ficar "pendurado" | Média | Alto | Monitoramento do atraso da fila com alerta quando ultrapassa o SLA; mecanismos de detecção e recuperação em [FDD §8‑9](FDD.md) | [09:11] Diego |
| **Vazamento de *secret*** (log da nossa aplicação ou do cliente) — permite forjar chamadas | Baixa | Alto | *Secret* única por endpoint (raio de impacto contido); *redact* no logger; *secret* só exposta na criação e na rotação; revisão de segurança da Sofia antes do deploy | [09:21]‑[09:22] Sofia/Diego; [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| **Processamento por um único worker vira gargalo sob pico** — latência acima de 10 s | Média | Médio | A cadência de verificação já opera dentro do SLA com folga; caminho preparado para escalar (particionar por pedido) quando necessário — adiado ([09:13]) | [09:12]‑[09:13] Diego |
| **Alteração no caminho crítico da mudança de status** degrada a escrita de pedidos | Baixa | Alto | A gravação do evento é uma operação leve, sem chamada externa; medir a latência da mudança de status antes/depois — [FDD §13](FDD.md) | [09:04] Bruno |
| **Cliente não implementa a deduplicação** — processa o mesmo evento duas vezes | Média | Médio (no lado do cliente) | Documentação destacada no portal do desenvolvedor; identificador único de evento, estável entre tentativas | [09:25]‑[09:26] Sofia/Marcos |
| **Prazo apertado** (fim de novembro / ~3 sprints) com a revisão de segurança concentrada no fim | Média | Alto | Entregar HMAC e geração de *secret* já na 1ª sprint para revisão antecipada; reservar 2 dias úteis para a Sofia | [09:45]‑[09:47] |

---

## 11. Critérios de aceitação

Nível de produto (critérios técnicos detalhados e testáveis em [FDD §11](FDD.md)):

1. Um cliente consegue cadastrar um webhook `https` informando os status desejados e
   recebe a *secret* na resposta de criação (RF-01, RF-13).
2. Ao mudar o status de um pedido de um customer com webhook ativo inscrito naquele
   status, o endpoint do cliente recebe um `POST` assinado com o payload e os headers
   especificados, em menos de 10 s no caminho feliz (RF-05, RF-06, RF-07, RNF-01).
3. Se nenhum webhook do customer quer aquele status, nenhum evento é gerado (RF-04).
4. Se a mudança de status sofre *rollback*, nenhum evento é entregue (RF-05, O3).
5. Um cliente que responde com erro/timeout recebe novas tentativas espaçadas; após
   esgotá-las, o evento aparece na DLQ (RF-08, RF-09).
6. Um usuário `ADMIN` consegue reprocessar um item da DLQ e o *replay* fica registrado
   com o autor; um usuário não-`ADMIN` não consegue (RF-10, RNF-11).
7. O cliente consegue listar as últimas entregas de um webhook com status, payload,
   resposta e tempo de resposta (RF-11).
8. O cliente consegue rotacionar a *secret*; entregas continuam validáveis com a *secret*
   anterior durante 24 h (RF-12).
9. O cliente consegue identificar e ignorar um evento recebido em duplicidade (RNF-08).
10. A revisão de segurança da Sofia foi concluída antes do deploy (RNF-12).

---

## 12. Estratégia de testes e validação

O plano de testes detalhado (casos, ferramentas, critérios técnicos) está no
[FDD §11 e §12](FDD.md). Em nível de produto, a feature só é considerada validada com:

- **Cobertura funcional automatizada** de cada requisito do §6 (CRUD, filtro de status,
  autorização do *replay*, histórico de entregas, rotação de *secret*).
- **Garantia de não perda de eventos:** teste de atomicidade — falha ao registrar o
  evento reverte a mudança de status junto (O3).
- **Comportamento de entrega ponta a ponta:** latência dentro do SLA de 10 s, ciclo
  completo de retry até a DLQ e *replay* funcionando.
- **Segurança:** verificação da assinatura e da rotação de *secret*; **gate obrigatório**
  de revisão do código de HMAC e geração de *secret* pela Sofia antes do deploy ([09:46]).
- **Validação de negócio:** Marcos valida o contrato publicado no portal do desenvolvedor
  com os três clientes ([09:26], [09:40], [09:47]).

---

## 13. Prazo

Estimativa da Tech Lead: **três sprints**, incluindo a revisão de segurança da Sofia ao
final; alvo comercial da Atlas é o **fim de novembro** ([09:45]‑[09:47]). Marcos confirma
o prazo com os clientes ([09:47]).

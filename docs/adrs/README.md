# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada aqui em arquivos individuais,
nomeados no formato `ADR-NNN-titulo-em-kebab-case.md`.

Formato: variante MADR com as seções Título, Status, Contexto, Decisão,
Alternativas Consideradas e Consequências. Toda informação registrada é
rastreável à [transcrição da reunião](../../TRANSCRICAO.md) (timestamps `[hh:mm]`)
ou ao código-fonte da aplicação.

## Índice

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox sobre o MySQL existente, na mesma transação de `changeStatus` | `[Revisão]` |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado (`src/worker.ts`) com polling de 2s | `[Revisão]` |
| [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada | `[Revisão]` |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | `[Revisão]` |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` para deduplicação no cliente | `[Revisão]` |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto (módulos, `AppError`, Pino, Zod, `requireRole`) | `[Revisão]` |
| [ADR-007](ADR-007-formato-de-payload-headers-e-limites.md) | Formato de payload (snapshot), headers `X-*`, timeout de 10s e limite de 64 KB | `[Revisão]` |

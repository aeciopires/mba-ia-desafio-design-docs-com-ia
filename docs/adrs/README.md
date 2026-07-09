<!-- TOC -->

- [Architectural Decision Records](#architectural-decision-records)
  - [ADRs da feature de Webhooks de Notificação de Pedidos](#adrs-da-feature-de-webhooks-de-notificação-de-pedidos)

<!-- TOC -->

# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante deve ser registrada aqui em arquivos individuais,
nomeados sequencialmente no formato `ADR-NNN-titulo-em-kebab-case.md` (por exemplo
`ADR-001-titulo-da-decisao.md`), seguindo o formato MADR (Status, Contexto, Decisão,
Alternativas Consideradas, Consequências).

## ADRs da feature de Webhooks de Notificação de Pedidos

- [ADR-001 — Outbox Pattern no MySQL](./ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002 — Worker em Processo Separado com Polling](./ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003 — Retry com Backoff Exponencial e Dead Letter Queue](./ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)
- [ADR-004 — Autenticação HMAC-SHA256 com Secret por Endpoint](./ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Entrega At-Least-Once com Deduplicação via X-Event-Id](./ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md)
- [ADR-006 — Reuso dos Padrões Existentes do Projeto](./ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Outbox com Chave Primária UUID e Payload Snapshot na Inserção](./ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md)
- [ADR-008 — Endpoint Administrativo de Replay da DLQ com Role ADMIN e Auditoria](./ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md)

# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/).

## 1.0.0 — 2026-07-10

### Added

- `docs/adrs/` — 8 Architecture Decision Records descrevendo as decisões técnicas do sistema de webhooks outbound de notificação de pedidos:
  - `ADR-001-outbox-pattern-no-mysql.md`
  - `ADR-002-worker-em-processo-separado-com-polling.md`
  - `ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md`
  - `ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md`
  - `ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md`
  - `ADR-006-reuso-dos-padroes-existentes-do-projeto.md`
  - `ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md`
  - `ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md`
- `docs/adrs/README.md` — índice dos ADRs com links para os 8 documentos.
- `docs/RFC.md` — proposta técnica consolidada, alternativas descartadas e questões em aberto.
- `docs/FDD.md` — especificação de implementação: fluxos, contratos HTTP, matriz de erros e integração com o código existente.
- `docs/PRD.md` — visão de produto: problema, escopo, requisitos, riscos e critérios de aceitação.
- `docs/TRACKER.md` — matriz de rastreabilidade de cada item registrado nos documentos à sua origem na transcrição (`TRANSCRICAO.md`) ou no código-base.
- `PLAN.md` — plano de produção documento por documento usado para orientar a criação do pacote.
- `README.md` — documentação do processo de produção (ferramentas de IA, workflow, prompts customizados, iterações e ajustes, guia de navegação da entrega).
- `LICENSE` — MIT License.

### Fixed

- `docs/adrs/README.md` — corrigida convenção de nomenclatura de ADRs, que orientava incorretamente o formato `0001-titulo.md` em vez do formato `ADR-NNN-titulo-em-kebab-case.md` efetivamente exigido e usado nos 8 ADRs.

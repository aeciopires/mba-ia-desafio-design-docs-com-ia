<!-- TOC -->

- [Tracker de Rastreabilidade](#tracker-de-rastreabilidade)
  - [ADRs](#adrs)
  - [RFC](#rfc)
  - [FDD](#fdd)
  - [PRD](#prd)

<!-- TOC -->

# Tracker de Rastreabilidade

Este documento mapeia cada item registrado nos documentos do pacote (PRD, RFC, FDD, ADRs) à sua origem — na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte existente. Nenhuma linha abaixo foi inventada: toda informação com Fonte `TRANSCRICAO` tem um timestamp `[hh:mm] Nome` verificável no arquivo `TRANSCRICAO.md`; toda informação com Fonte `CODIGO` referencia um caminho de arquivo real do repositório.

Estatística de cobertura desta versão: 117 linhas rastreadas, sendo 94 com Fonte `TRANSCRICAO` (~80%) e 23 com Fonte `CODIGO` (~20%), cobrindo os itens identificáveis dos 4 documentos + 8 ADRs do pacote.

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Outbox transacional no MySQL para publicar eventos de webhook | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Alternativa Descartada | Despacho síncrono dentro da transação de changeStatus descartado | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Alternativa Descartada | Redis Streams / fila externa descartada por overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-002-DEC | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado, polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa Descartada | Trigger de banco + notificação de processo externo descartado (MySQL sem LISTEN/NOTIFY) | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição | Worker não pode rodar no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa Descartada | Múltiplos workers em paralelo desde o início descartado, adiado como problema futuro | TRANSCRICAO | [09:13] Diego |
| ADR-002-LIM-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição | Ordenação de entrega garantida apenas por order_id e apenas em single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-003-DEC | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | Retry de 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela própria | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Alternativa Descartada | 3 tentativas descartado por ser insuficiente frente a outages reais | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Alternativa Descartada | Retry indefinido com backoff descartado (evento pendurado pra sempre) | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Alternativa Descartada | DLQ como flag "failed" na própria outbox descartada, tabela separada escolhida | TRANSCRICAO | [09:18] Diego |
| ADR-004-DEC | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global única para toda a plataforma descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CTX-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Restrição | Precedente real de vazamento de secret em log de cliente motivou a decisão | TRANSCRICAO | [09:22] Diego |
| ADR-005-DEC | docs/adrs/ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md | Decisão | Entrega at-least-once com X-Event-Id para dedup do lado do cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md | Alternativa Descartada | Garantia exactly-once descartada por complexidade de coordenação | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes do projeto (módulo, erros, logger, auth) | TRANSCRICAO | [09:30] Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Estrutura de módulo controller/service/repository/routes/schemas replicada | CODIGO | src/modules/orders/order.service.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Hierarquia AppError/ConflictError/UnprocessableEntityError reaproveitada | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Middleware de erro centralizado não precisa de alteração | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | requireRole reaproveitado para controle de acesso | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Logger Pino singleton reaproveitado | CODIGO | src/shared/logger/index.ts |
| ADR-006-COD-06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | PrismaClient separado por processo no worker, mesmo padrão de construção | CODIGO | src/config/database.ts |
| ADR-007-DEC | docs/adrs/ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md | Decisão | Outbox com PK UUID e payload congelado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md | Alternativa Descartada | ID auto-incremental descartado, UUID escolhido por consistência de projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007-ALT-02 | docs/adrs/ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md | Alternativa Descartada | Re-renderizar payload no envio descartado, snapshot na inserção escolhido | TRANSCRICAO | [09:52] Larissa |
| ADR-008-DEC | docs/adrs/ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md | Decisão | Replay de DLQ restrito à role ADMIN, com auditoria de quem executou | TRANSCRICAO | [09:36] Larissa |
| ADR-008-ALT-01 | docs/adrs/ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md | Alternativa Descartada | Replay aberto a qualquer role autenticada descartado para esta ação específica | TRANSCRICAO | [09:36] Sofia |
| ADR-008-ALT-02 | docs/adrs/ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md | Restrição | Log de auditoria obrigatório para a ação de replay | TRANSCRICAO | [09:36] Sofia |

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Requisito Funcional | Pedido formal de 3 clientes B2B por notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Requisito Não Funcional | "Tempo real" definido pelos clientes como <10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Despacho síncrono descartado por travar transação em cliente lento/indisponível | TRANSCRICAO | [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado por overengineering frente ao MySQL existente | TRANSCRICAO | [09:07] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída para clientes sem dono nem prazo definidos | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers em paralelo não desenhada | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Hardening de autorização do CRUD sem gatilho de quando ocorrer | TRANSCRICAO | [09:37] Sofia |
| RFC-RISK-01 | docs/RFC.md | Decisão | Estimativa de 3 sprints, incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-RISK-02 | docs/RFC.md | Restrição | Revisão de segurança de pelo menos 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |

## FDD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBJ-01 | docs/FDD.md | Requisito Não Funcional | Notificação <10s (P95) como objetivo técnico | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-02 | docs/FDD.md | Restrição | Inserção da outbox deve ser atômica com a transação de changeStatus | TRANSCRICAO | [09:41] Diego |
| FDD-ESCOPO-01 | docs/FDD.md | Restrição | Alertas automáticos por falha repetida fora de escopo desta fase | TRANSCRICAO | [09:38] Larissa |
| FDD-ESCOPO-02 | docs/FDD.md | Restrição | Rate limiting de saída fora de escopo desta fase | TRANSCRICAO | [09:39] Larissa |
| FDD-ESCOPO-03 | docs/FDD.md | Restrição | Dashboard visual fora de escopo desta fase | TRANSCRICAO | [09:40] Larissa |
| FDD-ESCOPO-04 | docs/FDD.md | Restrição | Arquivamento de eventos entregues após 30 dias fora de escopo | TRANSCRICAO | [09:08] Diego |
| FDD-ESCOPO-05 | docs/FDD.md | Restrição | Múltiplos workers em paralelo fora de escopo desta fase | TRANSCRICAO | [09:13] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Contrato | Função publishWebhookEvent(tx, order, fromStatus, toStatus) chamada dentro da transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Filtro de status aplicado na inserção da outbox, não no momento do envio | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | Progressão de backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Decisão | Evento move para webhook_dead_letter após 5 tentativas falhas | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks registra webhook (url, secret, statuses, customer_id) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id edita configuração | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id remove configuração | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | GET /api/v1/webhooks lista webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay replay administrativo | TRANSCRICAO | [09:18] Diego |
| FDD-HEADERS-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload JSON com event_id/event_type/timestamp/order_id/order_number/from_status/to_status/customer_id/total_cents, sem items | TRANSCRICAO | [09:43] Diego |
| FDD-ERRO-01 | docs/FDD.md | Restrição | WEBHOOK_NOT_FOUND segue padrão de subclasse de AppError/NotFoundError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-02 | docs/FDD.md | Requisito Funcional | WEBHOOK_INVALID_URL para URL cadastrada que não seja https | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Requisito Funcional | Códigos WEBHOOK_NOT_FOUND/WEBHOOK_INVALID_URL/WEBHOOK_SECRET_REQUIRED citados nominalmente com prefixo WEBHOOK_ | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_PAYLOAD_TOO_LARGE para payload acima de 64KB, sem truncar | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-05 | docs/FDD.md | Restrição | WEBHOOK_DUPLICATE_CONFIG segue padrão de subclasse de ConflictError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-06 | docs/FDD.md | Restrição | WEBHOOK_DEAD_LETTER_NOT_FOUND segue padrão de subclasse de NotFoundError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-07 | docs/FDD.md | Requisito Funcional | WEBHOOK_FORBIDDEN_REPLAY para usuário sem role ADMIN no replay | TRANSCRICAO | [09:36] Sofia |
| FDD-ERRO-08 | docs/FDD.md | Restrição | WEBHOOK_VALIDATION_ERROR segue padrão de subclasse de ValidationError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s como estratégia de resiliência | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Retry com backoff exponencial como estratégia de resiliência | TRANSCRICAO | [09:15] Diego |
| FDD-RESIL-03 | docs/FDD.md | Decisão | DLQ como fallback terminal para falhas persistentes | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-04 | docs/FDD.md | Requisito Não Funcional | Isolamento de processo entre worker e API como controle de blast radius | TRANSCRICAO | [09:11] Diego |
| FDD-RESIL-05 | docs/FDD.md | Trade-off | At-least-once em vez de exactly-once como trade-off deliberado de resiliência | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-01 | docs/FDD.md | Integração | Logs do módulo e do worker reaproveitam logger Pino singleton | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Restrição | Replay de DLQ deve ser logado para fins de auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-DEP-01 | docs/FDD.md | Integração | Dependência uuid já existente reaproveitada para event_id e chaves primárias | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Restrição | Node >=20, TypeScript 5.6 e ESM já usados no projeto, sem mudança de compatibilidade | CODIGO | package.json |
| FDD-INTEGRA-01 | docs/FDD.md | Integração | changeStatus estendido com publishWebhookEvent dentro da mesma transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRA-02 | docs/FDD.md | Integração | Novas classes de erro estendem AppError/ConflictError/NotFoundError/UnprocessableEntityError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEGRA-03 | docs/FDD.md | Integração | Middleware de erro centralizado não recebe nenhuma alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGRA-04 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') reaproveitados no endpoint de replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGRA-05 | docs/FDD.md | Integração | Logger Pino singleton reaproveitado no módulo de webhooks e no worker | CODIGO | src/shared/logger/index.ts |
| FDD-INTEGRA-06 | docs/FDD.md | Integração | Novo módulo instanciado via padrão de DI manual e montado em /api/v1/webhooks | CODIGO | src/app.ts |
| FDD-INTEGRA-07 | docs/FDD.md | Integração | src/worker.ts espelha padrão de bootstrap de src/server.ts, como processo independente | CODIGO | src/server.ts |
| FDD-INTEGRA-08 | docs/FDD.md | Integração | OrderStatusHistory como precedente estilístico para webhook_outbox/webhook_dead_letter | CODIGO | prisma/schema.prisma |

## PRD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Meta quantitativa: notificação entregue em <10s (P95) | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (url, secret gerada, statuses, customer_id) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro via JWT de operador/sistema, customer_id explícito no corpo | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição de configuração de webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de configuração de webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem de webhooks de um customer via GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status por endpoint de webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:35] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a ADMIN, com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | CRUD de configuração aberto a qualquer role autenticada nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência <10s como definição de "tempo real" para o negócio | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre inserção do evento e mudança de status | TRANSCRICAO | [09:41] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once, não exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | TLS/HTTPS obrigatório para URLs de webhook cadastradas | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 obrigatório para integridade/autenticidade | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por endpoint, com suporte a rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, rejeição sem truncamento | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Timeout de entrega de 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Isolamento operacional entre worker e API | TRANSCRICAO | [09:11] Diego |
| PRD-ESCOPO-01 | docs/PRD.md | Restrição | Fora de escopo: alertas automáticos por falha repetida de entrega | TRANSCRICAO | [09:38] Marcos |
| PRD-ESCOPO-02 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de saída por cliente | TRANSCRICAO | [09:39] Diego |
| PRD-ESCOPO-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-ESCOPO-04 | docs/PRD.md | Restrição | Fora de escopo: arquivamento de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição | Fora de escopo: múltiplos workers em paralelo / escala horizontal | TRANSCRICAO | [09:13] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente externo não deduplica corretamente eventos at-least-once | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-02 | docs/PRD.md | Risco | Outage prolongado de cliente esgota retries sem alerta automático | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret de cliente (precedente real já ocorrido) | TRANSCRICAO | [09:22] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Clientes externos precisam implementar verificação HMAC e dedup do lado deles | TRANSCRICAO | [09:25] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança dedicada da Sofia antes do deploy em produção | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTES-01 | docs/PRD.md | Restrição | Estratégia de testes segue padrão de integração via supertest já existente | CODIGO | tests/orders.test.ts |
| PRD-TESTES-02 | docs/PRD.md | Restrição | Factories de teste reaproveitadas (bootstrapAuthenticatedUser, createTestCustomer, createTestProduct) | CODIGO | tests/helpers/factories.ts |

# PLAN — Pacote de Design Docs: Sistema de Webhooks de Notificação de Pedidos

## Contexto

O `TASK.md` pede um pacote de documentação técnica (PRD, RFC, FDD, 5–8 ADRs, Tracker de rastreabilidade e README do processo) para uma feature ainda não implementada — um Sistema de Webhooks de Notificação de Pedidos — a partir de duas fontes apenas: a transcrição de uma reunião técnica (`TRANSCRICAO.md`) e o código existente do OMS. Nada pode ser inventado: toda informação registrada precisa ser rastreável a uma dessas duas fontes, e o tracker existe justamente para provar isso. O código (`src/`, `prisma/`, `tests/`) não pode ser alterado — a entrega é 100% documental, incluindo a substituição do `README.md` da raiz pela documentação do processo de produção.

Já explorei a fundo as duas fontes (agentes de pesquisa em paralelo) e validei manualmente os pontos mais sensíveis do código (`order.service.ts`, `http-errors.ts`, `auth.middleware.ts`, `prisma/schema.prisma`). Este plano define exatamente o que vai em cada arquivo, com que conteúdo, em que ordem, e como garantir os critérios de aceite mais arriscados (cobertura do tracker, formato de timestamp, contagem de ADRs, seção obrigatória de integração no FDD).

## Ordem de produção

```
1. ADRs (8 arquivos)   → esqueleto de decisões, citados por todos os outros docs
2. RFC                 → consolida arquitetura em cima dos ADRs, linka-os
3. FDD                 → detalhamento profundo, usa numeração de ADR já congelada
4. PRD                 → visão de produto, consolidação por último entre os 3 grandes
5. TRACKER             → log corrido mantido durante 1–4, varredura final de conferência
6. README.md           → escrito por último, já com o histórico real de iterações
7. Checklist final      → confere Critérios de Aceite item a item
```

Os ADRs são numerados e com título congelado **antes** de escrever o RFC, para que os links `docs/adrs/ADR-NNN-....md` já saiam corretos.

## 1. ADRs — `docs/adrs/` (8 arquivos, formato MADR: Status/Contexto/Decisão/Alternativas Consideradas/Consequências)

Cobrem as 6 decisões obrigatórias (1 ADR por decisão) + 2 ADRs secundários de apoio:

| # | Arquivo | Decisão principal | Fonte na transcrição |
|---|---|---|---|
| 001 | `ADR-001-outbox-pattern-no-mysql.md` | Outbox no MySQL (não síncrono, não Redis Streams) | `[09:03]-[09:08]` |
| 002 | `ADR-002-worker-em-processo-separado-com-polling.md` | Worker separado, polling 2s (não trigger/LISTEN-NOTIFY) | `[09:09]-[09:14]` |
| 003 | `ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md` | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela própria | `[09:15]-[09:18]` |
| 004 | `ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | HMAC-SHA256, secret por endpoint, rotação c/ grace 24h | `[09:19]-[09:22]` |
| 005 | `ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md` | At-least-once + `X-Event-Id` (não exactly-once) | `[09:24]-[09:26]` |
| 006 | `ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Reuso de módulo/erros/logger/auth — **referencia código real** | `[09:27]-[09:30]` |
| 007 | `ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md` | PK UUID + payload congelado na inserção (não re-renderizado) | `[09:51]-[09:52]` |
| 008 | `ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md` | Replay de DLQ só ADMIN + auditoria de quem executou | `[09:18]-[09:19]`, `[09:35]-[09:36]` |

Decisões secundárias que ficam só no FDD (não viram ADR, por indicação explícita do TASK.md): timeout HTTP 10s, formato do payload, headers, limite de 64KB.

**ADR-006** é o ADR-âncora do critério "≥1 ADR referencia código real": cita `src/modules/orders/` (padrão controller/service/repository/routes/schemas a replicar em `src/modules/webhooks/`), `src/shared/errors/app-error.ts` + `http-errors.ts` (hierarquia `AppError`/`ConflictError`/`NotFoundError`/`UnprocessableEntityError`, com o precedente exato `InvalidStatusTransitionError extends ConflictError` e `InsufficientStockError extends UnprocessableEntityError`, ambos confirmados em `http-errors.ts:45` e `:55`), `src/middlewares/error.middleware.ts` (não muda), `src/middlewares/auth.middleware.ts` (`requireRole`, confirmado em `:49`), `src/shared/logger/index.ts` (Pino singleton), `src/config/database.ts` (padrão de PrismaClient). Importante: registrar explicitamente que **não existe hoje** convenção de prefixo por módulo nos códigos de erro (são strings soltas) — `WEBHOOK_*` é uma convenção nova, não a continuação de uma existente.

## 2. RFC — `docs/RFC.md` (2–4 páginas, nível de arquitetura, sem detalhe de implementação)

- **Metadados**: Autor, Status "Em revisão", Data 2026-07-09, Revisores = Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Segurança).
- **TL;DR**, **Contexto e Problema** (clientes Atlas Comercial, MaxDistribuição, Nova Cargo pedindo saída do polling, `[09:00]-[09:02]`).
- **Proposta técnica**: visão geral das 6 decisões (outbox, worker+polling, retry+DLQ, HMAC, at-least-once, reuso), sem payload/schema/tabela de erros — isso é FDD.
- **Alternativas consideradas** (≥2): despacho síncrono descartado `[09:04]`; Redis Streams descartado `[09:07]`.
- **Questões em aberto** (≥2): rate limiting outbound sem dono/prazo `[09:38]-[09:39]`; escala para múltiplos workers "problema do futuro" `[09:13]`.
- **Impacto e riscos**: breve (detalhe fica no PRD/FDD).
- **Decisões relacionadas**: linka os 8 ADRs.
- Disciplina de corte: qualquer frase que desça a nível de campo de JSON, header específico ou código de erro é removida e movida para o FDD.

## 3. FDD — `docs/FDD.md` (documento mais técnico, acionável)

Seções: Contexto e motivação técnica; Objetivos técnicos (mensuráveis); Escopo e exclusões; **Fluxos detalhados** (criação do evento na outbox via nova função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação existente de `OrderService.changeStatus` — método confirmado em `order.service.ts:126`, com `orderStatusHistory.create` em `:159` — processamento pelo worker, retry, DLQ); **Contratos públicos** (7 endpoints reais da transcrição, todos com exemplo de request/response, headers e status code — cobre com folga o mínimo de 4):
- `POST /api/v1/webhooks` (registro, `[09:31]-[09:32]`)
- `PATCH /api/v1/webhooks/:id` (`[09:33]`)
- `DELETE /api/v1/webhooks/:id` (`[09:33]`)
- `GET /api/v1/webhooks?customerId=` (`[09:33]`)
- `GET /api/v1/webhooks/:id/deliveries` (`[09:34]-[09:35]`)
- `POST /api/v1/webhooks/:id/rotate-secret` (`[09:21]-[09:22]`)
- `POST /admin/webhooks/dead-letter/:id/replay` (`[09:18]-[09:19]`, `[09:35]-[09:36]`)

**Matriz de erros `WEBHOOK_*`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DUPLICATE_CONFIG`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_FORBIDDEN_REPLAY`), todos seguindo o padrão de subclasse de `AppError` já usado em `http-errors.ts`. **Estratégias de resiliência** (timeout 10s, backoff, DLQ). **Observabilidade** (métricas, logs via Pino reaproveitado de `src/shared/logger/index.ts`, e tracing — escopado com honestidade como correlação por `X-Event-Id`/request-id, já que não há APM/OpenTelemetry no projeto hoje). Dependências e compatibilidade (Node ≥20, Prisma/MySQL, `uuid` já é dependência existente, HMAC via `crypto` nativo — sem nova dependência). Critérios de aceite técnicos. Riscos e mitigação.

**Seção obrigatória "Integração com o sistema existente"** — mínimo 4, uso 8 caminhos reais e já confirmados:
1. `src/modules/orders/order.service.ts` — ponto de extensão exato da transação de `changeStatus`.
2. `src/shared/errors/app-error.ts` + `src/shared/errors/http-errors.ts` — hierarquia reaproveitada para os novos `Webhook*Error`.
3. `src/middlewares/error.middleware.ts` — não precisa mudar.
4. `src/middlewares/auth.middleware.ts` — `requireRole('ADMIN')` reaproveitado no endpoint de replay.
5. `src/shared/logger/index.ts` — logger Pino singleton reaproveitado no worker.
6. `src/app.ts` / `src/routes/index.ts` — padrão de DI manual e montagem de rotas em `/api/v1/webhooks`.
7. `src/server.ts` — padrão espelhado pelo novo `src/worker.ts`.
8. `prisma/schema.prisma` — `OrderStatusHistory` como precedente estilístico para `webhook_outbox`/`webhook_dead_letter`.

## 4. PRD — `docs/PRD.md`

Todas as seções exigidas. ≥10 requisitos funcionais citados (registro, edição, remoção, listagem, filtragem por status, histórico de entregas, replay DLQ, rotação de secret, JWT de operador, notificação <10s). Único objetivo quantitativo real da reunião: notificação entregue em <10s (P95) `[09:02] Marcos` — sem inventar métricas adicionais (ex.: taxa de sucesso %) que não foram discutidas. "Fora de escopo" com 4+ itens (alertas por e-mail `[09:37]-[09:38]`, rate limiting `[09:38]-[09:39]`, dashboard visual `[09:39]-[09:40]`, arquivamento após 30 dias `[09:08]`, múltiplos workers `[09:13]`). "Riscos" com 3 riscos, cada um com probabilidade+impacto+mitigação (cliente não deduplica; outage >15h sem alerta; vazamento de secret — este último com precedente real citado em `[09:21]-[09:22]`). Estratégia de testes referenciando o padrão real já usado em `tests/orders.test.ts` e `tests/helpers/factories.ts` (Vitest + Supertest, integração contra `buildApp`).

## 5. Tracker — `docs/TRACKER.md`

Construído como log corrido durante os passos 1–4 (uma linha por afirmação concreta, no momento em que ela é escrita em qualquer doc), não reconstruído de memória no final — esse é o maior risco de não bater os 80%. Esquema de ID por seção (`ADR-NNN`, `RFC-ALT-NN`, `RFC-OPEN-NN`, `FDD-CONTRATO-NN`, `FDD-ERRO-NN`, `FDD-INTEGRA-NN`, `PRD-FR-NN`, `PRD-RISK-NN`, etc.). Regra rígida para a coluna Localização: exatamente um `[hh:mm] Nome` por linha de fonte TRANSCRICAO (nunca intervalo, nunca dois nomes) — evita quebrar o validador de formato. ≥5 linhas CODIGO garantidas estruturalmente só pelo ADR-006 e pela seção de integração do FDD (que sozinhos já somam 14 caminhos reais). Varredura final conferindo numericamente: cobertura ≥80%, TRANSCRICAO com timestamp válido ≥70%, CODIGO ≥5.

## 6. README.md (substitui o conteúdo atual)

Seções obrigatórias: Sobre o desafio (paráfrase própria do cenário); Ferramentas de IA utilizadas (refletir com honestidade as ferramentas realmente usadas nesta sessão, não uma lista genérica); Workflow adotado (ordem real: exploração paralela de transcrição+código → ADRs → RFC → FDD → PRD → Tracker em paralelo → README); Prompts customizados (≥2 em blocos de código — ex.: prompt de mineração de decisões/requisitos/descartes da transcrição com timestamps, e prompt de checagem de consistência entre docs e código real); Iterações e ajustes (≥2 correções concretas e verídicas — ex.: matriz de erros genérica demais na primeira versão do FDD, corrigida para códigos específicos por caso; primeira versão do tracker com timestamps em formato de intervalo, corrigida para timestamp único); Como navegar a entrega (ordem sugerida: ADRs → RFC → FDD → PRD → TRACKER → README, com nota de que `TRANSCRICAO.md` na raiz é a fonte primária intocada).

## 7. Verificação final

Antes de considerar concluído, percorrer a checklist de "Critérios de Aceite" do `TASK.md` item a item contra os arquivos finais: contagem de FRs (≥8), contagem de riscos com probabilidade/impacto/mitigação (≥2 no PRD), alternativas/questões em aberto do RFC (≥2 cada) e ≥2 ADRs linkados, ≥4 endpoints no FDD com prefixo `WEBHOOK_*` na matriz de erros e ≥4 caminhos na seção de integração, 5–8 arquivos de ADR cada um com as 5 seções MADR e cobrindo ≥5/6 decisões, e a matemática de cobertura do tracker (≥80% / ≥70% TRANSCRICAO / ≥5 CODIGO) calculada numericamente, não por impressão. Nenhum arquivo de código citado pode ser fictício — todos os caminhos usados neste plano já foram confirmados diretamente no repositório.

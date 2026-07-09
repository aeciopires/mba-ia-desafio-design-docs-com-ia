<!-- TOC -->

- [FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos](#fdd--feature-design-document-sistema-de-webhooks-de-notificação-de-pedidos)
  - [1. Contexto e Motivação Técnica](#1-contexto-e-motivação-técnica)
  - [2. Objetivos Técnicos](#2-objetivos-técnicos)
  - [3. Escopo e Exclusões](#3-escopo-e-exclusões)
  - [4. Fluxos Detalhados](#4-fluxos-detalhados)
    - [4.1 Criação do evento na outbox](#41-criação-do-evento-na-outbox)
    - [4.2 Processamento pelo worker](#42-processamento-pelo-worker)
    - [4.3 Retry](#43-retry)
    - [4.4 Dead Letter Queue (DLQ)](#44-dead-letter-queue-dlq)
  - [5. Contratos Públicos](#5-contratos-públicos)
    - [5.1 `POST /api/v1/webhooks` — registrar um webhook](#51-post-apiv1webhooks--registrar-um-webhook)
    - [5.2 `PATCH /api/v1/webhooks/:id` — editar configuração](#52-patch-apiv1webhooksid--editar-configuração)
    - [5.3 `DELETE /api/v1/webhooks/:id` — remover configuração](#53-delete-apiv1webhooksid--remover-configuração)
    - [5.4 `GET /api/v1/webhooks?customerId=:customerId` — listar webhooks de um cliente](#54-get-apiv1webhookscustomeridcustomerid--listar-webhooks-de-um-cliente)
    - [5.5 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas](#55-get-apiv1webhooksiddeliveries--histórico-de-entregas)
    - [5.6 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret](#56-post-apiv1webhooksidrotate-secret--rotacionar-secret)
    - [5.7 `POST /admin/webhooks/dead-letter/:id/replay` — replay administrativo de DLQ](#57-post-adminwebhooksdead-letteridreplay--replay-administrativo-de-dlq)
    - [5.8 Headers de entrega enviados ao cliente (worker → endpoint do cliente)](#58-headers-de-entrega-enviados-ao-cliente-worker--endpoint-do-cliente)
    - [5.9 Payload do evento (worker → endpoint do cliente)](#59-payload-do-evento-worker--endpoint-do-cliente)
  - [6. Matriz de Erros (`WEBHOOK_*`)](#6-matriz-de-erros-webhook_)
  - [7. Estratégias de Resiliência](#7-estratégias-de-resiliência)
  - [8. Observabilidade](#8-observabilidade)
  - [9. Dependências e Compatibilidade](#9-dependências-e-compatibilidade)
  - [10. Critérios de Aceite Técnicos](#10-critérios-de-aceite-técnicos)
  - [11. Riscos e Mitigação](#11-riscos-e-mitigação)
  - [12. Integração com o Sistema Existente](#12-integração-com-o-sistema-existente)

<!-- TOC -->

# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

> Documento de implementação. Para a proposta arquitetural de mais alto nível, ver o [RFC](./RFC.md). Para o racional detalhado de cada decisão, ver os [ADRs](./adrs/).

## 1. Contexto e Motivação Técnica

O sistema hoje não possui nenhum mecanismo de notificação externa, eventos ou filas — clientes B2B dependem de polling em `GET /orders` (`[09:00] Marcos`). Esta feature adiciona um módulo de webhooks outbound que publica eventos de mudança de status de pedido, aproveitando a transação já existente em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) como ponto único de origem dos eventos.

Tecnicamente, a decisão central (ADR-001) é gravar o evento na tabela `webhook_outbox` **dentro** da mesma transação Prisma que já atualiza `orders`, insere em `order_status_history` e ajusta `stockQuantity`. Isso elimina qualquer janela de inconsistência entre "pedido mudou de status" e "evento foi registrado" (`[09:06] Diego`, `[09:40]-[09:41] Bruno/Diego`).

## 2. Objetivos Técnicos

- Garantir que a inserção do evento na outbox seja atômica com a transação de `changeStatus`: zero casos de mudança de status sem evento correspondente registrado (`[09:40]-[09:41] Bruno/Diego`).
- Entregar notificações dentro de <10 segundos (P95) na maioria dos casos, respeitando o requisito de negócio de "tempo real" definido como <10s (`[09:02] Marcos`), com o polling de 2s do worker (`[09:09] Diego`) como piso de latência mínima.
- Garantir at-least-once delivery com identificador único por evento (`X-Event-Id`) para suportar deduplicação do lado do cliente (`[09:24]-[09:26] Diego`).
- Limitar o retrabalho de reenvio a uma janela previsível (~15h) antes de mover eventos para DLQ (`[09:15]-[09:17] Diego`).
- Reaproveitar 100% dos componentes de infraestrutura transversal já existentes (logger, middleware de erro, autenticação) sem modificá-los (`[09:29]-[09:30]`, ver ADR-006).

## 3. Escopo e Exclusões

**Incluso**: CRUD de configuração de webhook, filtragem por status, outbox transacional, worker com polling e retry, DLQ com replay manual, assinatura HMAC-SHA256, rotação de secret, histórico de entregas.

**Excluído desta fase** (consistente com a seção "Fora de escopo" do [PRD](./PRD.md)):
- Alertas automáticos (e-mail ou outro canal) para clientes com falhas repetidas de entrega — adiado, `[09:37]-[09:38] Marcos/Larissa`.
- Rate limiting de saída por cliente — não implementado, apenas registrado como ponto de observação, `[09:38]-[09:39] Diego/Larissa`.
- Dashboard visual para o cliente acompanhar webhooks — fora de escopo, projeto separado do time de frontend, `[09:39]-[09:40] Marcos/Larissa`.
- Arquivamento de linhas entregues da outbox após 30 dias — mencionado como necessário eventualmente, mas fora do escopo desta feature, `[09:08] Diego`.
- Múltiplos workers em paralelo com particionamento/locking — "problema do futuro, não agora", `[09:13] Diego`.

## 4. Fluxos Detalhados

### 4.1 Criação do evento na outbox

1. Uma requisição autenticada chama `PATCH /api/v1/orders/:id/status`, que invoca `OrderService.changeStatus` (`order.service.ts:126`).
2. Dentro da transação já existente (`this.prisma.$transaction`), após `tx.order.update` (linha 158) e `tx.orderStatusHistory.create` (linha 159-167), uma nova chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` é adicionada (`[09:41] Bruno`: "função pura recebendo o tx", confirmado por Diego `[09:41]`).
3. `publishWebhookEvent` busca as configurações de webhook ativas do `customerId` do pedido que estejam inscritas no novo status (`toStatus`). O filtro por status é aplicado **neste momento de inserção**, não no envio: "Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela" (`[09:34] Bruno`, ver ADR-007).
4. Para cada configuração de webhook compatível, uma linha é inserida em `webhook_outbox` com: `id` (UUID, ver ADR-007), `webhookConfigId`, `eventId` (UUID, novo por linha), `status` (`pendente`), o payload JSON **já renderizado** (snapshot, ver ADR-007), `attemptCount = 0`, `createdAt`.
5. Se a inserção na outbox falhar por qualquer motivo, toda a transação (incluindo a mudança de status do pedido) sofre rollback — não existe caso de status mudado sem evento correspondente (`[09:40]-[09:41] Bruno/Diego`).

### 4.2 Processamento pelo worker

1. O processo `src/worker.ts` (novo entry point, mesmo padrão de `src/server.ts`) inicia com sua própria instância de `PrismaClient`, conectada ao mesmo `DATABASE_URL` (`[09:29]-[09:30] Diego/Bruno`).
2. A cada 2 segundos, o worker consulta `webhook_outbox` por linhas com status `pendente` (ou `falhou` com `nextAttemptAt <= now()`), ordenadas por `createdAt`, em lotes pequenos (`[09:07]-[09:09] Diego`).
3. Para cada evento, o worker monta a requisição HTTP: método `POST`, URL cadastrada no `webhook_config`, corpo = payload já renderizado, headers descritos na seção 5.3.
4. A assinatura HMAC-SHA256 é calculada sobre o corpo bruto do request usando a secret ativa do endpoint (ver ADR-004) e enviada em `X-Signature`.
5. A requisição tem timeout de 10 segundos (`[09:42] Diego`). Resposta 2xx dentro do timeout marca o evento como `entregue`. Qualquer outra resposta, erro de rede, ou timeout é tratado como falha.

### 4.3 Retry

1. Em caso de falha, o `attemptCount` é incrementado e o status volta para `falhou` com um `nextAttemptAt` calculado pela progressão de backoff: 1min / 5min / 30min / 2h / 12h (`[09:17] Diego`, ver ADR-003).
2. O worker só volta a tentar aquele evento quando `nextAttemptAt` for alcançado.
3. Esse ciclo se repete até 5 tentativas totais (`[09:15]-[09:16] Diego`).

### 4.4 Dead Letter Queue (DLQ)

1. Ao esgotar as 5 tentativas, o evento é removido de `webhook_outbox` e inserido em `webhook_dead_letter`, com payload, motivo da última falha e timestamp (`[09:18] Diego`, ver ADR-003).
2. Um administrador (role `ADMIN`) pode reprocessar manualmente via `POST /admin/webhooks/dead-letter/:id/replay`, que reinsere o evento em `webhook_outbox` com status `pendente` e `attemptCount = 0`, e registra em log quem executou a ação (`[09:18]-[09:19]`, `[09:35]-[09:36]`, ver ADR-008).

## 5. Contratos Públicos

Todos os endpoints (exceto o de replay administrativo) vivem sob `/api/v1/webhooks`, seguindo a convenção de montagem de rotas de `src/routes/index.ts`. Autenticação via `authenticate` (JWT de operador/sistema já existente — o cliente não tem JWT próprio, `[09:32] Marcos/Larissa`); o `customerId` é passado explicitamente no corpo/rota, não derivado do token (`[09:32] Larissa`).

### 5.1 `POST /api/v1/webhooks` — registrar um webhook

Requisito: `[09:31]-[09:32] Marcos`.

Request:
```json
{
  "customerId": "3f2a9e10-8b4d-4c1a-9b3e-1a2b3c4d5e6f",
  "url": "https://api.atlascomercial.com/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```
Headers: `Authorization: Bearer <jwt>`, `Content-Type: application/json`.

Response `201 Created`:
```json
{
  "id": "b1c2d3e4-f5a6-4b7c-8d9e-0f1a2b3c4d5e",
  "customerId": "3f2a9e10-8b4d-4c1a-9b3e-1a2b3c4d5e6f",
  "url": "https://api.atlascomercial.com/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_CHANGE_HERE",
  "active": true,
  "createdAt": "2026-07-09T14:32:00.000Z"
}
```
A `secret` só é retornada nesta resposta de criação (não fica recuperável depois, apenas rotacionável). Erros: `400 WEBHOOK_INVALID_URL` (URL não é `https`, `[09:23] Sofia`), `422 WEBHOOK_VALIDATION_ERROR`.

### 5.2 `PATCH /api/v1/webhooks/:id` — editar configuração

Requisito: `[09:33] Bruno`.

Request:
```json
{ "statuses": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: objeto do webhook atualizado (mesmo formato da criação, sem `secret`). Erros: `404 WEBHOOK_NOT_FOUND`, `422 WEBHOOK_VALIDATION_ERROR`.

### 5.3 `DELETE /api/v1/webhooks/:id` — remover configuração

Requisito: `[09:33] Bruno`.

Response `204 No Content`. Erros: `404 WEBHOOK_NOT_FOUND`.

### 5.4 `GET /api/v1/webhooks?customerId=:customerId` — listar webhooks de um cliente

Requisito: `[09:33] Bruno`.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "b1c2d3e4-f5a6-4b7c-8d9e-0f1a2b3c4d5e",
      "url": "https://api.atlascomercial.com/hooks/orders",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-09T14:32:00.000Z"
    }
  ]
}
```

### 5.5 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Requisito: últimos 100 registros, sucesso/falha, payload, resposta, tempo de resposta (`[09:34]-[09:35] Marcos`).

Response `200 OK`:
```json
{
  "data": [
    {
      "eventId": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
      "status": "entregue",
      "attemptCount": 1,
      "httpStatus": 200,
      "responseTimeMs": 184,
      "deliveredAt": "2026-07-09T14:32:04.100Z"
    },
    {
      "eventId": "f6a7b8c9-d0e1-4f2a-9b3c-4d5e6f7a8b9c",
      "status": "falhou",
      "attemptCount": 3,
      "httpStatus": 503,
      "responseTimeMs": 10000,
      "nextAttemptAt": "2026-07-09T15:02:10.000Z"
    }
  ]
}
```
Limitado às últimas 100 entregas por webhook (`[09:34]-[09:35] Marcos`). Erros: `404 WEBHOOK_NOT_FOUND`.

### 5.6 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Requisito: `[09:21]-[09:22] Sofia`.

Response `200 OK`:
```json
{
  "id": "b1c2d3e4-f5a6-4b7c-8d9e-0f1a2b3c4d5e",
  "secret": "whsec_CHANGE_HERE",
  "previousSecretValidUntil": "2026-07-10T14:32:00.000Z"
}
```
A secret anterior permanece válida por 24h (`[09:21] Sofia`). Erros: `404 WEBHOOK_NOT_FOUND`.

### 5.7 `POST /admin/webhooks/dead-letter/:id/replay` — replay administrativo de DLQ

Requisito: `[09:18]-[09:19] Diego`, autorização `[09:35]-[09:36] Sofia/Larissa`. Requer role `ADMIN` via `requireRole('ADMIN')` (ver seção 12).

Response `202 Accepted`:
```json
{ "eventId": "f6a7b8c9-d0e1-4f2a-9b3c-4d5e6f7a8b9c", "status": "pendente", "requeuedAt": "2026-07-09T16:00:00.000Z" }
```
Erros: `403 WEBHOOK_FORBIDDEN_REPLAY` (role diferente de ADMIN), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 5.8 Headers de entrega enviados ao cliente (worker → endpoint do cliente)

`[09:44]-[09:45] Diego/Sofia`:

| Header | Conteúdo |
| --- | --- |
| `X-Event-Id` | UUID único do evento (gerado na inserção da outbox) |
| `X-Signature` | HMAC-SHA256 do corpo, hex-encoded |
| `X-Timestamp` | Timestamp ISO 8601 do envio (detecção de replay attack do lado do cliente) |
| `X-Webhook-Id` | ID da configuração de webhook que originou o disparo |
| `Content-Type` | `application/json` |

### 5.9 Payload do evento (worker → endpoint do cliente)

Campos definidos em `[09:43]-[09:44] Diego/Bruno` — payload enxuto, sem `items` (cliente consulta `GET /orders/:id` se precisar de detalhes):
```json
{
  "event_id": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-09T14:32:00.500Z",
  "order_id": "9c8b7a6f-5e4d-4c3b-2a1f-0e9d8c7b6a5f",
  "order_number": "ORD-000482",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3f2a9e10-8b4d-4c1a-9b3e-1a2b3c4d5e6f",
  "total_cents": 15800
}
```

## 6. Matriz de Erros (`WEBHOOK_*`)

Todos seguem o padrão de subclasse de `AppError` já usado em `src/shared/errors/http-errors.ts` (mesmo padrão de `InvalidStatusTransitionError extends ConflictError` e `InsufficientStockError extends UnprocessableEntityError`).

| Código | HTTP Status | Quando ocorre |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook config referenciado não existe |
| `WEBHOOK_INVALID_URL` | 422 | URL cadastrada não é `https` (`[09:23] Sofia`) |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Operação exige secret ativa e nenhuma foi encontrada |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB — rejeitado sem truncar (`[09:23]-[09:24] Sofia/Diego/Larissa`) |
| `WEBHOOK_DUPLICATE_CONFIG` | 409 | Já existe configuração ativa idêntica (mesma URL+customer+status) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item de DLQ referenciado no replay não existe |
| `WEBHOOK_FORBIDDEN_REPLAY` | 403 | Usuário autenticado não possui role `ADMIN` no endpoint de replay (`[09:36] Sofia`) |
| `WEBHOOK_VALIDATION_ERROR` | 422 | Falha de validação Zod nos campos do payload de entrada |

## 7. Estratégias de Resiliência

- **Timeout**: 10 segundos por chamada HTTP do worker ao endpoint do cliente (`[09:42] Diego`).
- **Retry com backoff exponencial**: 5 tentativas, intervalos 1m/5m/30m/2h/12h (`[09:15]-[09:17] Diego`, ADR-003).
- **Fallback terminal**: DLQ (`webhook_dead_letter`) para eventos que esgotam as tentativas, com replay manual disponível (`[09:18]-[09:19] Diego`, ADR-008).
- **Isolamento de processo**: falha ou reinício da API não interrompe o worker, e vice-versa (`[09:11] Diego`, ADR-002).
- **At-least-once em vez de exactly-once**: aceito deliberadamente como trade-off de simplicidade (`[09:24]-[09:26] Diego`, ADR-005) — a resiliência do sistema não depende de garantir entrega única, apenas entrega eventual com idempotência do lado do cliente via `X-Event-Id`.

## 8. Observabilidade

- **Métricas**: profundidade da outbox (contagem de eventos `pendente`), taxa de sucesso/falha de entrega, distribuição de tentativas por número de retry, tamanho da DLQ, latência do ciclo de polling do worker.
- **Logs**: reuso do logger Pino singleton (`src/shared/logger/index.ts`) tanto na API quanto no worker, seguindo a convenção de eventos em snake_case já usada em `src/server.ts` (`server_started`, `shutdown_initiated`) e `src/middlewares/error.middleware.ts`. Eventos estruturados propostos: `webhook_event_enqueued`, `webhook_delivery_attempted`, `webhook_delivery_failed`, `webhook_delivery_succeeded`, `webhook_dlq_entered`, `webhook_replay_triggered` (este último incluindo o `userId` do administrador, para auditoria — `[09:36] Sofia`).
- **Tracing**: o projeto não possui hoje infraestrutura de tracing distribuído (APM/OpenTelemetry) — não introduzimos essa dependência nesta feature. A rastreabilidade fim a fim de um evento (da inserção na outbox até a tentativa de entrega, passando por eventuais retries) é obtida por correlação via `eventId`/`X-Event-Id`, presente em todo log estruturado relacionado a um mesmo evento.

## 9. Dependências e Compatibilidade

- Node.js ≥20, TypeScript 5.6, ESM (`"type": "module"`) — mesmas versões já usadas no projeto (`package.json`).
- Prisma 5.22 / MySQL — mesmo banco e ORM, com novas tabelas (`webhook_config`, `webhook_outbox`, `webhook_dead_letter`, `webhook_delivery_log`) adicionadas via migration, sem alterar tabelas existentes.
- `uuid` (11.0.3) já é dependência do projeto — reaproveitada para gerar `eventId` e chaves primárias UUID (ADR-007).
- HMAC-SHA256 é implementado com o módulo nativo `crypto` do Node.js — nenhuma dependência nova necessária.
- Nenhuma mudança de compatibilidade é exigida nos módulos existentes (`orders`, `users`, `customers`, `products`); a única alteração em código já existente é a chamada adicional a `publishWebhookEvent` dentro de `OrderService.changeStatus`.

## 10. Critérios de Aceite Técnicos

- Inserir um evento na outbox nunca ocorre fora da transação de `changeStatus`; um teste de integração deve provocar falha na inserção da outbox e verificar que a mudança de status também sofre rollback.
- O worker respeita o intervalo de polling de 2s e processa eventos em ordem de `createdAt` por `order_id`.
- Uma sequência de 5 falhas consecutivas de entrega move o evento para `webhook_dead_letter` com o motivo da última falha registrado.
- A assinatura em `X-Signature` é verificável do lado do cliente usando HMAC-SHA256 sobre o corpo bruto e a secret ativa.
- Um payload de evento maior que 64KB é rejeitado na inserção com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem truncamento.
- O endpoint de replay retorna `403 WEBHOOK_FORBIDDEN_REPLAY` para usuários sem role `ADMIN`.
- Rotação de secret mantém a secret anterior válida por exatamente 24h após a rotação.

## 11. Riscos e Mitigação

| Risco | Mitigação técnica |
| --- | --- |
| Cliente não deduplica corretamente eventos at-least-once | `X-Event-Id` estável por evento em todas as tentativas de reenvio; documentação explícita no portal de desenvolvedor (`[09:26] Marcos`) |
| Outage de cliente maior que a janela de retry (~15h) gera acúmulo silencioso na DLQ | Métrica de tamanho da DLQ (seção 8) + endpoint de replay manual (ADR-008); alerta automático fica para fase futura (`[09:37]-[09:38]`) |
| Vazamento de secret | Secret por endpoint (não global) + rotação com grace period de 24h (ADR-004), motivada por incidente real anterior (`[09:22] Diego`) |
| Worker único como ponto de gargalo/latência sob alto volume | Limitação documentada (ADR-002); escala horizontal fica como questão em aberto no RFC |

## 12. Integração com o Sistema Existente

Esta seção nomeia os pontos reais do código-base que o módulo de webhooks estende ou reaproveita, sem alterar comportamento de módulos existentes fora do ponto 1.

1. **`src/modules/orders/order.service.ts`** — o método `OrderService.changeStatus` (linhas 126-179) já executa `tx.order.update` (linha 158) e `tx.orderStatusHistory.create` (linhas 159-167) dentro de `this.prisma.$transaction`. A única alteração de comportamento em código existente introduzida por esta feature é a adição de uma chamada a `publishWebhookEvent(tx, order, from, to)` logo após essas duas operações, ainda dentro da mesma transação, para que a inserção do evento compartilhe atomicidade com a mudança de status (`[09:40]-[09:41] Bruno/Diego`).
2. **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`** — as novas classes de erro do módulo (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError`/`ConflictError`/`NotFoundError`/`UnprocessableEntityError` exatamente como `InvalidStatusTransitionError extends ConflictError` (linha 45) e `InsufficientStockError extends UnprocessableEntityError` (linha 55) já fazem hoje, introduzindo o prefixo `WEBHOOK_*` (ver ADR-006).
3. **`src/middlewares/error.middleware.ts`** — não recebe nenhuma alteração. O tratamento genérico `err instanceof AppError` (linha 15) já formata corretamente qualquer nova subclasse de erro do módulo de webhooks, incluindo os códigos `WEBHOOK_*`.
4. **`src/middlewares/auth.middleware.ts`** — o endpoint `POST /admin/webhooks/dead-letter/:id/replay` reaproveita `authenticate` e `requireRole('ADMIN')` (linhas 27-61) sem modificação, no mesmo padrão de uso já visto em `src/modules/users/user.routes.ts:15`.
5. **`src/shared/logger/index.ts`** — tanto os controllers/services do módulo de webhooks quanto o processo `src/worker.ts` importam o mesmo logger Pino singleton exportado por este arquivo, seguindo a convenção de eventos estruturados já usada no projeto.
6. **`src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`)** — o novo módulo é instanciado seguindo o mesmo padrão de injeção manual de dependências (`Repository -> Service -> Controller`) já usado para `orders`, `users`, `customers` e `products`, e montado como `/api/v1/webhooks` no mesmo `buildApiRouter`.
7. **`src/server.ts`** — o novo entry point `src/worker.ts` espelha a estrutura de bootstrap (`bootstrap()`, tratamento de `SIGINT`/`SIGTERM`, `logger.fatal` em caso de falha de inicialização) já usada em `server.ts`, mas como processo independente com script próprio `npm run worker` no `package.json`.
8. **`prisma/schema.prisma`** — o modelo `OrderStatusHistory` (linhas 116-131), com seus campos de auditoria (`changedAt`, `changedById`, índices em `orderId` e `changedAt`), é o precedente estilístico direto para o desenho dos novos modelos `webhook_outbox` e `webhook_dead_letter`: identificadores UUID (ver ADR-007), timestamps indexados, e relação com a entidade de origem.

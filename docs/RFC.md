<!-- TOC -->

- [RFC — Sistema de Webhooks de Notificação de Pedidos](#rfc--sistema-de-webhooks-de-notificação-de-pedidos)
  - [Metadados](#metadados)
  - [TL;DR](#tldr)
  - [Contexto e Problema](#contexto-e-problema)
  - [Proposta Técnica](#proposta-técnica)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Questões em Aberto](#questões-em-aberto)
  - [Impacto e Riscos](#impacto-e-riscos)
  - [Decisões Relacionadas](#decisões-relacionadas)

<!-- TOC -->

# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) — documento consolidado a partir da reunião técnica de definição da feature |
| Status | Em revisão |
| Data | 2026-07-09 |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| Documentos relacionados | [PRD](./PRD.md), [FDD](./FDD.md), ADRs [001](./adrs/ADR-001-outbox-pattern-no-mysql.md)–[008](./adrs/ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md) |

## TL;DR

Vamos construir um sistema de webhooks outbound para notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre mudanças de status de pedidos, substituindo o polling atual em `GET /orders`. A arquitetura usa o padrão Outbox no MySQL já existente — o evento é gravado atomicamente dentro da transação de `OrderService.changeStatus` — e um worker em processo separado, em polling de 2 segundos, é responsável por entregar os eventos via HTTP, com retry exponencial, Dead Letter Queue (DLQ) para falhas permanentes, assinatura HMAC-SHA256 e garantia de entrega at-least-once. A proposta reaproveita deliberadamente os padrões já existentes no projeto (módulos, classes de erro, logger, middleware de autenticação).

## Contexto e Problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente ser notificados em tempo real quando o status de seus pedidos muda na plataforma (`[09:00] Marcos`). Hoje, a integração desses clientes depende de polling manual em `GET /orders`, o que os próprios clientes descrevem como lento e caro de operar (`[09:00] Marcos`). A Atlas Comercial sinalizou risco de churn para um concorrente caso a entrega não ocorra até o fim do trimestre (`[09:00] Marcos`).

O requisito de latência foi esclarecido: "qualquer coisa abaixo de 10 segundos já é 'tempo real'" para os clientes (`[09:02] Marcos`) — não é necessário um mecanismo de push instantâneo, apenas eliminar a necessidade de polling manual e manter uma latência de notificação previsível e baixa.

O ponto de integração crítico é o método `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que já executa, dentro de uma única transação Prisma, a atualização do status do pedido, o registro em `order_status_history` e o ajuste de estoque. Qualquer solução de notificação precisa respeitar essa transação sem degradá-la.

## Proposta Técnica

A proposta combina seis decisões arquiteturais centrais, cada uma detalhada em seu próprio ADR:

1. **Outbox transacional no MySQL** ([ADR-001](./adrs/ADR-001-outbox-pattern-no-mysql.md)): o evento de mudança de status é gravado em uma tabela `webhook_outbox` dentro da mesma transação de `changeStatus`. Isso garante que o evento só existe se a mudança de status foi de fato commitada, sem exigir uma chamada HTTP síncrona dentro da transação de negócio.
2. **Worker separado em polling de 2 segundos** ([ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md)): um novo processo Node.js (`src/worker.ts`), independente da API, varre a outbox periodicamente e realiza as entregas HTTP. A ordenação de entrega é garantida apenas por `order_id`, e apenas enquanto houver um único worker ativo — limitação documentada, não um requisito não atendido.
3. **Retry com backoff exponencial e Dead Letter Queue** ([ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)): até 5 tentativas de entrega, com intervalos crescentes; eventos que esgotam as tentativas são movidos para uma tabela `webhook_dead_letter`, reprocessável manualmente por um endpoint administrativo.
4. **Autenticação HMAC-SHA256 com secret por endpoint** ([ADR-004](./adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)): cada requisição de webhook é assinada, permitindo ao cliente validar origem e integridade; secrets são isoladas por endpoint cadastrado e suportam rotação com grace period.
5. **Entrega at-least-once com deduplicação via `X-Event-Id`** ([ADR-005](./adrs/ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md)): o sistema não garante exactly-once; cada evento carrega um identificador único para que o cliente deduplique do lado dele.
6. **Reuso máximo dos padrões existentes do projeto** ([ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)): o novo módulo `src/modules/webhooks/` segue a mesma estrutura controller/service/repository/routes/schemas dos módulos existentes, reaproveita a hierarquia de `AppError`, o logger Pino e o middleware de erro centralizado sem modificá-los.

Em termos de superfície funcional, o módulo expõe operações de CRUD para configuração de webhooks (registro, edição, remoção, listagem), filtragem de eventos por status por endpoint, consulta de histórico de entregas, rotação de secret e um endpoint administrativo restrito para reprocessamento manual da DLQ (detalhes completos de contratos, payloads e códigos de erro ficam no [FDD](./FDD.md) — este RFC não repete esse nível de detalhe).

## Alternativas Consideradas

1. **Despacho síncrono dentro da transação de `changeStatus`** — descartada. A transação de mudança de status já é pesada (atualiza pedido, histórico e estoque); adicionar uma chamada HTTP síncrona nela faria com que um cliente lento ou indisponível bloqueasse mudanças de status de outros pedidos, sem uma semântica sã de rollback em caso de falha do cliente (`[09:04] Bruno`, `[09:06] Diego`). Trade-off: simplicidade de implementação vs. risco real de degradação de disponibilidade do core de pedidos — o risco pesou mais.
2. **Fila externa dedicada (ex.: Redis Streams)** — descartada. Resolveria o mesmo problema de desacoplamento, mas exigiria subir e operar infraestrutura nova (ex.: um cluster Redis) para um time pequeno, quando o MySQL já em produção resolve o mesmo problema via padrão outbox (`[09:07] Larissa`, `[09:07] Diego`). Trade-off: menor esforço operacional futuro de uma fila dedicada vs. custo imediato de infraestrutura adicional — o custo imediato pesou mais para a equipe.

## Questões em Aberto

1. **Rate limiting de saída para clientes**: se um cliente tiver muitos pedidos mudando de status em um curto intervalo (ex.: 50 em um minuto), o sistema hoje não limita a taxa de chamadas HTTP disparadas para aquele cliente. Diego levantou o ponto (`[09:38]`) e a equipe decidiu não endereçar agora: "a gente observa e implementa se virar problema" (`[09:39] Diego`), registrado como "observar e decidir depois" (`[09:39] Larissa`). Sem dono nem prazo definidos.
2. **Escala para múltiplos workers em paralelo**: a garantia de ordenação por `order_id` depende de um único worker ativo. Escalar horizontalmente exigiria particionamento por `order_id` ou lock pessimista, explicitamente classificado como "problema do futuro, não agora" (`[09:13] Diego`), sem desenho definido.
3. **Hardening de autorização do CRUD de configuração**: hoje qualquer role autenticada pode gerenciar configurações de webhook (diferente do endpoint de replay da DLQ, restrito a `ADMIN`). Sofia sinalizou que isso pode mudar no futuro — "por enquanto sim... mais pra frente a gente pode endurecer" (`[09:37] Sofia`) — sem gatilho ou critério definido para quando isso deve acontecer.

## Impacto e Riscos

A feature introduz um novo processo em produção (o worker), uma nova superfície de API exposta a clientes externos, e uma nova responsabilidade de gestão de secrets. Os principais riscos técnicos — deduplicação incorreta do lado do cliente, acúmulo silencioso de eventos na DLQ sem alerta automático, e vazamento de secret — têm mitigação descrita em detalhe no [PRD](./PRD.md) (seção de Riscos) e no [FDD](./FDD.md) (seção de Resiliência e Observabilidade). Do ponto de vista de cronograma, a equipe estimou 3 sprints, incluindo uma revisão de segurança dedicada da Sofia de pelo menos 2 dias úteis antes do deploy em produção (`[09:46]-[09:49]`).

## Decisões Relacionadas

- [ADR-001 — Outbox Pattern no MySQL](./adrs/ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002 — Worker em Processo Separado com Polling](./adrs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003 — Retry com Backoff Exponencial e Dead Letter Queue](./adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)
- [ADR-004 — Autenticação HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Entrega At-Least-Once com Deduplicação via X-Event-Id](./adrs/ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md)
- [ADR-006 — Reuso dos Padrões Existentes do Projeto](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Outbox com Chave Primária UUID e Payload Snapshot na Inserção](./adrs/ADR-007-outbox-com-chave-primaria-uuid-e-payload-snapshot-na-insercao.md)
- [ADR-008 — Endpoint Administrativo de Replay da DLQ com Role ADMIN e Auditoria](./adrs/ADR-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria.md)

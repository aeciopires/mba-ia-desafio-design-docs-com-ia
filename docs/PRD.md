<!-- TOC -->

- [PRD — Sistema de Webhooks de Notificação de Pedidos](#prd--sistema-de-webhooks-de-notificação-de-pedidos)
  - [Resumo e Contexto da Feature](#resumo-e-contexto-da-feature)
  - [Problema e Motivação](#problema-e-motivação)
  - [Público-Alvo e Cenários de Uso](#público-alvo-e-cenários-de-uso)
  - [Objetivos e Métricas de Sucesso](#objetivos-e-métricas-de-sucesso)
  - [Escopo](#escopo)
    - [Incluso](#incluso)
    - [Fora de Escopo](#fora-de-escopo)
  - [Requisitos Funcionais](#requisitos-funcionais)
  - [Requisitos Não Funcionais](#requisitos-não-funcionais)
  - [Decisões e Trade-offs Principais](#decisões-e-trade-offs-principais)
  - [Dependências](#dependências)
  - [Riscos e Mitigação](#riscos-e-mitigação)
  - [Critérios de Aceitação](#critérios-de-aceitação)
  - [Estratégia de Testes e Validação](#estratégia-de-testes-e-validação)

<!-- TOC -->

# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O Order Management System (OMS) hoje não possui nenhum mecanismo de notificação externa, eventos ou filas. Clientes B2B que integram com a plataforma precisam consultar `GET /orders` periodicamente para descobrir mudanças de status de seus pedidos. Esta feature adiciona um sistema de webhooks outbound: sempre que o status de um pedido muda, o sistema notifica proativamente, via HTTP, os endpoints previamente cadastrados pelo cliente, eliminando a necessidade de polling.

## Problema e Motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente serem notificados em tempo real quando o status de seus pedidos muda (`[09:00] Marcos`). O modelo atual de polling em `GET /orders` foi descrito pelo próprio time de produto como algo que "tá deixando a integração lenta e cara" para os clientes (`[09:00] Marcos`). O risco de negócio é concreto: a Atlas Comercial sinalizou que pode migrar para um concorrente caso a entrega não ocorra até o fim do trimestre (`[09:00] Marcos`).

## Público-Alvo e Cenários de Uso

- **Clientes B2B integradores** (times de engenharia de Atlas Comercial, MaxDistribuição, Nova Cargo e futuros clientes com o mesmo perfil de integração): consomem a API para cadastrar endpoints de webhook, escolher quais mudanças de status desejam acompanhar, e implementam do lado deles a recepção, validação de assinatura e deduplicação dos eventos recebidos.
- **Usuários operadores/administradores internos** (via JWT do sistema, não JWT do cliente — `[09:32] Marcos/Larissa`): cadastram e mantêm as configurações de webhook em nome dos clientes, e (no caso de administradores) reprocessam manualmente eventos presos na Dead Letter Queue.

Cenário principal: o time de integração da Atlas Comercial cadastra um endpoint HTTPS via `POST /api/v1/webhooks`, seleciona os status `SHIPPED` e `DELIVERED`, e passa a receber automaticamente uma chamada HTTP assinada sempre que um pedido deles muda para um desses status — sem precisar mais consultar `GET /orders` em loop.

## Objetivos e Métricas de Sucesso

- **Objetivo 1 (quantitativo)**: notificação de mudança de status entregue ao cliente em menos de 10 segundos (P95), medido do momento do commit da transação de `changeStatus` até a primeira tentativa de entrega HTTP. Meta baseada diretamente no requisito definido pelos clientes: "qualquer coisa abaixo de 10 segundos já é 'tempo real'" (`[09:02] Marcos`).
- **Objetivo 2 (qualitativo)**: eliminar a necessidade de polling manual em `GET /orders` pelos três clientes que solicitaram a feature, resolvendo a queixa de integração "lenta e cara" (`[09:00] Marcos`).
- **Objetivo 3 (qualitativo)**: entregar a feature dentro do prazo estimado de 3 sprints, incluindo revisão de segurança dedicada, para atender o compromisso assumido com a Atlas Comercial de entrega até o fim do trimestre (`[09:00] Marcos`, `[09:45]-[09:47] Larissa/Sofia`).

> Nota de rastreabilidade: nenhuma métrica de taxa de sucesso de entrega (ex.: "99.9% de entregas bem-sucedidas") foi discutida na reunião. Não incluímos esse tipo de meta aqui para não inventar um número sem origem identificável — ver [TRACKER](./TRACKER.md).

## Escopo

### Incluso

- Cadastro, edição, remoção e listagem de configurações de webhook por cliente.
- Filtragem de eventos por status de pedido, por endpoint de webhook cadastrado.
- Notificação outbound assíncrona via HTTP, assinada com HMAC-SHA256.
- Garantia de entrega at-least-once, com retry automático e Dead Letter Queue para falhas persistentes.
- Endpoint de histórico de entregas (últimos 100 registros) por webhook.
- Rotação de secret por endpoint, com grace period.
- Endpoint administrativo de reprocessamento manual de eventos presos na DLQ.

### Fora de Escopo

- **Alertas automáticos para o cliente em caso de falhas repetidas de entrega** (ex.: e-mail após 3 falhas seguidas) — explicitamente adiado para uma fase futura, após medição de impacto (`[09:37]-[09:38] Marcos/Larissa`).
- **Rate limiting de saída por cliente** (proteção contra um cliente receber muitas chamadas em rajada) — não implementado nesta fase; equipe decidiu "observar e decidir depois" (`[09:38]-[09:39] Diego/Larissa`).
- **Dashboard visual para o cliente acompanhar seus webhooks** — descartado para esta fase; seria um projeto separado do time de frontend, escopo atual é API-only (`[09:39]-[09:40] Marcos/Larissa`).
- **Arquivamento de eventos entregues da outbox após 30 dias** — mencionado como necessário eventualmente, mas fora do escopo desta feature (`[09:08] Diego`).
- **Múltiplos workers em paralelo / escala horizontal do processamento** — tratado como "problema do futuro, não agora" (`[09:13] Diego`).

## Requisitos Funcionais

1. O cliente (via usuário operador/sistema autenticado) deve conseguir cadastrar um webhook informando URL, lista de status desejados e `customer_id`; a secret é gerada pelo sistema e devolvida apenas na criação (`[09:31]-[09:32] Marcos`).
2. O cadastro de webhook é feito via API autenticada com o JWT de operador/sistema já existente, não um JWT próprio do cliente; `customer_id` é passado explicitamente, não derivado do token (`[09:32] Marcos/Larissa`).
3. O cliente deve conseguir editar uma configuração de webhook existente via `PATCH` (`[09:33] Bruno`).
4. O cliente deve conseguir remover uma configuração de webhook via `DELETE` (`[09:33] Bruno`).
5. O cliente deve conseguir listar os webhooks cadastrados de um determinado `customer_id` via `GET` (`[09:33] Bruno`).
6. Cada webhook deve poder ser configurado para receber apenas um subconjunto de status de pedido (ex.: apenas `SHIPPED` e `DELIVERED`) (`[09:33]-[09:34] Marcos`).
7. O cliente deve conseguir consultar o histórico das últimas 100 entregas de um webhook, incluindo sucesso/falha, payload, resposta recebida e tempo de resposta (`[09:34]-[09:35] Marcos`).
8. Um administrador (role `ADMIN`) deve conseguir reprocessar manualmente um evento que caiu na Dead Letter Queue, via endpoint dedicado, com a ação registrada para auditoria (`[09:18]-[09:19] Diego`, `[09:35]-[09:36] Sofia/Larissa`).
9. O cliente deve conseguir solicitar rotação da secret de um webhook via API, mantendo a secret anterior válida por 24h em paralelo (`[09:21]-[09:22] Sofia`).
10. As demais operações de CRUD de configuração de webhook (fora do replay de DLQ) devem ser acessíveis a qualquer role autenticada nesta fase, sem restrição adicional (`[09:36]-[09:37] Marcos/Sofia`).

## Requisitos Não Funcionais

- **Latência**: notificação considerada "tempo real" para o negócio quando entregue em menos de 10 segundos (`[09:02] Marcos`); arquitetura de polling de 2s comfortably atende esse piso (`[09:09] Diego`).
- **Consistência**: a inserção do evento de notificação deve ser atômica com a transação de mudança de status do pedido — nunca deve existir mudança de status sem evento correspondente registrado (`[09:40]-[09:41] Bruno/Diego`).
- **Garantia de entrega**: at-least-once, não exactly-once; o cliente é responsável por deduplicar via identificador único do evento (`[09:24]-[09:26] Diego`).
- **Segurança em trânsito**: URLs de webhook cadastradas devem ser obrigatoriamente HTTPS (`[09:23] Sofia`).
- **Segurança de integridade/autenticidade**: toda entrega deve ser assinada com HMAC-SHA256, permitindo ao cliente validar origem e integridade do payload (`[09:19]-[09:20] Sofia`).
- **Gestão de segredo**: secret exclusiva por endpoint (não global), com suporte a rotação e grace period de 24h (`[09:21]-[09:22] Sofia`).
- **Limite de payload**: eventos maiores que 64KB são rejeitados com erro explícito, sem truncamento (`[09:23]-[09:24] Sofia/Diego/Larissa`).
- **Timeout de entrega**: chamadas HTTP do worker ao cliente têm timeout de 10 segundos (`[09:42] Diego`).
- **Isolamento operacional**: o processo responsável por disparar as notificações roda separado da API, de forma que um restart da API não interrompa o processamento de eventos (`[09:11] Diego`).
- **Auditabilidade**: toda ação de replay manual de DLQ deve ser registrada com o usuário responsável (`[09:36] Sofia`).

## Decisões e Trade-offs Principais

As decisões arquiteturais centrais estão documentadas em detalhe nos ADRs correspondentes, e não são repetidas aqui em profundidade:

- Outbox transacional no MySQL em vez de despacho síncrono ou fila externa dedicada — ver [ADR-001](./adrs/ADR-001-outbox-pattern-no-mysql.md).
- Worker em processo separado com polling de 2s em vez de reação via trigger de banco — ver [ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- Retry de 5 tentativas com backoff exponencial e DLQ em tabela própria — ver [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md).
- HMAC-SHA256 com secret por endpoint e rotação — ver [ADR-004](./adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md).
- At-least-once com deduplicação via `X-Event-Id` em vez de exactly-once — ver [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-deduplicacao-via-x-event-id.md).

## Dependências

- Três clientes externos (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam implementar, do lado deles, verificação de assinatura HMAC-SHA256 e deduplicação por `X-Event-Id` — dependência de terceiros fora do controle direto da equipe.
- A extensão de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) é pré-requisito técnico para qualquer evento ser gerado.
- Revisão de segurança dedicada da Sofia (mínimo 2 dias úteis) antes do deploy em produção, com foco em HMAC e geração de secret (`[09:46] Sofia`).
- Autenticação existente (JWT de operador/sistema) é reaproveitada — não há dependência de um novo mecanismo de identidade para clientes.

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente externo não implementa deduplicação corretamente e processa o mesmo evento de pedido mais de uma vez | Média | Alto (erro de negócio do lado do cliente) | `X-Event-Id` estável e documentado com destaque no portal de desenvolvedor (`[09:25]-[09:26] Diego/Marcos`) |
| Outage prolongado de um cliente (acima da janela de retry de ~15h) esgota as tentativas e gera acúmulo de eventos na DLQ sem alerta automático, já que alertas ficaram fora de escopo desta fase (`[09:37]-[09:38]`) | Média (há precedente real de outage de 2h, `[09:16] Diego`) | Médio (pedidos "presos" sem notificação até replay manual) | Endpoint de replay manual (`[09:18]-[09:19]`) + monitoramento do tamanho da DLQ via métricas (ver FDD, seção de Observabilidade) |
| Vazamento de secret de um cliente (já ocorreu antes, motivou a decisão de secret por endpoint) | Baixa-média (há precedente real, `[09:22] Diego`) | Alto (permitiria forjar webhooks válidos para o cliente afetado) | Secret exclusiva por endpoint + rotação com grace period de 24h (`[09:21]-[09:22] Sofia`) + revisão de segurança dedicada antes do deploy (`[09:46] Sofia`) |

## Critérios de Aceitação

- Um cliente cadastrado recebe notificação de mudança de status em menos de 10 segundos (P95) para os status configurados.
- Nenhuma mudança de status de pedido ocorre sem um evento correspondente registrado na outbox (verificável via teste de rollback conjunto).
- Um evento que falha 5 vezes é movido para a DLQ e pode ser reprocessado manualmente por um usuário `ADMIN`.
- A assinatura HMAC-SHA256 enviada é verificável pelo cliente usando a secret ativa do endpoint.
- Rotação de secret mantém a secret anterior válida por exatamente 24h.
- O endpoint de histórico de entregas retorna corretamente as últimas 100 entregas de um webhook, com status, payload, resposta e tempo de resposta.

## Estratégia de Testes e Validação

Seguir o padrão de testes de integração já estabelecido no projeto: testes via `supertest` contra `buildApp({ prisma })` com banco real (não mockado), como já implementado em `tests/orders.test.ts` e `tests/auth.test.ts`, usando as factories existentes em `tests/helpers/factories.ts` (`bootstrapAuthenticatedUser`, `createTestCustomer`, `createTestProduct`) mais novas factories equivalentes para configuração de webhook. Os testes devem cobrir, no mínimo: criação de webhook e devolução de secret única na criação; filtragem por status na inserção da outbox; transição de status de pedido gerando evento correspondente na mesma transação; falha de entrega avançando corretamente pela progressão de retry; movimentação para DLQ após 5 falhas; replay de DLQ restrito a `ADMIN`; rejeição de payload acima de 64KB. Adicionalmente, a revisão de segurança dedicada da Sofia (mínimo 2 dias úteis, `[09:46] Sofia`) cobre especificamente a geração e verificação de HMAC e o fluxo de rotação de secret antes do deploy em produção.

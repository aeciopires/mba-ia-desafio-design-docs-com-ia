<!-- TOC -->

- [ADR-001: Outbox Pattern no MySQL para publicação de eventos de webhook](#adr-001-outbox-pattern-no-mysql-para-publicação-de-eventos-de-webhook)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-001: Outbox Pattern no MySQL para publicação de eventos de webhook

## Status

Aceito — `[09:08] Larissa`: "Tá decidido então: outbox em MySQL."

## Contexto

O sistema de webhooks precisa notificar clientes externos sempre que o status de um pedido muda, sem comprometer a confiabilidade nem a performance da transação de mudança de status já existente em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`). Essa transação hoje atualiza `orders`, insere em `order_status_history` e ajusta `stockQuantity` de produtos — tudo dentro de um único `this.prisma.$transaction`.

Larissa levantou a pergunta inicial em `[09:03]`: "a gente dispara isso sincronamente no service de orders quando o status muda, ou faz algum tipo de fila/outbox?". Bruno respondeu em `[09:04]`: "Síncrono não rola. A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido. Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos", complementando em `[09:04]`: "Sem falar que se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá."

Diego, ao entrar na call, reforça em `[09:06]`: "Síncrono está fora de questão. Aliás, eu nem chamaria de 'fila' — o que a gente quer aqui é um padrão outbox", explicando o mecanismo: "quando o status do pedido muda, dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox com o evento. Um worker separado fica lendo essa tabela e disparando as chamadas HTTP. Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível" `[09:06]`.

## Decisão

Adotar o padrão Outbox Transacional usando o MySQL já existente no projeto: a inserção do evento de webhook (na futura tabela `webhook_outbox`) acontece **dentro da mesma transação** de `OrderService.changeStatus`, junto com o `tx.order.update` e o `tx.orderStatusHistory.create` (`order.service.ts:158-167`). Um worker separado (ver ADR-002) é responsável por ler a tabela e efetivamente disparar as chamadas HTTP, de forma assíncrona e desacoplada da transação de negócio.

A tabela é indexada por status (`pendente`, `processando`, `falhou`, `entregue`) e por `created_at`, conforme detalhado por Diego em `[09:08]`: "A tabela tem índice no campo de status ... e em created_at. Worker lê só os pendentes em batch pequeno, processa, marca como entregue." O arquivamento de linhas entregues após ~30 dias fica fora do escopo desta feature (`[09:08] Diego`: "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature").

## Alternativas Consideradas

1. **Despacho síncrono dentro da transação de `changeStatus`** — descartado. Bloquearia a transação (que já mexe em estoque e histórico) enquanto aguarda resposta HTTP de um cliente externo, e não há uma semântica sã de rollback caso o cliente esteja fora do ar (`[09:04] Bruno`).
2. **Redis Streams ou infraestrutura de fila dedicada** — levantada por Larissa em `[09:07]`: "A alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra." Descartada por Diego em `[09:07]`: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" — o trade-off decisivo foi custo operacional de nova infraestrutura vs. reaproveitamento do banco já em produção.

## Consequências

**Positivas**
- Consistência forte entre mudança de status e criação do evento: commit da transação = evento garantido; rollback = evento nunca existiu (`[09:06] Diego`).
- Nenhuma infraestrutura nova a operar — reaproveita o MySQL e o `PrismaClient` já existentes no projeto.
- Índices em `status` e `created_at` mantêm o worker eficiente mesmo com acúmulo de eventos (`[09:08] Diego`).

**Negativas**
- Entrega passa a ser assíncrona por natureza: o cliente externo só é notificado depois que um worker separado processa a linha da outbox — não há entrega no instante exato do commit (mitigado pelo polling de 2s definido no ADR-002, dentro do requisito de <10s de `[09:02] Marcos`).
- Introduz a necessidade de um processo consumidor (worker) para operar e monitorar, o que não existia antes no projeto.
- Arquivamento de eventos entregues após 30 dias fica como responsabilidade futura, não coberta por esta decisão (`[09:08] Diego`).

<!-- TOC -->

- [ADR-008: Endpoint Administrativo de Replay da DLQ com Role ADMIN e Auditoria](#adr-008-endpoint-administrativo-de-replay-da-dlq-com-role-admin-e-auditoria)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-008: Endpoint Administrativo de Replay da DLQ com Role ADMIN e Auditoria

## Status

Aceito — `[09:36] Larissa`: "Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe."

## Contexto

O endpoint de replay manual da Dead Letter Queue foi proposto por Diego em `[09:18]`: "Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente" (ver ADR-003 para o contexto completo da DLQ). Larissa retomou o ponto de controle de acesso mais adiante, perguntando em `[09:35]`: "Quem é admin? Tem que ser role ADMIN do JWT?". Sofia respondeu de forma categórica em `[09:36]`: "Tem que ser ADMIN sim. Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria."

Esse controle de acesso contrasta deliberadamente com o restante do CRUD de configuração de webhooks, que fica aberto a qualquer role autenticada nesta fase: Marcos perguntou em `[09:36]` "O resto do CRUD de configuração de webhook pode ser qualquer role autenticada?", e Sofia respondeu "Por enquanto sim. Mais pra frente a gente pode endurecer" `[09:37]` — ou seja, o endurecimento de acesso via `requireRole('ADMIN')` é, nesta fase, exclusivo do endpoint de replay, por lidar diretamente com reenvio forçado de notificações para URLs externas de clientes.

O mecanismo de controle de role a ser reaproveitado já existe no código: `requireRole` em `src/middlewares/auth.middleware.ts` (linhas 49-61), hoje usado apenas em `src/modules/users/user.routes.ts:15` (`requireRole('ADMIN')` na rota `GET /:id`). O endpoint de replay será o segundo ponto do sistema, e o primeiro fora do módulo de usuários, a usar esse middleware.

## Decisão

O endpoint `POST /admin/webhooks/dead-letter/:id/replay` exige a role `ADMIN`, reaproveitando o middleware `requireRole('ADMIN')` já existente em `src/middlewares/auth.middleware.ts`, encadeado após `authenticate` (mesmo padrão de uso já visto em `user.routes.ts`). Toda execução bem-sucedida do replay registra, para fins de auditoria, o usuário que executou a ação (via log estruturado com o Pino logger existente, contendo `userId`/`requestId`, seguindo a convenção de eventos já usada em `src/middlewares/request-logger.middleware.ts`).

## Alternativas Consideradas

1. **Permitir qualquer role autenticada também no replay, consistente com o restante do CRUD de configuração** — descartada especificamente para esta ação. Sofia justificou o tratamento diferenciado: mexer na fila de reenvio de notificações é uma operação sensível o suficiente para não ser tratada como operação rotineira de operador (`[09:36] Sofia`).
2. **Sem log de auditoria dedicado, confiando apenas nos logs gerais de requisição** — descartada. Sofia exigiu explicitamente que o endpoint "logue quem fez o replay, pra auditoria" (`[09:36]`), indicando a necessidade de um registro identificável da ação administrativa, não apenas do tráfego HTTP genérico.

## Consequências

**Positivas**
- Reduz o risco de reenvio indevido ou abusivo de notificações para clientes externos, restringindo a ação a um grupo de usuários privilegiados.
- Reaproveita infraestrutura já testada e em uso (`requireRole`), sem necessidade de criar um novo mecanismo de autorização.
- Log de auditoria dá rastreabilidade a uma ação administrativa sensível, apoiando investigações futuras.

**Negativas**
- Cria uma assimetria de autorização dentro do próprio módulo de webhooks: CRUD de configuração aberto a qualquer role autenticada, mas replay restrito a `ADMIN` — assimetria que precisa ser documentada com clareza para não confundir desenvolvedores futuros (ver FDD, seção de Contratos Públicos).
- É o primeiro uso de `requireRole` fora do módulo de usuários — qualquer particularidade não coberta pelos testes atuais do middleware (pensado originalmente só para `user.routes.ts`) pode aparecer pela primeira vez aqui.
- Se o único ADMIN disponível estiver indisponível, eventos presos na DLQ não podem ser reprocessados manualmente até que outro ADMIN aja — não há fallback de escalonamento automático nesta decisão.

<!-- TOC -->

- [ADR-006: Reuso dos Padrões Existentes do Projeto (módulo, erros, logger, auth)](#adr-006-reuso-dos-padrões-existentes-do-projeto-módulo-erros-logger-auth)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-006: Reuso dos Padrões Existentes do Projeto (módulo, erros, logger, auth)

## Status

Aceito — `[09:30] Larissa`: "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."

## Contexto

Ao entrar no bloco de estrutura de código, Bruno descreveu o padrão vigente em `[09:27]`: "A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual. Vou propor uma pasta src/modules/webhooks com toda a estrutura." Esse padrão é observável diretamente no código: `src/modules/orders/`, `src/modules/users/`, `src/modules/customers/` e `src/modules/products/` seguem todos a mesma composição (`*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts`), com a instanciação manual (sem framework de DI) feita em `src/app.ts` (`buildControllers`, linhas 26-53) e a montagem de rotas em `src/routes/index.ts` (`buildApiRouter`, linhas 21-31).

Sobre tratamento de erros, Bruno afirmou em `[09:28]`: "a gente já tem um padrão. Tem classe AppError, classes específicas tipo InsufficientStockError, InvalidStatusTransitionError. Todas usam código tipo INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION. Quero seguir igual pra webhook. Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc." Larissa confirmou a convenção de prefixo em `[09:29]`: "Prefixo WEBHOOK_ pra tudo do módulo." Isso é verificável em `src/shared/errors/app-error.ts` (classe base `AppError`) e `src/shared/errors/http-errors.ts`, onde `InvalidStatusTransitionError extends ConflictError` (linha 45) e `InsufficientStockError extends UnprocessableEntityError` (linha 55) já seguem exatamente esse padrão de subclasse com código específico.

Sobre logging e middleware de erro, Bruno afirmou em `[09:29]`: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo. O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada." Isso corresponde exatamente a `src/shared/logger/index.ts` (logger Pino singleton) e `src/middlewares/error.middleware.ts`, cujo tratamento genérico `instanceof AppError` (linha 15) já formata qualquer subclasse nova sem exigir alteração.

Sobre a instância do Prisma no worker, Diego perguntou em `[09:29]`: "o pool de conexão do Prisma já tá lá. O worker abre o mesmo PrismaClient ou um separado?". Bruno respondeu em `[09:30]`: "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node" — mesma lógica de construção de singleton já usada em `src/config/database.ts`, replicada (não compartilhada em memória) no novo processo `src/worker.ts`.

## Decisão

O módulo de webhooks reaproveita, sem modificação, os seguintes padrões e componentes já existentes no código-base:

1. **Estrutura modular** — novo diretório `src/modules/webhooks/` seguindo exatamente o layout `controller/service/repository/routes/schemas` já usado em `src/modules/orders/`, `src/modules/users/`, `src/modules/customers/` e `src/modules/products/`, com instanciação e montagem seguindo o mesmo padrão manual de DI de `src/app.ts` e `src/routes/index.ts`.
2. **Hierarquia de erros** — novas classes de erro do módulo estendem `AppError`/`ConflictError`/`NotFoundError`/`UnprocessableEntityError` de `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`, exatamente como `InvalidStatusTransitionError` e `InsufficientStockError` já fazem hoje. É introduzida uma **convenção nova** de prefixo `WEBHOOK_*` para os códigos de erro do módulo — nova porque, ao contrário do que se poderia supor, o código-base **não tem hoje** uma convenção de prefixo por módulo (os códigos existentes como `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`, `NOT_FOUND`, `CONFLICT` são strings soltas, sem prefixo de domínio). `WEBHOOK_*` é, portanto, um padrão novo aplicado apenas ao módulo de webhooks, não a continuação de algo pré-existente.
3. **Logger** — reuso direto do singleton Pino exportado por `src/shared/logger/index.ts`, tanto na API quanto no processo do worker.
4. **Middleware de erro centralizado** — `src/middlewares/error.middleware.ts` não sofre nenhuma alteração; seu tratamento genérico de `AppError` já cobre as novas classes de erro do módulo de webhooks.
5. **Autenticação/autorização** — reuso de `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts` (o mesmo `requireRole('ADMIN')` já usado em `src/modules/users/user.routes.ts:15`) para o endpoint administrativo de replay da DLQ (ver ADR-008).
6. **PrismaClient** — o worker (`src/worker.ts`) instancia seu próprio `PrismaClient`, seguindo o mesmo padrão de construção de `src/config/database.ts`, apontando para o mesmo `DATABASE_URL`, mas como instância de processo separada.

## Alternativas Consideradas

1. **Introduzir uma biblioteca ou framework externo para fila/erros do módulo de webhooks** (por exemplo, um shape de erro diferente ou uma lib de terceiros para orquestrar o worker) — implicitamente descartada. O tom da decisão de `[09:27]` a `[09:30]` é de consistência deliberada com o restante do projeto ("reuso máximo do que já existe", `[09:30] Larissa`), e a uniformidade observada na codebase (mesma composição de módulo em `orders`, `users`, `customers`, `products`) reforça esse princípio já em prática antes mesmo da feature de webhooks existir.

## Consequências

**Positivas**
- Baixo custo cognitivo para o time: revisores de código já conhecem o padrão de módulo, erros, logger e middleware — não há nova convenção a aprender além do prefixo `WEBHOOK_*`.
- Middleware de erro e logger não precisam de nenhuma alteração de código para suportar o novo módulo.
- Consistência arquitetural reduz risco de bugs por divergência de padrão entre módulos.

**Negativas**
- O prefixo `WEBHOOK_*` é uma convenção nova que **diverge** do padrão atual (sem prefixo) usado pelos demais módulos — é uma inconsistência documentada deliberadamente, não um detalhe a esconder, e pode motivar uma futura padronização retroativa dos códigos de erro existentes (fora do escopo desta feature).
- Como o worker abre sua própria instância de `PrismaClient` em processo separado, qualquer configuração de pool de conexões precisa ser dimensionada considerando dois processos concorrentes acessando o mesmo banco, não apenas um.

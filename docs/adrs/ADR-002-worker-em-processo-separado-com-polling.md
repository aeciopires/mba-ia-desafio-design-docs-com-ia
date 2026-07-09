<!-- TOC -->

- [ADR-002: Worker em Processo Separado com Polling de 2 segundos](#adr-002-worker-em-processo-separado-com-polling-de-2-segundos)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-002: Worker em Processo Separado com Polling de 2 segundos

## Status

Aceito — `[09:10] Larissa`: "Vamos registrar isso como uma decisão. Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos."

## Contexto

Com o padrão outbox definido (ADR-001), é preciso decidir como o worker descobre que há eventos pendentes para enviar, e onde esse worker roda.

Bruno perguntou em `[09:09]` se não daria "pra usar trigger do banco pra ser mais reativo". Diego explicou em `[09:09]` por que essa via não se aplica ao MySQL: "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito. Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo." Marcos confirmou que 2 segundos atende (`[09:10]`: "2 segundos serve, perfeito"), já que o requisito de negócio é "qualquer coisa abaixo de 10 segundos já é 'tempo real'" (`[09:02] Marcos`).

Sobre o isolamento do processo, Diego afirmou em `[09:11]`: "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker." Larissa propôs o formato concreto em `[09:11]`: "Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script 'npm run worker'." Bruno lembrou em `[09:11]` que o worker "vai precisar conectar no mesmo banco e usar o mesmo Prisma client", ao que Diego respondeu em `[09:11]`: "Sim, mesmo banco, mesma stack. Só não pode ser o mesmo processo" — o que implica uma instância própria de `PrismaClient` no processo do worker (`[09:29]-[09:30]`, ver ADR-006).

Uma limitação foi discutida e documentada explicitamente: ordenação de eventos. Larissa perguntou em `[09:12]` se, com mudanças rápidas de status em sequência, o cliente recebe na ordem certa. Diego respondeu em `[09:12]`: "Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem. Se a gente escala pra múltiplos workers em paralelo no futuro, perde a garantia. Por enquanto, single-worker e ordering implícita por order_id." Bruno perguntou sobre escala futura (`[09:13]`), e Diego respondeu: "Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora" `[09:13]`. Larissa fechou o ponto em `[09:13]`: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker." Marcos confirmou que isso é aceitável, pois "os clientes nunca pediram garantia de ordering global, eles só querem saber se cada pedido deles mudou" `[09:14]`.

## Decisão

Implementar o worker como um processo Node.js separado, com um novo entry point `src/worker.ts` (espelhando o padrão já usado em `src/server.ts`) e um script `npm run worker`. O worker mantém sua própria instância de `PrismaClient`, conectada ao mesmo `DATABASE_URL` do restante da aplicação, mas rodando em processo distinto da API.

O worker opera em loop de polling, verificando a cada 2 segundos se há eventos pendentes na `webhook_outbox` (ordenados por `created_at`), processando-os em lote pequeno. A garantia de ordenação de entrega é explicitamente limitada: só é garantida por `order_id`, e apenas enquanto houver um único worker em execução. Escalar para múltiplos workers em paralelo (com particionamento por `order_id` ou lock pessimista) fica fora do escopo desta decisão e é tratado como problema futuro.

## Alternativas Consideradas

1. **Reação via trigger de banco + notificação de processo externo** — descartada. MySQL não oferece um mecanismo nativo de pub/sub (como `LISTEN`/`NOTIFY` do PostgreSQL); triggers só executam SQL e não conseguem notificar um processo Node.js diretamente. Workarounds cogitados (escrever em arquivo, chamar um endpoint a partir do banco) foram classificados como "esquisito" por Diego (`[09:09]`) e descartados por complexidade desnecessária frente ao requisito de latência já satisfeito por polling de 2s.
2. **Worker rodando dentro do próprio processo da API** — implicitamente descartada: quebraria o isolamento operacional, pois um restart da API (deploy, crash) derrubaria o worker junto (`[09:11] Diego`).
3. **Múltiplos workers em paralelo desde o início** — descartada para esta fase; adiciona necessidade de particionamento por `order_id` ou locking pessimista para preservar ordenação, tratado como problema futuro (`[09:13] Diego`).

## Consequências

**Positivas**
- Isola a carga e a disponibilidade de disparo de webhooks do processo da API — um restart de deploy não interrompe permanentemente o processamento (o worker é reiniciado de forma independente).
- Modelo simples de operar e depurar (loop de polling, sem infraestrutura de mensageria adicional).
- Latência de pior caso previsível e documentada (2s de polling), dentro do requisito de negócio de <10s.

**Negativas**
- Garantia de ordenação de entrega é apenas por `order_id` e apenas com um único worker ativo — limitação conhecida e documentada, não resolvida por esta decisão.
- Escalar horizontalmente (múltiplos workers) exigirá trabalho adicional de particionamento/locking não desenhado nesta fase (`[09:13] Diego`) — ver questão em aberto no RFC.
- Um novo processo (`src/worker.ts`) passa a fazer parte do deploy e da observabilidade da aplicação, aumentando a superfície operacional.

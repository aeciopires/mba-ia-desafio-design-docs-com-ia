<!-- TOC -->

- [ADR-003: Retry com Backoff Exponencial e Dead Letter Queue](#adr-003-retry-com-backoff-exponencial-e-dead-letter-queue)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-003: Retry com Backoff Exponencial e Dead Letter Queue

## Status

Aceito — `[09:17] Larissa`: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."

## Contexto

Larissa levantou o problema em `[09:14]`: "Vamos pra retry. Se o cliente tá offline, o que a gente faz?". Diego propôs a estratégia em `[09:15]`: "Backoff exponencial. Tenta de novo depois de algum tempo, vai aumentando o intervalo, e depois de um teto de tentativas considera falha permanente e move pra DLQ."

Sobre o número de tentativas, houve debate explícito. Diego sugeriu 5 em `[09:15]`: "Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu. Cinco já dá pra cobrir uma janela de até 12 ou 24 horas." Bruno contrapôs com 3 tentativas em `[09:16]` por ser "mais agressivo". Diego rebateu com um precedente concreto: "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada" `[09:16]`. Larissa fechou com 5 tentativas (`[09:16]`).

A progressão de backoff foi definida por Diego em `[09:17]`: "Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa." Marcos considerou aceitável: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" `[09:17]`.

Sobre onde armazenar falhas permanentes, Larissa perguntou em `[09:17]`: "Faz numa tabela separada ou marca como 'failed' na própria outbox?". Diego respondeu em `[09:18]`: "Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento." Bruno concordou e perguntou sobre reprocessamento (`[09:18]`), ao que Diego respondeu: "Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente" `[09:18]` (ver ADR-008 para o desenho completo desse endpoint).

## Decisão

Implementar retry com backoff exponencial: até 5 tentativas de entrega por evento, com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas entre tentativas sucessivas (~15 horas de janela total entre a primeira falha e a última tentativa). Esgotadas as 5 tentativas, o evento é movido para uma tabela separada `webhook_dead_letter` (contendo payload, motivo da falha e timestamp), em vez de apenas marcar um status de "falhou" na própria `webhook_outbox`.

## Alternativas Consideradas

1. **3 tentativas** — descartada. Diego argumentou que 3 tentativas se esgotariam em apenas 30 minutos, insuficiente para cobrir indisponibilidades reais e prolongadas de clientes (citando um precedente de indisponibilidade de 2 horas em manutenção planejada) `[09:16]`.
2. **Retry indefinido com backoff crescente, sem teto de tentativas** — descartada. Diego apontou que deixaria eventos "pendurados pra sempre" caso o cliente nunca mais volte a responder `[09:15]`, sem um estado terminal claro.
3. **Marcar o evento como "failed" na própria tabela `webhook_outbox`, sem tabela separada** — implicitamente descartada em favor de uma tabela `webhook_dead_letter` dedicada, para manter a leitura da outbox "limpa" (isto é, restrita a eventos ainda ativos) e fornecer uma estrutura própria para investigação/reprocessamento `[09:17]-[09:18] Diego`.

## Consequências

**Positivas**
- Janela de retry de ~15 horas cobre indisponibilidades reais e prolongadas já observadas em produção (precedente de outage de 2h citado por Diego), reduzindo falsos negativos de entrega.
- Separar a DLQ em tabela própria mantém consultas à outbox operacional rápidas e simples, e fornece um registro estruturado (payload + motivo + timestamp) para investigação.
- Existência de um estado terminal (DLQ) evita acúmulo indefinido de eventos "pendurados".

**Negativas**
- Um cliente com indisponibilidade maior que ~15 horas terá seus eventos movidos para a DLQ e não será notificado automaticamente disso — não há alerta proativo nesta fase (alertas por e-mail foram explicitamente adiados, `[09:37]-[09:38]`), exigindo intervenção manual via replay (ADR-008).
- Introduz a necessidade de uma tabela e um fluxo operacional adicionais (`webhook_dead_letter`) que precisam ser monitorados (ver seção de Observabilidade no FDD).
- Sem alerta automático, um volume grande de falhas pode passar despercebido até alguém consultar a DLQ manualmente.

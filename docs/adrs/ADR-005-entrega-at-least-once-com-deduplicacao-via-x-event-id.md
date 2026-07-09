<!-- TOC -->

- [ADR-005: Entrega At-Least-Once com Deduplicação via X-Event-Id](#adr-005-entrega-at-least-once-com-deduplicação-via-x-event-id)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-005: Entrega At-Least-Once com Deduplicação via X-Event-Id

## Status

Aceito — `[09:26] Larissa`: "Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

## Contexto

Diego colocou a garantia de entrega em discussão em `[09:24]`: "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado." Bruno perguntou como o cliente diferenciaria eventos duplicados (`[09:25]`), e Diego respondeu: "A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele" `[09:25]`.

Sofia levantou a objeção central em `[09:25]`: "Isso joga responsabilidade pro cliente." Diego reconheceu o trade-off explicitamente e justificou a escolha: "Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim. Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" `[09:25]`. Marcos se comprometeu a mitigar o impacto por meio de documentação: "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema" `[09:26]`.

## Decisão

Garantir semântica de entrega **at-least-once** (pelo menos uma vez), não exactly-once. Cada evento carrega um `X-Event-Id` (UUID) gerado no momento da inserção na outbox, único por evento, permitindo que o cliente implemente deduplicação do lado dele. A responsabilidade de deduplicar entregas duplicadas é explicitamente do cliente integrador, e este comportamento deve ser documentado com destaque no material de integração (portal de desenvolvedor).

## Alternativas Consideradas

1. **Garantia exactly-once** — descartada. Diego argumentou que exigiria coordenação transacional entre os dois lados (sistema emissor e sistema receptor), adicionando complexidade desproporcional ao ganho, frente a um padrão at-least-once + `X-Event-Id` já validado por players de mercado como Stripe e GitHub `[09:25] Diego`.

## Consequências

**Positivas**
- Implementação significativamente mais simples do lado do worker: reenviar em caso de dúvida (timeout, resposta ambígua) é sempre seguro, sem necessidade de coordenação distribuída.
- Segue um padrão de mercado already conhecido por integradores B2B experientes, reduzindo a curva de aprendizado do lado do cliente.
- `X-Event-Id` gerado na inserção da outbox (ver ADR-007) garante unicidade estável do identificador ao longo de todas as tentativas de reenvio do mesmo evento.

**Negativas**
- Transfere explicitamente parte do ônus de correção para o cliente integrador (`[09:25] Sofia`) — um cliente que não implementa deduplicação corretamente pode processar o mesmo evento de negócio mais de uma vez.
- Exige documentação clara e visível no portal de desenvolvedor (compromisso assumido por Marcos em `[09:26]`) como mitigação, o que é uma dependência de processo, não apenas técnica.
- Não há, nesta decisão, nenhum mecanismo do lado do servidor para detectar ou alertar sobre clientes que estão claramente falhando em deduplicar.

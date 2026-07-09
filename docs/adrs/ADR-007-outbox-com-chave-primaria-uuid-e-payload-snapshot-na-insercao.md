<!-- TOC -->

- [ADR-007: Outbox com Chave Primária UUID e Payload Congelado na Inserção](#adr-007-outbox-com-chave-primária-uuid-e-payload-congelado-na-inserção)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-007: Outbox com Chave Primária UUID e Payload Congelado na Inserção

## Status

Aceito — `[09:51] Larissa`: "UUID, segue o padrão do resto do projeto. Tudo é uuid." / `[09:52] Bruno`: "Beleza, snapshot. Decidido."

## Contexto

Ao final da reunião, Diego levantou uma dúvida de modelagem em `[09:51]`: "Quando a gente for modelar a outbox, prefere id auto incremental ou UUID?". Larissa respondeu: "UUID, segue o padrão do resto do projeto. Tudo é uuid" `[09:51]`. Isso é consistente com o schema Prisma atual (`prisma/schema.prisma`), onde todos os modelos de negócio (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`) usam `id String @id @default(uuid()) @db.Char(36)`.

Em seguida, Bruno levantou uma segunda dúvida de modelagem em `[09:51]`: "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?". Larissa respondeu em `[09:52]`: "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito." Diego concordou ("Concordo, snapshot na inserção", `[09:52]`) e Bruno fechou o ponto (`[09:52]`: "Beleza, snapshot. Decidido.").

Essa decisão também se conecta ao ponto discutido em `[09:33]-[09:34]`, sobre o filtro de eventos por status ser aplicado no momento da inserção na outbox (e não no momento do envio): Bruno explicou em `[09:34]`: "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela" — reforçando que a linha da outbox, quando criada, já reflete todas as decisões de "o que enviar e com qual conteúdo" tomadas no momento exato da mudança de status.

## Decisão

A chave primária da tabela `webhook_outbox` (e das demais tabelas novas do módulo, como `webhook_dead_letter`) é um UUID, consistente com a convenção já usada em todos os modelos existentes do `prisma/schema.prisma`. O payload JSON do evento é renderizado e "congelado" (snapshot) no momento da inserção na outbox — não é re-renderizado a partir do estado atual do pedido no momento do envio pelo worker.

## Alternativas Consideradas

1. **ID auto-incremental para a outbox** — descartada, em favor de manter consistência com a convenção de UUID já usada em 100% dos modelos existentes do projeto (`[09:51] Larissa`).
2. **Armazenar apenas `order_id` na outbox e renderizar o payload dinamicamente no momento do envio** — descartada. Larissa argumentou que isso criaria "caso esquisito" se o pedido mudasse de estado entre a inserção do evento e o envio efetivo pelo worker — o payload entregue ao cliente não corresponderia mais ao estado do pedido no momento exato da mudança de status que originou o evento (`[09:52] Larissa`).

## Consequências

**Positivas**
- Consistência de convenção de identificadores com o restante do projeto, sem necessidade de justificar uma exceção.
- O payload entregue ao cliente é uma fotografia fiel do estado do pedido exatamente no momento da mudança de status que gerou o evento — imune a mudanças posteriores no pedido, mesmo que o envio HTTP só ocorra minutos ou horas depois (por retry).
- Simplifica o worker: ele apenas lê e envia o payload já pronto, sem precisar reconsultar o estado atual do pedido nem lidar com inconsistências de re-renderização tardia.

**Negativas**
- Um bug na lógica de renderização do payload no momento da inserção não pode ser corrigido "para trás": eventos já inseridos na outbox (ou já reenviados via replay da DLQ) continuam carregando o payload no formato antigo, mesmo após uma correção de código.
- Qualquer enriquecimento futuro do payload (novos campos) só se aplica a eventos inseridos após o deploy da mudança — eventos antigos na outbox/DLQ mantêm o formato anterior.

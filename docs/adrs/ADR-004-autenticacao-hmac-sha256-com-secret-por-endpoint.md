<!-- TOC -->

- [ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint](#adr-004-autenticação-hmac-sha256-com-secret-por-endpoint)
  - [Status](#status)
  - [Contexto](#contexto)
  - [Decisão](#decisão)
  - [Alternativas Consideradas](#alternativas-consideradas)
  - [Consequências](#consequências)

<!-- TOC -->

# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint

## Status

Aceito — `[09:22] Sofia`: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."

## Contexto

Ao iniciar a parte de segurança, Sofia colocou o problema em `[09:19]`: "a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio." A solução proposta foi assinatura HMAC: "A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele" `[09:20]`. Questionada sobre o algoritmo (`[09:20] Bruno`), Sofia respondeu: "SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso" `[09:20]`.

Sobre o escopo da secret, Sofia foi enfática em `[09:21]`: "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo." Bruno confirmou o desenho da tabela de configuração: "a tabela de configuração de webhook armazena url + secret + customer_id + estado ativo?" `[09:21]`, confirmado por Sofia.

A rotação de secret foi definida também por Sofia em `[09:21]`: "a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre." Diego trouxe o motivador de negócio para essa exigência em `[09:22]`: "A gente já teve cliente que vazou secret em log de aplicação dele uma vez" — um incidente real que justifica tanto o isolamento por endpoint quanto a capacidade de rotação sem downtime.

## Decisão

Assinar o corpo de cada requisição de webhook com HMAC-SHA256, usando uma secret exclusiva por endpoint cadastrado (não uma secret global da plataforma). A assinatura é enviada no header `X-Signature`. Cada endpoint de webhook suporta rotação de secret via API; durante um período de 24 horas após a rotação, tanto a secret antiga quanto a nova são aceitas como válidas, para permitir a migração gradual do lado do cliente.

## Alternativas Consideradas

1. **Não assinar as requisições (confiar apenas em TLS/HTTPS)** — implicitamente descartada: Sofia tratou a assinatura como requisito não negociável desde o início da discussão de segurança, pois TLS garante confidencialidade em trânsito mas não prova a origem do payload nem sua integridade fim a fim para quem recebe (`[09:19]-[09:20] Sofia`).
2. **Secret única e global para toda a plataforma** — descartada explicitamente. Sofia apontou o risco de blast radius: "se vaza uma, vaza tudo" `[09:21]`, motivando o isolamento por endpoint.

## Consequências

**Positivas**
- Limita o impacto de um vazamento de secret a um único endpoint/cliente, em vez de comprometer todos os clientes integrados.
- Rotação com grace period de 24h permite ao cliente migrar sem downtime na validação de assinaturas, resolvendo diretamente o cenário do incidente citado por Diego (`[09:22]`).
- HMAC-SHA256 é um padrão amplamente suportado, reduzindo o esforço de implementação do lado do cliente.

**Negativas**
- Exige uma tabela de configuração própria (`url + secret + customer_id + estado ativo`, `[09:21] Bruno`) e um fluxo de geração/rotação/expiração de secrets — nova responsabilidade operacional que não existia no projeto.
- Durante a janela de 24h de rotação, o sistema precisa validar contra duas secrets simultaneamente, adicionando complexidade à lógica de verificação (mesmo que a verificação em si ocorra do lado do cliente, o sistema precisa manter e expor a secret anterior por esse período).
- Clientes precisam implementar corretamente a verificação HMAC do lado deles; erros de implementação no cliente não são de responsabilidade do sistema, mas geram suporte adicional.

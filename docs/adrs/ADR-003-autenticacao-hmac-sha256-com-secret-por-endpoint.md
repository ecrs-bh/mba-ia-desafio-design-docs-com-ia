# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

- **Status:** Aceito
- **Data:** 2026-10-05
- **Autores:** Sofia (Engenheira de Segurança), Bruno (Engenheiro Pleno), Larissa (Tech Lead)

## Contexto

A plataforma enviará requisições HTTP POST com payloads de dados de pedidos para URLs externas fornecidas pelos clientes B2B. É imprescindível garantir que:
1. O cliente recebedor possa autenticar que a requisição partiu legitimamente do nosso sistema.
2. O payload não tenha sido adulterado em trânsito (integridade).
3. A segurança de um cliente não fique comprometida caso a chave de outro vaze.

## Decisão

Implementar autenticação de saída utilizando assinaturas **HMAC com algoritmo SHA-256 (HMAC-SHA256)** com **secret única por endpoint de webhook**.

1. **Assinatura do Payload:**
   * O worker calculará a assinatura `HMAC-SHA256(payload_body, endpoint_secret)` sobre o corpo bruto do JSON.
   * A assinatura no formato hexadecimal será enviada no header HTTP `X-Signature`.

2. **Secret por Endpoint e Rotação:**
   * Cada cadastro de webhook (`WebhookEndpoint`) possuirá sua própria `secret` cryptograficamente segura gerada pelo nosso sistema.
   * Suporte a rotação de secret via API (`POST /webhooks/:id/rotate-secret`).
   * A rotação utilizará um período de carência (*grace period*) de **24 horas**, durante o qual a secret antiga (`previousSecret`) e a nova secret (`secret`) permanecem ambas válidas para cálculo de assinatura, permitindo que o cliente migre suas chaves sem downtime.

3. **Requisitos Adicionais de Segurança:**
   * **TLS Obrigatório:** A URL do destino do webhook DEVE utilizar o esquema `https://`. URLs `http://` serão rejeitadas no schema de validação Zod.
   * **Teto de Tamanho:** O payload máximo permitido é de **64KB**. Requisições que excedam este limite serão rejeitadas com erro sem envio.

## Alternativas Consideradas

* **Secret Global / Plataforma Única:** Rejeitada devido ao alto risco de segurança — o vazamento de uma única chave em logs de um cliente comprometeria a autenticidade dos webhooks de todos os outros clientes B2B.
* **Rotação Instantânea de Chave:** Rejeitada pois quebraria o recebimento de notificações do cliente no intervalo entre a solicitação de rotação e a atualização nos servidores dele.
* **Basic Auth / Bearer Token Estático:** Rejeitada por não garantir a integridade do corpo da mensagem (*tamper-proofing*).

## Consequências

### Positivas
* **Integridade e Autenticidade:** O receptor valida que o payload veio do nosso sistema e que não sofreu modificação.
* **Isolamento de Segurança:** O comprometimento da secret de um cliente não impacta os demais.
* **Rotação sem Indisponibilidade:** O grace period de 24 horas garante transição suave de chaves.

### Negativas
* **Gerenciamento de Estado de Chaves:** O modelo da tabela `webhook_endpoints` precisa armazenar os campos `secret`, `previousSecret` e `secretExpiresAt`.
* **Complexidade no Cliente:** O cliente precisa implementar a checagem da assinatura HMAC-SHA256 no receptor do webhook.

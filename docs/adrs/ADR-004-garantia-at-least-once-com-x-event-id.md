# ADR-004: Garantia At-Least-Once com X-Event-Id

- **Status:** Aceito
- **Data:** 2026-10-05
- **Autores:** Diego (Engenheiro Sênior), Larissa (Tech Lead), Marcos (Product Manager)

## Contexto

Em sistemas distribuídos que realizam notificações via HTTP com políticas de retry, falhas de rede no momento do envio da resposta do cliente podem fazer com que o remetente considere que o envio falhou, quando na verdade o cliente já havia processado a requisição.

A garantia de entrega bidirecional perfeita (*exactly-once*) em chamadas HTTP remotas exige coordenação distribuída síncrona (como Two-Phase Commit), o que gera alta complexidade e perda de disponibilidade.

## Decisão

Adotar a semântica de entrega **At-Least-Once (pelo menos uma vez)**, repassando a responsabilidade de desduplicação idempotente para o receptor do cliente B2B através de um identificador único de evento.

1. **Identificador Único (`X-Event-Id`):**
   * No momento em que a mudança de status do pedido é registrada, é gerado um UUIDv4 único para o evento de webhook.
   * Esse identificador é persistido no campo `event_id` da `webhook_outbox` e transmitido tanto no header HTTP `X-Event-Id` quanto no corpo da mensagem JSON (`event_id`).

2. **Headers HTTP Padronizados:**
   * `X-Event-Id`: Identificador UUIDv4 único da notificação para desduplicação.
   * `X-Webhook-Id`: Identificador do cadastro do webhook receptor.
   * `X-Signature`: Assinatura HMAC-SHA256 do payload.
   * `X-Timestamp`: Carimbo de data/hora ISO-8601 de envio da mensagem (permite ao cliente prevenir ataques de *replay*).
   * `Content-Type`: `application/json`.

3. **Idempotência no Cliente:**
   * O cliente B2B deve registrar os `X-Event-Id` processados. Se o mesmo `X-Event-Id` for recebido mais de uma vez devido a um retry do worker, o cliente deve ignorar a execução repetida e retornar status HTTP `200 OK`.

## Alternativas Consideradas

* **Garantia Exactly-Once Síncrona:** Rejeitada por requerer coordenação distribuída bidirecional complexa e impraticável na Web sobre protocolo HTTP público.
* **Desduplicação apenas por `order_id`:** Rejeitada pois um mesmo pedido sofre múltiplas transições de status ao longo do tempo (ex: `PENDING` -> `PAID` -> `PROCESSING` -> `SHIPPED` -> `DELIVERED`). Usar apenas o `order_id` impediria o recebimento legítimo de transições subsequentes.

## Consequências

### Positivas
* **Alinhamento com Padrões de Mercado:** Padrão consagrado adotado por provedores como Stripe e GitHub.
* **Simplicidade e Tolerância a Falhas:** Permite retentativas sem risco de corrupção de estado na plataforma remetente.

### Negativas
* **Exigência ao Cliente B2B:** Os clientes devem implementar lógica de cache/tabela de deduplicação idempotente baseada no `X-Event-Id`.

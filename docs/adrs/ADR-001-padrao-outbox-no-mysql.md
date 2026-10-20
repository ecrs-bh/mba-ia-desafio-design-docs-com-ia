# ADR-001: Padrão Outbox no MySQL

- **Status:** Aceito
- **Data:** 2026-10-05
- **Autores:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno)

## Contexto

Os clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) necessitam de notificações em tempo real quando o status dos seus pedidos é alterado. Atualmente, o método [`changeStatus()`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts#L126-L179) no `OrderService` executa uma transação SQL atômica via Prisma (`this.prisma.$transaction`) que atualiza o registro na tabela `orders`, grava o histórico na `order_status_history` e realiza o débito/estorno de estoque na tabela `products`.

Foram discutidas duas abordagens para notificar os sistemas externos:
1. **Chamada HTTP síncrona dentro da transação do `OrderService`:** Descartada por introduzir alto acoplamento, latência imprevisível e risco de dar rollback na alteração do pedido caso a API do cliente estivesse fora do ar ou lenta.
2. **Uso de middleware de mensageria externo (ex: Redis Streams / Cluster ou RabbitMQ):** Descartado por representar *overengineering* e introduzir complexidade operacional/infraestrutura desnecessária para o tamanho atual da equipe e volumetria.

## Decisão

Adotar o **Padrão Transactional Outbox (Outbox Pattern)** utilizando o próprio banco de dados MySQL da aplicação.

Dentro da mesma transação SQL (`tx`) que atualiza o status da ordem e insere na `order_status_history` em `OrderService.changeStatus()`, será inserido um registro na nova tabela `webhook_outbox`. A publicação será intermediada pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`.

O payload gravado no outbox será um *snapshot* do estado do pedido gerado no momento da transação SQL, contendo os campos essenciais (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`).

## Alternativas Consideradas

* **Chamada HTTP Síncrona:** Rejeitada devido ao risco de degradar a performance da transação principal de pedidos e falhar a alteração de status por indisponibilidade do cliente B2B.
* **Fila / Event Streaming com Redis Cluster:** Rejeitada por introduzir novos componentes de infraestrutura para manter e monitorar, quando a transacionalidade do MySQL satisfaz completamente o requisito.

## Consequências

### Positivas
* **Garantia de Consistência Atômica:** Se a transação SQL do pedido for commitada, o evento de webhook estará garantidamente registrado. Se a transação der rollback, o evento é cancelado junto.
* **Desacoplamento:** A API de pedidos responde imediatamente ao cliente sem aguardar a latência das redes de terceiros.
* **Zero Infraestrutura Adicional:** Utilização do cluster MySQL já existente e gerenciado pela aplicação.

### Negativas
* **Crescimento da Tabela:** A tabela `webhook_outbox` acumulará registros ao longo do tempo, exigindo uma política de expurgo ou arquivamento assíncrono para registros entregues com mais de 30 dias (fora do escopo desta fase).
* **Sobrecarga no MySQL:** Adiciona uma inserção a mais na transação de mudança de status de pedidos.

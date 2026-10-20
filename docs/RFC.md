# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

* **Autor:** Bruno (Engenheiro Pleno) & Diego (Engenheiro Sênior)
* **Status:** Em Revisão (Proposed)
* **Data:** 2026-10-05
* **Revisores:** Larissa (Tech Lead), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## 1. Resumo Executivo (TL;DR)

Esta RFC propõe a arquitetura para o **Sistema de Webhooks de Notificação de Pedidos** (Outbound Webhooks), permitindo que clientes B2B (ex: Atlas Comercial, MaxDistribuição e Nova Cargo) recebam notificações HTTP POST em tempo real (latência < 10s) sempre que o status de um pedido for alterado no Order Management System (OMS).

A solução adota o **Padrão Transactional Outbox** gravando eventos na tabela `webhook_outbox` do MySQL na mesma transação atômica que altera o status da ordem. Um worker Node.js isolado processa os eventos via polling a cada 2 segundos, garante entregas com a semântica *at-least-once* e aplica retentativas com *backoff* exponencial de 5 tentativas (~15 horas) antes de direcionar falhas permanentes para a **Dead Letter Queue (DLQ)** (`webhook_dead_letter`). A autenticação das notificações utiliza HMAC-SHA256 com secret única por endpoint e suporte a rotação com *grace period* de 24 horas.

---

## 2. Contexto e Problema

Atualmente, parceiros B2B realizam requisições periódicas no endpoint `GET /orders` (*polling*) para monitorar a transição de status de seus pedidos. Essa prática acarreta:
1. **Ineficiência e Custos:** Sobrecarga desnecessária na infraestrutura do OMS devido ao grande volume de consultas sem alteração de estado.
2. **Latência de Integração:** O cliente só toma conhecimento da mudança no próximo ciclo de consulta.
3. **Risco de Churn Comercial:** Clientes estratégicos como a Atlas Comercial formalizaram a necessidade de notificações em tempo real sob risco de migração para concorrentes ao fim do trimestre.

---

## 3. Proposta Técnica (Visão Geral de Arquitetura)

### 3.1 Arquitetura da Solução

A solução é composta por quatro pilares arquiteturais integrados ao OMS:

```
+------------------+         +------------------+         +----------------------+
|                  |         |  MySQL Database  |         |                      |
|   Order Service  |-------> |                  | <------ |  Webhook Worker Process
| (changeStatus)   | (tx)    |  webhook_outbox  | (poll)  |  (src/worker.ts)     |
+------------------+         +------------------+         +----------+-----------+
                                                                     |
                                                           HTTP POST | (HMAC-SHA256)
                                                                     v
                                                          +----------------------+
                                                          | External B2B Client  |
                                                          |  Webhook Endpoint    |
                                                          +----------------------+
```

1. **Captura Atômica de Eventos (Transactional Outbox):**
   * No método `changeStatus()` do `OrderService`, a função `publishWebhookEvent()` é executada dentro do callback `$transaction` do Prisma.
   * Se a alteração do pedido for efetivada, o evento é gravado na tabela `webhook_outbox` com status `PENDING` e snapshot dos dados.

2. **Processamento Assíncrono Desacoplado (Worker Process):**
   * Processo Node.js independente (`src/worker.ts`) em loop de polling de 2 segundos.
   * Seleciona lotes de eventos pendentes ordenados por `createdAt ASC` (single-worker garante ordem sequencial por pedido).

3. **Resiliência e Recuperação (Retry & DLQ):**
   * Em caso de falha de rede ou erro HTTP (status != 2xx), aplica-se retry com backoff exponencial: **1m, 5m, 30m, 2h e 12h**.
   * Se a 5ª retentativa falhar, o evento é movido para a tabela `webhook_dead_letter`.
   * Endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay` (exige role `ADMIN`) permite reprocessar eventos da DLQ.

4. **Segurança e Integridade (HMAC-SHA256 & TLS):**
   * Assinatura do payload no header `X-Signature` calculada via HMAC-SHA256 usando chave secreta por endpoint.
   * Suporte a rotação de secret mantendo a chave antiga válida por 24 horas (`grace period`).
   * URLs obrigatoriamente HTTPS e limite rígido de payload de 64KB.

---

## 4. Alternativas Consideradas

Durante a fase de concepção técnica, foram analisadas e descartadas duas alternativas principais:

### Alternativa 1: Disparo HTTP Síncrono no `OrderService`
* **Descrição:** Disparar a requisição HTTP POST para o endpoint do cliente diretamente dentro do método `OrderService.changeStatus()`.
* **Motivo do Descarte / Trade-off:** Risco inaceitável de indisponibilidade e alta latência. Se a API do cliente B2B estivesse lenta ou fora do ar, a transação de alteração de status do pedido no OMS travaria ou sofreria rollback, impedindo operações internas.

### Alternativa 2: Event Streaming / Mensageria com Redis Cluster ou RabbitMQ
* **Descrição:** Publicar eventos de mudança de status em um broker de mensageria externo (ex: Redis Streams ou RabbitMQ) para consumo por um worker.
* **Motivo do Descarte / Trade-off:** *Overengineering* para o momento da empresa. Subir e gerenciar um cluster Redis ou RabbitMQ adicionaria custos operacionais e complexidade de infraestrutura para uma equipe enxuta, enquanto o padrão Outbox no MySQL garante transacionalidade nativa e custo zero de infraestrutura adicional.

---

## 5. Questões em Aberto e Pontos Adiados

Os seguintes pontos foram discutidos na reunião técnica e definidos como **fora do escopo desta RFC**, devendo ser reavaliados em fases futuras:

1. **Rate Limiting de Saída para Endpoints de Clientes:**
   * **Situação:** Se um cliente tiver dezenas de pedidos alterando status simultaneamente, o worker enviará requisições em rajada. Decidiu-se **não implementar rate limiting de saída nesta fase** e monitorar o comportamento em produção antes de introduzir throttling.
2. **Notificações por E-mail para Endpoints com Falhas Consecutivas:**
   * **Situação:** Enviar alerta por e-mail para o administrador do cliente B2B quando seu webhook acumular falhas. Decidiu-se que **alertas por e-mail estão fora de escopo**, sendo a observabilidade restrita aos endpoints da API e logs do sistema.

---

## 6. Impacto, Cronograma e Riscos

### Cronograma Estimado
* **Duração Total:** 3 Sprints (incluindo 2 dias úteis dedicados à revisão de segurança pela equipe de AppSec).
* **Distribuição:**
  * Sprint 1: Modelagem das tabelas MySQL (Prisma), funções do Outbox e estrutura da DLQ.
  * Sprint 2: Desenvolvimento do Worker (`src/worker.ts`), engine de retries, assinaturas HMAC-SHA256 e rotação de secret.
  * Sprint 3: Endpoints CRUD de webhooks, consulta de deliveries, integração no `OrderService.changeStatus()`, testes end-to-end e revisão de segurança.

### Riscos Identificados e Mitigações
* **Crescimento da Tabela Outbox:** Aumento de volume na tabela `webhook_outbox`. *Mitigação:* Planejar job assíncrono para expurgo/arquivamento de eventos entregues há mais de 30 dias.
* **Race Conditions em Múltiplos Workers:** *Mitigação:* Executar como single-worker na Fase 1, garantindo ordenação natural por `order_id` e `createdAt`.

---

## 7. Decisões Relacionadas (ADRs)

As decisões arquiteturais detalhadas que fundamentam esta proposta estão registradas nos seguintes ADRs:

* [`ADR-001: Padrão Outbox no MySQL`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-001-padrao-outbox-no-mysql.md)
* [`ADR-002: Política de Retry com Backoff Exponencial e DLQ`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md)
* [`ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
* [`ADR-004: Garantia At-Least-Once com X-Event-Id`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)
* [`ADR-005: Worker em Processo Separado em Polling`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md)
* [`ADR-006: Reuso dos Padrões Existentes do Projeto`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

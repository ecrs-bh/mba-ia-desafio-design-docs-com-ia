# Tracker de Rastreabilidade

O Tracker de Rastreabilidade fornece uma referência cruzada que mapeia cada requisito, decisão arquitetural, restrição, trade-off e contrato público registrado nos documentos de design (`PRD.md`, `RFC.md`, `FDD.md` e ADRs) à sua origem exata na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte da aplicação (`src/`, `prisma/`).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Notificação em tempo real de mudança de status de pedidos para clientes B2B | `TRANSCRICAO` | `[09:00] Marcos` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência de notificação inferior a 10 segundos | `TRANSCRICAO` | `[09:02] Marcos` |
| PRD-SCOP-01 | `docs/PRD.md` | Restrição | Escopo exclusivo para outbound webhooks (sistema dispara para clientes) | `TRANSCRICAO` | `[09:03] Sofia` |
| ADR-001-01 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Trade-off | Inviabilidade de chamada HTTP síncrona no OrderService devido à latência e acoplamento | `TRANSCRICAO` | `[09:04] Bruno` |
| ADR-001-02 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Adoção do Padrão Transactional Outbox em MySQL | `TRANSCRICAO` | `[09:06] Diego` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Descarte do uso de Redis Cluster / Streams por overengineering | `TRANSCRICAO` | `[09:07] Diego` |
| FDD-DB-01 | `docs/FDD.md` | Especificação Técnica | Tabela Outbox com índices em status e created_at e expurgo após 30 dias | `TRANSCRICAO` | `[09:08] Diego` |
| ADR-005-01 | `docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md` | Decisão Arquitetural | Worker assíncrono em polling de 2 segundos e descarte de triggers de banco | `TRANSCRICAO` | `[09:09] Diego` |
| ADR-005-02 | `docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md` | Decisão Arquitetural | Worker executado como processo Node.js separado (entrypoint `src/worker.ts`) | `TRANSCRICAO` | `[09:11] Diego` |
| ADR-005-03 | `docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md` | Restrição | Single-worker para garantir ordenação por `created_at` e `order_id` | `TRANSCRICAO` | `[09:12] Diego` |
| ADR-002-01 | `docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md` | Decisão Arquitetural | Política de retry com Backoff Exponencial limitados a 5 tentativas | `TRANSCRICAO` | `[09:15] Diego` |
| ADR-002-02 | `docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md` | Especificação Técnica | Intervals do Backoff Exponencial: 1m, 5m, 30m, 2h, 12h (janela de ~15h) | `TRANSCRICAO` | `[09:16] Diego` |
| ADR-002-03 | `docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md` | Decisão Arquitetural | Tabela `webhook_dead_letter` (DLQ) separada e endpoint de replay manual | `TRANSCRICAO` | `[09:18] Diego` |
| ADR-003-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão Arquitetural | Autenticação via assinatura HMAC-SHA256 no header `X-Signature` | `TRANSCRICAO` | `[09:20] Sofia` |
| ADR-003-02 | `docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão Arquitetural | Secret única por endpoint com suporte a rotação com grace period de 24h | `TRANSCRICAO` | `[09:21] Sofia` |
| ADR-003-03 | `docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Requisito Não Funcional | Obrigatoriedade de protocolo HTTPS para URLs de webhooks | `TRANSCRICAO` | `[09:23] Sofia` |
| FDD-RES-01 | `docs/FDD.md` | Restrição | Limite máximo do tamanho de payload em 64KB | `TRANSCRICAO` | `[09:24] Diego` |
| ADR-004-01 | `docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Garantia de entrega *at-least-once* com `X-Event-Id` (UUID) para deduplicação | `TRANSCRICAO` | `[09:25] Diego` |
| ADR-006-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Decisão Arquitetural | Novo módulo na pasta `src/modules/webhooks` espelhando os módulos atuais | `TRANSCRICAO` | `[09:28] Bruno` |
| FDD-ERR-01 | `docs/FDD.md` | Convenção de Erros | Matriz de erros com prefixo `WEBHOOK_*` derivando de `AppError` | `TRANSCRICAO` | `[09:28] Bruno` |
| ADR-006-02 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Reuso de Padrão | Reuso do Pino Logger e do middleware centralizado de tratamento de erros | `TRANSCRICAO` | `[09:29] Bruno` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Endpoints CRUD de configuração de webhook (POST /webhooks) | `TRANSCRICAO` | `[09:31] Marcos` |
| FDD-FLOW-01 | `docs/FDD.md` | Regra de Negócio | Filtro de eventos assinados na inserção da outbox (economizar linhas) | `TRANSCRICAO` | `[09:34] Bruno` |
| FDD-API-03 | `docs/FDD.md` | Requisito Funcional | Endpoint para histórico de entregas `GET /webhooks/:id/deliveries` | `TRANSCRICAO` | `[09:34] Marcos` |
| FDD-API-04 | `docs/FDD.md` | Autorização | Endpoint de replay da DLQ restrito a usuários com papel `UserRole.ADMIN` | `TRANSCRICAO` | `[09:36] Sofia` |
| PRD-OUT-01 | `docs/PRD.md` | Fora de Escopo | Notificações por e-mail para falhas de webhooks descartadas na Fase 1 | `TRANSCRICAO` | `[09:37] Larissa` |
| PRD-OUT-02 | `docs/PRD.md` | Fora de Escopo | Rate limiting de saída mantido fora do escopo inicial para observação | `TRANSCRICAO` | `[09:39] Larissa` |
| PRD-OUT-03 | `docs/PRD.md` | Fora de Escopo | Dashboard visual no frontend fora do escopo (exclusivo APIs backend) | `TRANSCRICAO` | `[09:40] Larissa` |
| FDD-INT-01 | `docs/FDD.md` | Integração de Código | Função `publishWebhookEvent` integrada ao `$transaction` de `changeStatus` | `TRANSCRICAO` | `[09:41] Bruno` |
| FDD-RES-02 | `docs/FDD.md` | Especificação Técnica | Timeout de chamada HTTP pelo worker fixado em 10 segundos | `TRANSCRICAO` | `[09:42] Diego` |
| FDD-API-05 | `docs/FDD.md` | Contrato Público | Cabeçalhos HTTP de envio (`X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp`) | `TRANSCRICAO` | `[09:44] Diego` |
| RFC-IMP-01 | `docs/RFC.md` | Cronograma / Risco | Prazo estimado em 3 Sprints incluindo 2 dias úteis de revisão com AppSec | `TRANSCRICAO` | `[09:46] Larissa` |
| ADR-001-03 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Convenção de Dados | Identificadores únicos padronizados como UUIDv4 | `TRANSCRICAO` | `[09:51] Larissa` |
| ADR-001-04 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Regra de Negócio | Payload gravado na outbox como snapshot do estado no momento do evento | `TRANSCRICAO` | `[09:52] Larissa` |
| FDD-INT-02 | `docs/FDD.md` | Código Fonte | Inserção da chamada outbox no método `changeStatus` dentro da `$transaction` | `CODIGO` | `src/modules/orders/order.service.ts` |
| FDD-INT-03 | `docs/FDD.md` | Código Fonte | Extensão do esquema do banco relacional com novas entidades via Prisma | `CODIGO` | `prisma/schema.prisma` |
| FDD-INT-04 | `docs/FDD.md` | Código Fonte | Extensão da hierarquia `AppError` para novos códigos de erro com prefixo `WEBHOOK_*` | `CODIGO` | `src/shared/errors/http-errors.ts` |
| FDD-INT-05 | `docs/FDD.md` | Código Fonte | Reuso de middlewares de autenticação JWT e controle de papéis (`requireRole`) | `CODIGO` | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Código Fonte | Tratamento centralizado de exceções capturando erros da aplicação e do Prisma | `CODIGO` | `src/middlewares/error.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Código Fonte | Utilização do Pino Logger estruturado para rastreamento de entregas de webhooks | `CODIGO` | `src/shared/logger/index.ts` |

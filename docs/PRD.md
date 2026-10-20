### PRD: Order Management System — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0.0  
Data: 2026-10-05  
Responsável: Marcos (Product Manager) & Larissa (Tech Lead)  

---

### Resumo

O Sistema de Webhooks de Notificação de Pedidos é uma funcionalidade *outbound* desenvolvida para permitir que clientes B2B (ex: Atlas Comercial, MaxDistribuição e Nova Cargo) recebam notificações via HTTP POST em tempo real (latência < 10s) sempre que o status de um pedido for alterado na nossa plataforma OMS. A solução elimina a necessidade de consultas periódicas (*polling*) no endpoint `GET /orders`, reduzindo custos de infraestrutura e prevenindo a perda de clientes estratégicos.

---

### Contexto e problema

**Público-alvo**
- **Clientes B2B / Parceiros Integrados:** Empresas compradoras (ex: Atlas Comercial, MaxDistribuição, Nova Cargo) que possuem sistemas de ERP/WMS próprios e precisam sincronizar o status dos pedidos em tempo real.
- **Equipe de Operações e Suporte Interno:** Operadores e administradores da plataforma que necessitam monitorar entregas e reprocessar falhas.

**Cenários de uso chave**
- **Sincronização em Tempo Real de Status:** O cliente B2B cadastra um webhook para os eventos `SHIPPED` e `DELIVERED`. Assim que o operador atualiza o pedido no OMS, o sistema envia um evento HTTP POST assinado via HMAC-SHA256 para o ERP do cliente em menos de 10 segundos.
- **Recuperação Transparente de Indisponibilidade:** O servidor do cliente entra em manutenção por 2 horas. O OMS retenta o envio em intervalos crescentes (backoff exponencial de 1m, 5m, 30m, 2h, 12h) e entrega a mensagem assim que o receptor volta a ficar online.
- **Reprocessamento de Falhas Definitivas (DLQ):** Um evento atinge a exaustão de retentativas (5 tentativas) e vai para a `webhook_dead_letter`. O operador corrige a URL com o cliente e aciona o reprocessamento manual via API de Admin.

**Onde essa feature será implantada**
- Integrada ao backend principal do Order Management System (Node.js + TypeScript + Prisma + MySQL). O envio assíncrono ocorrerá via worker em processo separado (`src/worker.ts`).

**Problemas priorizados**
- **Sobrecarga de Polling no `GET /orders`:** Clientes B2B batem continuamente no endpoint de busca de pedidos para identificar mudanças, gerando latência e tráfego caro. (Impacto: Alto | Prioridade: Alta)
- **Risco Iminente de Churn Comercial:** A Atlas Comercial formalizou risco de migração para concorrentes se as notificações em tempo real não forem entregues até o fim do trimestre. (Impacto: Crítico | Prioridade: Alta)
- **Falta de Padronização e Segurança em Notificações Externas:** Inexistência de mecanismo seguro para disparo de eventos para fora da rede. (Impacto: Médio | Prioridade: Média)

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar alterações de status em tempo real | Latência de entrega (tempo entre commit do pedido e envio HTTP `200 OK`) | < 10 segundos (p99) |
| Eliminar tráfego de polling desnecessário | Redução de requisições de leitura repetitivas no `GET /orders` por parceiros B2B | Queda de >= 70% nas requisições de polling B2B |
| Garantir resiliência e recuperação de entregas | Taxa de sucesso final de entrega de eventos gerados (incluindo retries) | >= 99.5% dos eventos válidos |
| Retenção de clientes B2B estratégicos | Risco de churn da Atlas Comercial, MaxDistribuição e Nova Cargo por falta de webhook | Redução para 0% de cancelamentos |

---

### Escopo

**Incluso**
- Cadastro, edição, remoção e listagem de webhooks por cliente B2B (CRUD via API REST).
- Filtro de eventos desejados por webhook (ex: notificar apenas transições para `SHIPPED` ou `DELIVERED`).
- Gravação atômica do evento na tabela `webhook_outbox` dentro da transação SQL do `OrderService.changeStatus()`.
- Worker desacoplado em processo separado (`src/worker.ts`) em loop de polling de 2 segundos.
- Assinatura de payload via HMAC-SHA256 no header `X-Signature` e secret individual por endpoint.
- Suporte à rotação de secret via API com *grace period* de 24 horas.
- Validação rígida de URLs exigindo protocolo seguro `https://` e teto de payload em 64KB.
- Política de retry com 5 tentativas (backoff exponencial 1m, 5m, 30m, 2h, 12h) cobrindo ~15 horas.
- Redirecionamento para Dead Letter Queue (`webhook_dead_letter`) em falhas definitivas.
- Endpoint de replay manual da DLQ restrito a usuários com perfil `ADMIN` (`POST /admin/webhooks/dead-letter/:id/replay`).
- Endpoint de consulta de histórico de entregas por webhook (`GET /webhooks/:id/deliveries`).

**Fora de escopo**
- Notificações por e-mail para administradores de clientes quando webhooks falharem (postergado para fase futura).
- Rate limiting de saída configurável por cliente (a ser observado em produção antes de decidir).
- Dashboard visual no frontend para gestão de webhooks (escopo exclusivo de APIs backend; painel é projeto separado).
- Suporte a múltiplos workers em paralelo com ordenação global garantida (limitado a single-worker na fase 1).

---

### Requisitos funcionais

#### RF-01 Cadastrar Endpoint de Webhook
Permitir que clientes B2B cadastrem URLs de destino HTTPS para receber notificações de mudanças de status de pedidos.

**Fluxo principal**
1. O cliente envia uma requisição `POST /webhooks` contendo `customerId`, `url` (HTTPS) e lista de `events`.
2. O sistema valida os campos via schema Zod.
3. O sistema gera uma secret criptográfica única (ex: `whsec_...`).
4. O sistema persiste o registro na tabela `webhook_endpoints` e retorna o ID e a secret gerada.

**Fluxos alternativos e exceções**
- Se a URL informada não utilizar o protocolo `https://`, o sistema rejeita com código 400 (`WEBHOOK_INVALID_URL`).

**Erros previstos**
- `WEBHOOK_INVALID_URL` (400), `VALIDATION_ERROR` (400), `UNAUTHORIZED` (401).

**Prioridade:** Alta

---

#### RF-02 Listar e Consultar Webhooks do Cliente
Permitir a consulta dos webhooks cadastrados para um determinado `customerId`.

**Fluxo principal**
1. Requisição `GET /webhooks?customerId=...`.
2. O sistema valida a autenticação JWT.
3. O sistema busca os webhooks do cliente e retorna a lista (omitindo a chave secret por segurança).

**Fluxos alternativos e exceções**
- Se o cliente não possuir webhooks, retorna lista vazia.

**Erros previstos**
- `UNAUTHORIZED` (401), `FORBIDDEN` (403).

**Prioridade:** Média

---

#### RF-03 Editar e Desativar Webhook
Permitir a atualização da URL, eventos assinados ou desativação de um webhook.

**Fluxo principal**
1. Requisição `PATCH /webhooks/:id` com os campos a serem alterados.
2. O sistema atualiza os dados no banco de dados e retorna o objeto atualizado.

**Fluxos alternativos e exceções**
- Se o webhook não for encontrado, retorna erro 404.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INVALID_URL` (400).

**Prioridade:** Média

---

#### RF-04 Remover Webhook
Permitir a exclusão de um webhook cadastrado.

**Fluxo principal**
1. Requisição `DELETE /webhooks/:id`.
2. O sistema remove o registro e desativa os envios futuros.

**Fluxos alternativos e exceções**
- Se o ID não existir, retorna 404.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Média

---

#### RF-05 Disparo Atômico de Evento no Change Status (Transactional Outbox)
Garantir que toda mudança de status de pedido no OMS registre o evento de notificação atomicamente na tabela `webhook_outbox`.

**Fluxo principal**
1. Operação de alteração de status invocada em `OrderService.changeStatus()`.
2. Na mesma transação SQL (`$transaction`), o sistema atualiza a ordem, grava a história e executa `publishWebhookEvent()`.
3. Para cada webhook ativo do cliente que assine aquele status, o evento é gravado com status `PENDING` e snapshot JSON do pedido.
4. A transação SQL é commitada.

**Fluxos alternativos e exceções**
- Se nenhum webhook do cliente assinar aquele status, nenhum registro é inserido na outbox (otimização de banco).
- Se a transação do pedido falhar (ex: falta de estoque), o evento da outbox é cancelado junto via rollback.

**Erros previstos**
- `INVALID_STATUS_TRANSITION` (409), `INSUFFICIENT_STOCK` (422).

**Prioridade:** Alta

---

#### RF-06 Processamento Assíncrono pelo Worker e Envio HTTP
Processar a fila da outbox e enviar notificações HTTP POST com segurança e headers padronizados.

**Fluxo principal**
1. O worker (`src/worker.ts`) lê eventos `PENDING` a cada 2s.
2. Gera a assinatura HMAC-SHA256 do payload usando a secret do webhook.
3. Dispara HTTP POST para o endpoint do cliente com os headers `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp`.
4. Em caso de HTTP 2xx em < 10s, marca o evento como `DELIVERED`.

**Fluxos alternativos e exceções**
- Se o cliente responder com erro (4xx/5xx) ou dar timeout (10s), o worker agenda a retentativa com backoff exponencial.

**Erros previstos**
- `WEBHOOK_DELIVERY_FAILED` (502).

**Prioridade:** Alta

---

#### RF-07 Rotação de Secret com Grace Period
Permitir a rotação da chave secreta de um webhook mantendo a chave antiga válida por 24 horas.

**Fluxo principal**
1. Requisição `POST /webhooks/:id/rotate-secret`.
2. O sistema move a `secret` atual para `previousSecret`, define `secretExpiresAt = NOW() + 24h` e gera uma nova `secret`.
3. Retorna a nova secret para o cliente.

**Fluxos alternativos e exceções**
- Durante as 24 horas, o worker assina com a nova secret, mas o cliente pode validar com ambas se necessário.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Média

---

#### RF-08 Reprocessamento Manual de DLQ (Replay Admin)
Permitir que usuários com perfil `ADMIN` re-enfileirem eventos descartados na Dead Letter Queue.

**Fluxo principal**
1. Requisição `POST /admin/webhooks/dead-letter/:id/replay`.
2. O sistema valida se o usuário possui `UserRole.ADMIN`.
3. O evento é removido da `webhook_dead_letter` e reinserido na `webhook_outbox` como `PENDING` com contagem de tentativas zerada.

**Fluxos alternativos e exceções**
- Se o usuário for um operador comum (`OPERATOR`), o sistema retorna 403 Forbidden.

**Erros previstos**
- `FORBIDDEN` (403), `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Alta

---

#### RF-09 Consulta do Histórico de Entregas
Permitir a visualização do histórico de tentativas de entrega de um webhook.

**Fluxo principal**
1. Requisição `GET /webhooks/:id/deliveries?page=1&pageSize=20`.
2. Retorna a lista paginada com status code HTTP, tempo de resposta e número de tentativas.

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404).

**Prioridade:** Média

---

### Requisitos não funcionais

**Performance**
- O tempo entre o commit da alteração do pedido e o disparo da requisição pelo worker deve ser inferior a 2 segundos em condições normais de carga.
- Timeout de requisições HTTP efetuadas pelo worker limitado rigorosamente a 10 segundos.

**Disponibilidade**
- Uptime do worker de processamento de 99.9% mensal. O worker roda como processo Node.js independente para que crashes da API web não interrompam o processamento de notificações.

**Segurança e autorização**
- Obrigatoriedade de protocolo `https://` para todas as URLs de webhooks cadastradas.
- Autenticação e integridade do payload via HMAC-SHA256 no cabeçalho `X-Signature`.
- Restrição de endpoints de gestão via JWT e restrição do endpoint de replay da DLQ ao papel `UserRole.ADMIN`.

**Observabilidade**
- Logs estruturados em formato JSON usando Pino Logger em [`src/shared/logger/index.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/logger/index.ts).
- Rastreamento ponta a ponta correlacionando `eventId` nos logs de alteração de pedido e disparo do worker.

**Confiabilidade e integridade de dados**
- Atomicidade garantida pelo padrão Transactional Outbox usando o `$transaction` do Prisma no MySQL.
- Semântica de entrega *at-least-once* com UUID no header `X-Event-Id` para deduplicação no cliente.

**Compatibilidade e portabilidade**
- APIs REST em JSON compatíveis com o padrão do projeto. Payloads formatados conforme padrão ISO-8601 para datas.

**Compliance**
- Trilha de auditoria completa na tabela `webhook_dead_letter` registrando motivo de falhas e usuário admin responsável por acionar o replay.

**Acessibilidade no frontend consumidor**
- N/A (Feature exclusivamente backend exposta via API REST JSON).

---

### Arquitetura e abordagem

**Abordagem**
- Implementação de um módulo de **Outbound Webhooks** em `src/modules/webhooks/` desacoplado do ciclo de requisição HTTP através do padrão **Transactional Outbox** gravado no MySQL e processado por um **Worker Node.js em Polling** (`src/worker.ts`).

**Componentes**
- **Módulo de Webhooks API (`src/modules/webhooks/`):** Controladores, serviços, repositórios e schemas Zod para CRUD e gestão.
- **Transactional Outbox helper (`publishWebhookEvent`):** Executado dentro de [`src/modules/orders/order.service.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts).
- **Worker Process (`src/worker.ts`):** Processo separado rodando loop de polling a cada 2s com engine de retries e assinador HMAC-SHA256.
- **Banco de Dados MySQL:** Tabelas `webhook_endpoints`, `webhook_outbox` e `webhook_dead_letter`.

**Integrações**
- **Order Management Service (`OrderService.changeStatus`):** Disparador primário dos eventos de alteração de status.
- **Endpoints de Clientes B2B:** Receptores externos HTTPS consumindo payloads em formato JSON.

---

### Decisões e trade-offs

#### Decisão: Padrão Transactional Outbox em MySQL em vez de Fila/Broker (Redis/RabbitMQ)
- **Justificativa:** Mantém a consistência atômica com a mudança de status do pedido sem adicionar novos componentes de infraestrutura para a equipe gerenciar.
- **Trade-off:** Aumento de escritas no banco MySQL existente e necessidade de polling periódico pelo worker.

#### Decisão: Worker em Processo Separado com Polling de 2s
- **Justificativa:** Evita que falhas ou deploys na API principal interrompam os envios e atende com folga a latência desejada (< 10s).
- **Trade-off:** Geração de consultas periódicas no MySQL mesmo sem novos eventos (mitigado por índices adequados).

#### Decisão: Assinatura HMAC-SHA256 com Secret Única por Endpoint
- **Justificativa:** Garante a autenticidade e integridade da notificação, isolando o impacto caso a chave de um cliente seja vazada.
- **Trade-off:** Exige que os clientes B2B implementem a validação da assinatura HMAC no receptor.

---

### Dependências

#### Dependência de Infraestrutura: Banco MySQL Existente
- A aplicação utiliza o cluster MySQL existente configurado no Prisma (`DATABASE_URL`). As novas tabelas serão criadas via Prisma Migrations.

#### Dependência de Segurança: Revisão de AppSec (Sofia)
- A entrega exige alocação de 2 dias úteis da Engenheira de Segurança (Sofia) antes do deploy em produção para revisão da geração de secrets e algoritmo HMAC.

---

### Riscos e mitigação

#### Acúmulo de Registros Processados na Tabela Outbox
- **Probabilidade:** Alta
- **Impacto:** Degradação de performance do MySQL em consultas de polling.
- **Mitigação:**
  - Criação de índices compostos em `(status, nextRetryAt, createdAt)`.
  - Batching de leitura no worker.
- **Plano de contingência:** Criação de rotina cron para expurgo de eventos com status `DELIVERED` há mais de 30 dias.

#### Indisponibilidade Prolongada do Cliente B2B
- **Probabilidade:** Média
- **Impacto:** Retentativas repetidas ocupando ciclos do worker.
- **Mitigação:**
  - Limite de 5 tentativas com backoff exponencial escalonado em até 15 horas.
  - Movimentação automática para a DLQ após a 5ª falha.
- **Plano de contingência:** Acionamento de reprocessamento manual via API de Admin após reabertura do endpoint do cliente.

---

### Critérios de aceitação

Checklist objetivo que define se a feature está pronta:

- [ ] Endpoints HTTP para CRUD de webhooks (`POST`, `GET`, `PATCH`, `DELETE /webhooks`) funcionando conforme contratos JSON.
- [ ] Alteração de status de pedido no `OrderService` insere atomicamente o evento na tabela `webhook_outbox`.
- [ ] Worker (`src/worker.ts`) envia notificações via HTTP POST em menos de 10 segundos com os cabeçalhos `X-Event-Id`, `X-Signature`, `X-Webhook-Id` e `X-Timestamp`.
- [ ] Tentativas de cadastro com URLs `http://` são rejeitadas com erro 400 (`WEBHOOK_INVALID_URL`).
- [ ] Falhas temporárias ativam retry com backoff exponencial (1m, 5m, 30m, 2h, 12h) e falhas definitivas vão para `webhook_dead_letter`.
- [ ] Endpoint `POST /admin/webhooks/dead-letter/:id/replay` restrito exclusivamente a usuários com papel `UserRole.ADMIN`.
- [ ] Rotação de secret (`POST /webhooks/:id/rotate-secret`) mantém a chave anterior válida por 24 horas.

---

### Testes e validação

**Tipos de teste obrigatórios**
- **Testes Unitários:** Validação das funções de cálculo de HMAC-SHA256, cálculo de backoff exponencial e schemas Zod.
- **Testes de Integração:** Validação da gravação atômica na outbox durante o `OrderService.changeStatus()` usando transações do Prisma.
- **Testes End-to-End (E2E):** Simulação de servidor receptor B2B recebendo webhooks, testando casos de sucesso (200 OK), retry (500 Error) e exaustão (DLQ).
- **Teste de Segurança:** Revisão do código de geração de secrets e rotação com a equipe de AppSec.

**Estratégia de validação**
- Execução da suíte de testes automatizados via Vitest (`npm test`).
- Teste exploratório em ambiente de staging criando pedidos, simulando oscilações de rede no receptor e executando o replay manual via endpoint de admin.

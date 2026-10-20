# ADR-005: Worker em Processo Separado em Polling

- **Status:** Aceito
- **Data:** 2026-10-05
- **Autores:** Diego (Engenheiro Sênior), Larissa (Tech Lead), Bruno (Engenheiro Pleno)

## Contexto

Após a alteração de status do pedido gravar o registro na tabela `webhook_outbox`, faz-se necessário um componente que leia essa tabela, identifique as notificações pendentes, monte os cabeçalhos de segurança (HMAC-SHA256) e efetue a chamada HTTP para o destino de cada cliente B2B.

Foi debatido como o worker deve ser executado na infraestrutura da aplicação e como ele deve ser notificado sobre novos eventos na outbox.

## Decisão

Executar o worker como um **processo Node.js independente** com estratégia de **Polling a cada 2 segundos** no banco de dados MySQL.

1. **Separação de Processos:**
   * O worker terá um ponto de entrada próprio no projeto (`src/worker.ts`), iniciado via script dedicado (ex: `npm run worker`).
   * Não rodará dentro da mesma instância do servidor HTTP principal (`src/server.ts`).
   * Instanciará seu próprio `PrismaClient` independente para gerenciar seu pool de conexões com o MySQL.

2. **Estratégia de Polling:**
   * A cada 2 segundos, o worker executa uma consulta buscando em batch os registros da `webhook_outbox` com status `PENDING` ou `RETRY_PENDING` e `nextRetryAt <= NOW()`, ordenados por `createdAt ASC`.
   * O intervalo de 2 segundos satisfaz com folga o requisito dos clientes B2B de receber a notificação em tempo real (latência abaixo de 10 segundos).

3. **Garantia de Ordenação por Pedido (Single-Worker):**
   * Inicialmente, a aplicação rodará com **uma única instância do worker (single-worker)**.
   * Isso garante a ordenação estrita das notificações de cada pedido (`order_id`) sem necessidade de travas distribuídas complexas ou partições nesta primeira fase.

4. **Timeout de Chamada HTTP:**
   * O tempo máximo de espera (*timeout*) para a resposta HTTP do endpoint do cliente B2B é fixado em **10 segundos**. Se o cliente não responder em 10s, o envio é abortado, considerado falha e agendado para retry.

## Alternativas Consideradas

* **Worker Embutido no Servidor HTTP (API):** Rejeitado pois se o servidor web reiniciar por conta de deploys ou tráfego elevado, o ciclo de vida do envio de notificações seria interrompido ou perdido.
* **Triggers de Banco de Dados:** Rejeitado pois o MySQL não possui suporte nativo a pub/sub com notificadores externos (tipo `NOTIFY/LISTEN` do PostgreSQL). Usar triggers para executar comandos de sistema ou HTTP seria frágil e arriscado.
* **Múltiplos Workers Paralelos sem Lock:** Rejeitado nesta fase para evitar condições de corrida em que a notificação `SHIPPED` pudesse ser entregue antes de `PAID` devido a paralelismo de threads.

## Consequências

### Positivas
* **Isolamento de Processos:** Reinicializações ou picos de tráfego na API não afetam a execução do worker de webhooks e vice-versa.
* **Cumprimento do SLA de Latência:** O polling de 2s garante tempo de resposta bem inferior ao teto de 10 segundos exigido pelos clientes.
* **Ordenação Garantida:** Single-worker assegura ordem sequencial natural de envio por `created_at`.

### Negativas
* **Polling Contínuo no MySQL:** Consultas frequentes (a cada 2s) ocorrem mesmo se não houver eventos pendentes. Mitigado por índices em `status`, `nextRetryAt` e `createdAt` e leitura em lotes pequenos.

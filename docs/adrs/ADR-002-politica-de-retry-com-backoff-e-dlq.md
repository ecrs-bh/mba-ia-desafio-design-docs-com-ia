# ADR-002: Política de Retry com Backoff Exponencial e DLQ

- **Status:** Aceito
- **Data:** 2026-10-05
- **Autores:** Diego (Engenheiro Sênior), Larissa (Tech Lead), Bruno (Engenheiro Pleno)

## Contexto

Endpoints de webhooks mantidos por clientes B2B estão sujeitos a indisponibilidades temporárias (ex: reinicializações, manutenções programadas ou flutuações de rede). A entrega de notificações precisa ser resiliente a essas falhas sem travar a fila de envios e sem tentar indefinidamente em endpoints mortos.

Na reunião técnica, foram discutidos o limite de tentativas de reenvio, a progressão do intervalo entre tentativas (*backoff*) e o tratamento de falhas definitivas.

## Decisão

Adotar uma política de retentativas com **Backoff Exponencial** configurada para **5 tentativas no máximo**, seguida do envio do evento para uma **Dead Letter Queue (DLQ)** persistida no banco MySQL.

1. **Progressão do Backoff:**
   * Tentativa 1: Envio inicial (imediato / polling).
   * Tentativa 2: Retry após 1 minuto.
   * Tentativa 3: Retry após 5 minutos.
   * Tentativa 4: Retry após 30 minutos.
   * Tentativa 5: Retry após 2 horas.
   * Tentativa 6 (Última): Retry após 12 horas.
   * Janela total de resiliência: aproximadamente **15 horas** desde a primeira falha até a exaustão.

2. **Dead Letter Queue (DLQ):**
   * Caso a 5ª tentativa de retry (6º envio) falhe, o evento é movido para a tabela `webhook_dead_letter`.
   * A tabela `webhook_dead_letter` armazenará o payload original, o ID do webhook, a quantidade de tentativas efetuadas, a última mensagem/código de erro e o timestamp da falha final.

3. **Reprocessamento Manual (Replay):**
   * Disponibilizar um endpoint de administração `POST /admin/webhooks/dead-letter/:id/replay`.
   * O reprocessamento recoloca o evento na `webhook_outbox` em estado pendente, zerando a contagem de tentativas.
   * O endpoint exige autenticação e autorização com papel `ADMIN` (`requireRole(UserRole.ADMIN)`).

## Alternativas Consideradas

* **3 Tentativas de Retry:** Rejeitada por ser uma janela muito curta (~30 minutos), insuficiente para cobrir janelas normais de manutenção de clientes B2B (que chegam a 2 horas).
* **Retry Indefinito:** Rejeitado pois manteria eventos falhos acumulando na outbox perpetuamente quando um cliente desativasse seu endpoint sem avisar.
* **Flag de Falha na própria Outbox (sem tabela separada de DLQ):** Rejeitada para manter a tabela `webhook_outbox` limpa e otimizada apenas para processamento de eventos ativos, isolando o histórico de falhas e auditoria na `webhook_dead_letter`.

## Consequências

### Positivas
* **Resiliência:** Cobertura de até 15 horas contra indisponibilidades temporárias de clientes B2B.
* **Isolamento de Erros:** Eventos falhos definitivos não poluem o fluxo de entregas ativas.
* **Auditabilidade e Recuperação:** Possibilidade de replay manual controlado pela equipe de operação via API autenticada com auditoria de ações.

### Negativas
* **Latência de Exaustão:** Um evento com falha permanente leva ~15 horas até ser finalmente movido para a DLQ.

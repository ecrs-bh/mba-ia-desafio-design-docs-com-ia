# Processo de Produção: Pacote de Design Docs para Sistema de Webhooks

## Sobre o Desafio

Este repositório contém a documentação técnica completa para a construção da nova funcionalidade de **Sistema de Webhooks de Notificação de Pedidos (Outbound Webhooks)** no Order Management System (OMS) existente. A solução foi projetada a partir da transcrição literal de uma reunião técnica entre Tech Lead, Product Manager, Engenheiros de Software e Engenheira de Segurança ([`TRANSCRICAO.md`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md)) combinada com a análise direta da arquitetura de código-fonte em TypeScript/Node.js do repositório.

O objetivo principal deste desafio foi atuar como maestro no uso de Inteligência Artificial para transformar discussões informais e trechos de código em um pacote de documentos de arquitetura acionáveis, consistentes e livres de alucinações. O pacote é composto por PRD, RFC, FDD, 6 ADRs, um Tracker de Rastreabilidade transversal e este guia do processo.

---

## Ferramentas de IA Utilizadas

- **Antigravity AI (Google DeepMind - Modelo Gemini 3.6 Flash):** Ferramenta primária de engenharia e Pair Programming agentic. Utilizada para análise profunda da base de código TypeScript/Prisma, varredura rigorosa da transcrição com marcação temporal, elaboração incremental dos documentos Markdown e validação cruzada dos critérios de aceite.
- **Claude / Cursor IDE Assistant:** Utilizado em sessões auxiliares de revisão crítica para checar a formatação de tabelas, validações de contratos OpenAPI/JSON e adequação do texto às diretrizes MADR.

---

## Workflow Adotado

A produção da documentação seguiu um fluxo estritamente incremental e orientado a decisões ("bottom-up técnico para top-down de produto"), conforme as seguintes etapas:

```
[1. Leitura & Avaliação] ---> [2. ADRs (Decisões)] ---> [3. RFC (Arquitetura)]
                                                                  |
[6. README & Validação] <--- [5. Tracker (Rastreio)] <--- [4. FDD & PRD (Detalhes)]
```

1. **Leitura e Mapeamento Inicial:** Análise completa da transcrição (`TRANSCRICAO.md`), do código base (`src/modules/orders/order.service.ts`, `prisma/schema.prisma`, `src/shared/errors/http-errors.ts`) e alinhamento do escopo com o usuário sem alteração de código.
2. **ADRs Primeiro (Etapa 1):** Formalização isolada das 6 decisões arquiteturais principais (`docs/adrs/ADR-001` a `ADR-006`). Definir primeiro as decisões criou a espinha dorsal técnica necessária para evitar ambiguidades nos documentos mais altos.
3. **RFC da Feature (Etapa 2):** Elaboração da proposta arquitetural (`docs/RFC.md`) integrando o resumo técnico, alternativas descartadas (Disparo Síncrono e Redis/RabbitMQ) e questões em aberto (Rate Limiting e Notificações por E-mail).
4. **FDD da Feature (Etapa 3):** Construção da especificação detalhada de implementação (`docs/FDD.md`) baseada no esqueleto `fdd_esqueleto.md`, incluindo diagramas Mermaid, 5 contratos públicos HTTP, matriz de erros com prefixo `WEBHOOK_*` e a seção de integração com arquivos reais do projeto.
5. **PRD da Feature (Etapa 4):** Elaboração do documento de produto (`docs/PRD.md`) seguindo a estrutura `esqueleto_prd.md`, consolidando objetivos quantitativos, cenários de uso, 9 requisitos funcionais detalhados e itens fora de escopo.
6. **Tracker de Rastreabilidade (Etapa 5):** Criação da matriz de rastreabilidade (`docs/TRACKER.md`) mapeando 40 itens dos documentos de volta à transcrição (timestamps `[hh:mm] Nome`) e ao código fonte.
7. **Documentação do Processo (Etapa 6):** Atualização do `README.md` principal relatando a jornada, ferramentas, prompts e iterações.

---

## Prompts Customizados

Abaixo estão dois dos prompts principais utilizados durante o processo para orientar a geração dos documentos com máximo rigor:

### Prompt 1: Extração Rígida de Decisões e Geração dos ADRs (Etapa 1)

```markdown
Você é um Engenheiro Principal de Software atuando no projeto OMS em Node.js/TypeScript.
Com base unicamente no arquivo TRANSCRICAO.md e na análise do código em src/ e prisma/schema.prisma:

1. Gere 6 arquivos de ADR em docs/adrs/ no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).
2. As ADRs devem cobrir exatamente:
   - ADR-001: Padrão Outbox no MySQL
   - ADR-002: Política de retry com backoff exponencial e DLQ
   - ADR-003: Autenticação HMAC-SHA256 com secret por endpoint
   - ADR-004: Garantia at-least-once com X-Event-Id
   - ADR-005: Worker em processo separado em polling
   - ADR-006: Reuso dos padrões existentes do projeto
3. A ADR-006 DEVE referenciar explicitamente arquivos reais: src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts, src/middlewares/auth.middleware.ts, src/middlewares/error.middleware.ts e src/shared/logger/index.ts.
4. NÃO invente requisitos ou decisões que não foram explicitamente combinados na transcrição.
```

### Prompt 2: Validação Rastreável e Construção do Tracker (Etapa 5)

```markdown
Gere o arquivo docs/TRACKER.md contendo a tabela de rastreabilidade no formato exato:
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |

Regras estritas de geração:
- Mapeie pelo menos 35 itens cobrindo docs/PRD.md, docs/RFC.md, docs/FDD.md e docs/adrs/*.
- Garantir que pelo menos 70% das linhas possuam Fonte = TRANSCRICAO com localização exata no formato `[hh:mm] Nome` extraído do arquivo TRANSCRICAO.md.
- Garantir que pelo menos 5 linhas possuam Fonte = CODIGO com caminhos de arquivos reais (ex: src/modules/orders/order.service.ts, prisma/schema.prisma, src/shared/errors/http-errors.ts, etc.).
- Qualquer item sem origem identificável na transcrição ou no código deve ser removido.
```

---

## Iterações e Ajustes

Durante a produção da documentação, a atuação como maestro exigiu revisões críticas e ajustes diretos nas respostas geradas pela IA para evitar superficialidade ou descumprimento de requisitos. As duas iterações principais foram:

1. **Ajuste na Estrutura do FDD (Etapa 3):**
   * *Problema:* No primeiro rascunho do FDD, a IA gerou os contratos de forma genérica e omitiu o esqueleto rígido exigido em `fdd_esqueleto.md`, além de não destacar a seção de integração de arquivos com caminhos explicitados.
   * *Correção:* Forçamos o reaproveitamento estrito do template `fdd_esqueleto.md` e instruímos a IA a detalhar requisição e resposta de 5 contratos HTTP com campos completos em JSON, além de abrir a seção 9 dedicada à integração com 5 arquivos reais do projeto (`order.service.ts`, `schema.prisma`, `http-errors.ts`, `auth.middleware.ts`, `logger`).

2. **Refinamento do PRD e Mapeamento de Fora de Escopo (Etapa 4):**
   * *Problema:* A IA tentou incluir o envio de e-mails em caso de erro e um dashboard visual no frontend como requisitos funcionais do PRD.
   * *Correção:* Revisamos a transcrição nos timestamps `[09:37] Larissa` (descarte de e-mail) e `[09:40] Larissa` (descarte de dashboard), corrigindo o PRD para mover esses itens explicitamente para a seção "Fora de Escopo", alinhando o documento com a decisão real do time.

---

## Como Navegar a Entrega

Recomendamos a leitura dos documentos entregues na seguinte ordem lógica:

1. [`docs/PRD.md`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/PRD.md): **Visão de Produto** (Problema B2B, metas quantitativas, requisitos funcionais e escopo).
2. [`docs/RFC.md`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/RFC.md): **Proposta Arquitetural** (Visão geral da solução, alternativas descartadas e cronograma).
3. [`docs/adrs/`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/): **Decisões Registradas** (6 ADRs em formato MADR detalhando cada escolha técnica).
   * [`ADR-001: Padrão Outbox no MySQL`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-001-padrao-outbox-no-mysql.md)
   * [`ADR-002: Política de Retry com Backoff e DLQ`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md)
   * [`ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
   * [`ADR-004: Garantia At-Least-Once com X-Event-Id`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)
   * [`ADR-005: Worker em Processo Separado em Polling`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md)
   * [`ADR-006: Reuso dos Padrões Existentes do Projeto`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
4. [`docs/FDD.md`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/FDD.md): **Especificação de Construção** (Contratos HTTP, diagramas Mermaid, matriz de erros e integração de código).
5. [`docs/TRACKER.md`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/TRACKER.md): **Matriz de Rastreabilidade** (Referência cruzada de timestamps da transcrição e arquivos de código).

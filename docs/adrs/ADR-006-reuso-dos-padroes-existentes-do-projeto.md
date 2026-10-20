# ADR-006: Reuso dos Padrões Existentes do Projeto

- **Status:** Aceito
- **Data:** 2026-10-05
- **Autores:** Bruno (Engenheiro Pleno), Larissa (Tech Lead), Sofia (Engenheira de Segurança)

## Contexto

A adição do módulo de webhooks deve manter total coerência com a arquitetura de código, convenções e padrões já adotados na codebase do Order Management System (OMS). A inclusão de novos frameworks, padrões divergentes de tratamento de erro ou estruturas de rotas causaria fragmentação e aumentaria o débito técnico.

## Decisão

Reutilizar integralmente a infraestrutura compartilhada, convenções de código e padrões arquiteturais existentes na aplicação.

1. **Estrutura Modular por Domínio:**
   * O novo código ficará concentrado no diretório `src/modules/webhooks/`, contendo a divisão em camadas padronizada do projeto: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` e `webhook.processor.ts`.

2. **Tratamento de Erros e Códigos com Prefixo `WEBHOOK_*`:**
   * Reutilização da classe base [`AppError`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts) e das exceções HTTP em [`src/shared/errors/http-errors.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/errors/http-errors.ts) (`NotFoundError`, `BadRequestError`, `ValidationError`, `ConflictError`, `ForbiddenError`, `UnauthorizedError`).
   * Padronização de novos códigos de erro da funcionalidade utilizando o prefixo `WEBHOOK_*` (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_INACTIVE`).
   * Reuso do middleware centralizado de captura de erros em [`src/middlewares/error.middleware.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts).

3. **Autenticação, Autorização e Middleware:**
   * Utilização dos middlewares em [`src/middlewares/auth.middleware.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts) (`requireAuth` para autenticação via JWT).
   * Utilização do middleware `requireRole(UserRole.ADMIN)` para proteger o endpoint de replay manual de Dead Letter Queue (`POST /admin/webhooks/dead-letter/:id/replay`).

4. **Observabilidade e Logging:**
   * Reuso do logger estruturado baseado em Pino disponível em [`src/shared/logger/index.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/logger/index.ts) e do [`src/middlewares/request-logger.middleware.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/request-logger.middleware.ts).

5. **Validação de Entradas com Zod:**
   * Utilização de schemas Zod em conjunto com o middleware [`src/middlewares/validate.middleware.ts`](file:///home/s052167316/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/validate.middleware.ts) para validação estrita de rotas HTTP.

## Alternativas Consideradas

* **Criação de Framework/Abstração Customizada:** Rejeitada por gerar complexidade desnecessária e divergir da forma como os módulos de `orders`, `products`, `customers`, `users` e `auth` estão construídos.
* **Sistema de Permissões Customizado:** Rejeitado para o replay da DLQ em favor da reutilização direta do enum `UserRole.ADMIN` já suportado pelo middleware `requireRole`.

## Consequências

### Positivas
* **Manutenibilidade:** Mantém a consistência total da base de código, simplificando code reviews e a curva de aprendizado da equipe.
* **Eficiência:** Evita a reescrita de utilitários de autenticação, resposta HTTP, validação e logging que já se provaram estáveis na aplicação.

### Negativas
* **Acoplamento às Convenções Existentes:** Dependência mantida dos padrões e bibliotecas já escolhidos para o projeto (Zod, Prisma, Pino).

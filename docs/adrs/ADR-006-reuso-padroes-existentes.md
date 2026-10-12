# ADR-006: Reuso dos padrões existentes do projeto para o módulo de webhooks

## Status

Aceito — 2026-10 (reunião técnica de 09:00, decisão em [09:27]-[09:30])

## Contexto

O OMS já tem convenções estabelecidas de estrutura de código, tratamento de erros e observabilidade: módulos por domínio em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas` (ver, por exemplo, `src/modules/customers/`, `src/modules/orders/`); classes de erro derivadas de `AppError` (`src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`) com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`; logger Pino centralizado (`src/shared/logger/index.ts`); e um middleware de erro único que já trata `AppError`, `ZodError` e erros do Prisma (`src/middlewares/error.middleware.ts`). Era preciso decidir se a feature de webhooks introduz padrões novos ou segue os existentes ([09:27] Bruno).

## Decisão

O módulo de webhooks segue **integralmente** os padrões já estabelecidos no código:

- Estrutura de módulo em `src/modules/webhooks/` com `controller`, `service`, `repository`, `routes`, `schemas`, seguindo o mesmo layout de `src/modules/orders/` e `src/modules/customers/` ([09:27]-[09:28] Bruno, Diego).
- O worker vive como entry point separado (`src/worker.ts`, análogo a `src/server.ts`), com a lógica de processamento dentro do módulo (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`) ([09:28] Bruno).
- Erros do módulo estendem `AppError` com códigos prefixados por **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), seguindo o padrão já usado por `InsufficientStockError`/`InvalidStatusTransitionError` em `src/shared/errors/http-errors.ts` ([09:28]-[09:29] Bruno, Larissa).
- Reaproveita o logger Pino existente (`src/shared/logger/index.ts`) sem introduzir novo mecanismo de log ([09:29] Bruno).
- O middleware de erro centralizado (`src/middlewares/error.middleware.ts`) trata os novos erros automaticamente, pois já reconhece qualquer subclasse de `AppError` — nenhuma alteração necessária nele ([09:29] Bruno).
- Validação de entrada segue o padrão de schemas Zod já usado em `order.schemas.ts` / `customer.schemas.ts`.
- O worker usa um `PrismaClient` próprio (nova instância, mesmo `DATABASE_URL`), já que `PrismaClient` é por processo — não compartilha a instância de `src/config/database.ts` usada pela API ([09:29]-[09:30] Diego, Bruno).
- O endpoint administrativo de replay de DLQ reaproveita o middleware `requireRole` já existente em `src/middlewares/auth.middleware.ts`, exigindo role `ADMIN` ([09:35]-[09:36] Sofia, Larissa).

## Alternativas Consideradas

- **Introduzir uma camada de erros ou um sistema de logging específico para webhooks:** descartado — não há motivo identificado na reunião para desviar do padrão existente, e isso fragmentaria o tratamento de erros entre módulos ([09:28]-[09:29] Bruno, Diego).

## Consequências

**Positivas**
- Baixo custo de onboarding para quem já conhece a codebase: a feature de webhooks "parece" com qualquer outro módulo do OMS.
- Reaproveita toda a infraestrutura de observabilidade e tratamento de erro já testada em produção, sem retrabalho.
- Facilita a revisão de código, pois os revisores já conhecem as convenções aplicadas.

**Negativas / Trade-offs**
- Qualquer limitação do padrão atual (ex.: `AppError` não ter campos específicos para contexto de webhook, como `webhookId` ou `attemptNumber`) é herdada pelo módulo novo; extensões desses padrões, se necessárias, afetam código compartilhado usado por todos os módulos, não só webhooks.
- `requireRole('ADMIN')` restringe o replay de DLQ a administradores — o CRUD de configuração de webhook, por decisão separada, aceita qualquer role autenticada por ora ([09:36]-[09:37] Sofia, Marcos), o que é uma postura permissiva que pode precisar ser revisitada.

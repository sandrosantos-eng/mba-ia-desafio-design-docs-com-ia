# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O OMS atual (`src/`) é um monólito Express + Prisma/MySQL organizado em módulos de domínio (`src/modules/{auth,users,customers,products,orders}`), sem nenhum mecanismo de notificação externa. A feature de webhooks adiciona um novo módulo `src/modules/webhooks/` e um novo processo (`src/worker.ts`) que entrega eventos de mudança de status de pedido para endpoints HTTP cadastrados por clientes B2B, de forma assíncrona, confiável e autenticada.

A decisão arquitetural (outbox transacional + worker + HMAC + retry/DLQ) está em `docs/RFC.md` e nos ADRs em `docs/adrs/`. Este documento detalha **como construir**.

## Objetivos técnicos

- Garantir que todo evento de mudança de status gerado é entregue pelo menos uma vez (at-least-once), sem perda silenciosa.
- Garantir que a geração do evento é atômica com a mudança de status (sem evento órfão nem status mudado sem evento).
- Manter o impacto em `OrderService.changeStatus` mínimo e isolado.
- Entregar o evento em até ~10 segundos no caminho feliz (polling de 2s + tentativa imediata).
- Autenticar e proteger a integridade de cada entrega via HMAC-SHA256.

## Escopo e exclusões

**Dentro do escopo:**
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`.
- Módulo `src/modules/webhooks/` (CRUD de endpoints, listagem de deliveries, rotação de secret).
- Função `publishWebhookEvent` integrada em `OrderService.changeStatus`.
- Worker `src/worker.ts` com polling, retry/backoff, HMAC, DLQ.
- Endpoint administrativo de replay de DLQ.

**Fora de escopo (explicitamente descartado ou adiado na reunião):**
- Notificação proativa (e-mail) ao cliente quando o webhook dele está falhando ([09:37]-[09:38] Marcos, Larissa) — fase futura.
- Rate limiting de saída por cliente/endpoint ([09:38]-[09:39] Diego, Larissa) — observar e decidir depois.
- Dashboard visual para o cliente acompanhar webhooks ([09:39]-[09:40] Larissa, Marcos) — projeto separado de frontend.
- Múltiplos workers em paralelo / particionamento por `order_id` ([09:12]-[09:13] Diego, Bruno) — problema futuro de escala.
- Arquivamento de eventos entregues após 30 dias ([09:08] Diego) — fora do escopo desta feature.

## Fluxos detalhados

### 1. Criação do evento na outbox

Dentro de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), após o `tx.order.update` e o `tx.orderStatusHistory.create` e **antes** do commit da transação:

1. Buscar `webhook_endpoints` ativos do `customerId` do pedido cujo filtro de eventos inclua o `toStatus`.
2. Se não houver nenhum endpoint interessado, não inserir nada na outbox (otimização decidida em [09:33]-[09:34] Bruno, Diego).
3. Se houver, renderizar o payload (snapshot, ver seção Contratos) e inserir uma linha por endpoint interessado em `webhook_outbox`, com `id` (UUID), `event_id` (UUID), `webhook_endpoint_id`, `order_id`, `payload` (JSON), `status = 'PENDING'`, `attempt_count = 0`, `created_at`.
4. Se a inserção falhar, propagar o erro — a transação inteira (incluindo a mudança de status) sofre rollback ([09:40]-[09:41] Bruno, Diego).

```ts
// src/modules/webhooks/webhook.events.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: OrderWithRelations,
  fromStatus: OrderStatus | null,
  toStatus: OrderStatus,
): Promise<void> {
  const endpoints = await tx.webhookEndpoint.findMany({
    where: { customerId: order.customerId, active: true, eventTypes: { has: toStatus } },
  });
  if (endpoints.length === 0) return;

  const payload = renderOrderStatusChangedPayload(order, fromStatus, toStatus);
  await tx.webhookOutboxEvent.createMany({
    data: endpoints.map((endpoint) => ({
      id: randomUUID(),
      eventId: randomUUID(),
      webhookEndpointId: endpoint.id,
      orderId: order.id,
      payload,
      status: 'PENDING',
      attemptCount: 0,
    })),
  });
}
```

Chamada em `OrderService.changeStatus`, logo após o `tx.orderStatusHistory.create`:

```ts
await publishWebhookEvent(tx, refreshed!, from, to);
```

### 2. Processamento pelo worker

Loop principal de `src/worker.ts`, delegando a lógica para `src/modules/webhooks/webhook.processor.ts`:

1. A cada 2 segundos, buscar até N (ex.: 20) linhas de `webhook_outbox` com `status = 'PENDING'` e `next_attempt_at <= now()`, ordenadas por `created_at` ASC.
2. Para cada evento, carregar o `webhook_endpoint` associado (url, secret ativa, secret anterior se em grace period).
3. Montar os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`).
4. Validar tamanho do payload serializado; se > 64KB, marcar o evento como falho permanentemente sem tentar HTTP (ver matriz de erros, `WEBHOOK_PAYLOAD_TOO_LARGE`) e mover direto para DLQ ([09:23]-[09:24] Sofia, Diego, Larissa).
5. Disparar `POST` para a `url` do endpoint com timeout de 10 segundos ([09:42] Diego, Sofia).
6. Em sucesso (2xx): marcar `status = 'DELIVERED'`, registrar em histórico de deliveries (ver Contratos) com status code, tempo de resposta e corpo da resposta (truncado).
7. Em falha (timeout, erro de rede, status >= 400): incrementar `attempt_count`, registrar a tentativa no histórico de deliveries, calcular `next_attempt_at` conforme a tabela de backoff (ver Retry) e manter `status = 'PENDING'` — ou mover para DLQ se `attempt_count` atingiu 5.

### 3. Retry

Tabela de backoff por `attempt_count` (decisão em [ADR-003](adrs/ADR-003-retry-backoff-exponencial-dlq.md)):

| Tentativa | Delay até a próxima |
| --- | --- |
| 1 → 2 | 1 minuto |
| 2 → 3 | 5 minutos |
| 3 → 4 | 30 minutos |
| 4 → 5 | 2 horas |
| 5 → DLQ | 12 horas (última janela antes de desistir) |

Total: 5 tentativas, ~15 horas entre a primeira falha e a desistência.

### 4. DLQ

Ao esgotar as 5 tentativas (ou em caso de payload > 64KB), o worker:

1. Insere uma linha em `webhook_dead_letter` com `webhook_outbox_event_id`, `payload`, `failure_reason`, `attempt_count`, `moved_to_dlq_at`.
2. Marca o evento original em `webhook_outbox` como `status = 'DEAD'` (mantido para rastreabilidade, não deletado).

Reprocessamento manual via `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`): recria o evento em `webhook_outbox` com `status = 'PENDING'`, `attempt_count = 0`, e loga o `userId` do operador que disparou o replay para auditoria ([09:36] Sofia).

## Contratos públicos

Todos os endpoints usam `authenticate` (JWT) por padrão; o replay de DLQ usa também `requireRole('ADMIN')`.

### `POST /customers/:customerId/webhooks`

Cadastra um endpoint de webhook para um customer.

Request:
```json
{
  "url": "https://atlas.example.com/webhooks/oms",
  "eventTypes": ["SHIPPED", "DELIVERED"]
}
```

Response `201`:
```json
{
  "id": "6e1a9e2a-...-c1d4",
  "customerId": "3fa1...",
  "url": "https://atlas.example.com/webhooks/oms",
  "eventTypes": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_7f2e...a91",
  "active": true,
  "createdAt": "2026-10-09T09:30:00.000Z"
}
```
`400 WEBHOOK_INVALID_URL` se a URL não for `https://`.

### `GET /customers/:customerId/webhooks`

Lista os webhooks cadastrados do customer.

Response `200`:
```json
{
  "items": [
    { "id": "6e1a9e2a-...-c1d4", "url": "https://atlas.example.com/webhooks/oms", "eventTypes": ["SHIPPED", "DELIVERED"], "active": true }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

### `PATCH /webhooks/:id`

Atualiza `url` e/ou `eventTypes` de um webhook existente.

Request:
```json
{ "eventTypes": ["PAID", "SHIPPED", "DELIVERED"] }
```
Response `200`: objeto do webhook atualizado. `404 WEBHOOK_NOT_FOUND` se o id não existir.

### `DELETE /webhooks/:id`

Remove o cadastro de um webhook. Response `204`, sem corpo.

### `POST /webhooks/:id/rotate-secret`

Gera uma nova secret, mantendo a anterior válida por 24h (grace period).

Response `200`:
```json
{
  "id": "6e1a9e2a-...-c1d4",
  "secret": "whsec_9b0c...44f",
  "previousSecretValidUntil": "2026-10-10T09:30:00.000Z"
}
```

### `GET /webhooks/:id/deliveries`

Histórico das últimas 100 tentativas de entrega ([09:34] Marcos).

Response `200`:
```json
{
  "items": [
    {
      "id": "d41e...",
      "eventId": "a9c0...",
      "status": "DELIVERED",
      "attemptCount": 1,
      "responseStatusCode": 200,
      "responseTimeMs": 184,
      "deliveredAt": "2026-10-09T10:00:02.000Z"
    },
    {
      "id": "d41f...",
      "eventId": "a9c1...",
      "status": "FAILED",
      "attemptCount": 3,
      "responseStatusCode": 503,
      "responseTimeMs": 10001,
      "nextAttemptAt": "2026-10-09T10:30:02.000Z"
    }
  ],
  "page": 1,
  "pageSize": 100,
  "total": 2
}
```

### `POST /admin/webhooks/dead-letter/:id/replay`

Reprocessa manualmente um evento em DLQ. Requer `role = ADMIN` ([09:35]-[09:36] Sofia, Larissa). Response `200`:
```json
{ "outboxEventId": "f3a2...", "status": "PENDING", "requeuedBy": "user-id-do-admin" }
```
`404 WEBHOOK_DEAD_LETTER_NOT_FOUND` se o id não existir em `webhook_dead_letter`.

## Matriz de erros previstos

| Código | HTTP Status | Descrição |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook endpoint não encontrado por id |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada não é `https://` ([09:23] Sofia) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação exige secret presente/válida e ela está ausente |
| `WEBHOOK_INVALID_EVENT_TYPE` | 400 | `eventTypes` contém um status que não existe em `OrderStatus` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB; evento vai direto para DLQ sem tentativa HTTP ([09:23]-[09:24] Sofia, Diego) |
| `WEBHOOK_SIGNATURE_GENERATION_FAILED` | 500 | Falha interna ao calcular HMAC (secret corrompida/ausente no momento do envio) |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno, não HTTP de resposta ao cliente da nossa API) | Timeout de 10s na chamada ao endpoint do cliente ([09:42] Diego) — registrado no histórico de deliveries, não retornado via API síncrona |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Id de DLQ inexistente no replay |
| `WEBHOOK_FORBIDDEN_REPLAY` | 403 | Replay de DLQ chamado sem role `ADMIN` (via `requireRole`) |

## Estratégias de resiliência

- **Timeout:** 10 segundos por tentativa de entrega HTTP ([09:42] Diego, Sofia).
- **Retry:** backoff exponencial 1m/5m/30m/2h/12h, 5 tentativas (ver [ADR-003](adrs/ADR-003-retry-backoff-exponencial-dlq.md)).
- **Fallback:** esgotado o retry, DLQ com reprocessamento manual via endpoint admin — não há fallback automático (ex.: e-mail) nesta fase.
- **Transação atômica:** inserção na outbox é parte da mesma transação de `changeStatus`; falha na outbox reverte a mudança de status ([09:40]-[09:41] Bruno, Diego).
- **Isolamento de processo:** worker roda separado da API, com `PrismaClient` próprio, para não ser afetado por restarts da API (ver [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)).

## Observabilidade

- **Métricas** (via logs estruturados Pino, agregáveis por quem consome o log): contagem de eventos publicados na outbox por status de pedido; contagem de entregas bem-sucedidas vs. falhas por tentativa; tamanho da fila `PENDING` na outbox (para alertar acúmulo); contagem de eventos movidos para DLQ por período.
- **Logs:** reaproveita o logger Pino já existente (`src/shared/logger/index.ts`). Cada tentativa de entrega loga `event_id`, `webhook_endpoint_id`, `order_id`, `attempt_count`, resultado e tempo de resposta, seguindo os mesmos campos (`service`, `env`) já definidos no logger base.
- **Tracing:** o `requestId` já propagado pelo `request-logger.middleware.ts` da API é correlacionado ao `event_id` da outbox nos logs de criação do evento, permitindo rastrear de qual requisição HTTP (mudança de status) um evento de webhook se originou.

## Dependências e compatibilidade

- Depende de `@prisma/client`, `uuid` e `zod`, já presentes no `package.json` — nenhuma dependência nova de runtime é necessária para HMAC (módulo `crypto` nativo do Node cobre HMAC-SHA256).
- Depende de uma migration Prisma adicionando `webhook_endpoints`, `webhook_outbox_events` (ou `webhook_outbox`), `webhook_deliveries` (histórico) e `webhook_dead_letter`.
- Novo script `npm run worker` em `package.json`, análogo a `npm run dev` / `start`, apontando para `src/worker.ts`.
- Compatível com o schema de autenticação JWT existente (`src/middlewares/auth.middleware.ts`); não requer mudança no formato do token.

## Integração com o sistema existente

- **`src/modules/orders/order.service.ts` (método `changeStatus`):** recebe a chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação já existente, imediatamente após `tx.orderStatusHistory.create`. Nenhuma outra parte do método muda; a função de publicação é pura em relação ao `tx` recebido ([09:41] Bruno, Diego).
- **`src/shared/errors/http-errors.ts`:** novas classes de erro do módulo (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError` da mesma forma que `InsufficientStockError` e `InvalidStatusTransitionError`, com códigos prefixados `WEBHOOK_` (ver matriz de erros acima e [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)).
- **`src/middlewares/error.middleware.ts`:** nenhuma alteração necessária — o middleware já serializa qualquer subclasse de `AppError` para a resposta HTTP padrão (`{ error: { code, message, details } }`), cobrindo os novos erros `WEBHOOK_*` automaticamente.
- **`src/middlewares/auth.middleware.ts`:** o endpoint `POST /admin/webhooks/dead-letter/:id/replay` usa `requireRole('ADMIN')` da mesma forma que outros endpoints administrativos protegidos, sem alteração no middleware.
- **`src/routes/index.ts`:** novo `buildWebhookRouter(controller)` é registrado em `buildApiRouter`, seguindo exatamente o padrão de `buildOrderRouter` / `buildCustomerRouter` já presentes.
- **`src/shared/logger/index.ts`:** o worker (`src/worker.ts`) importa a mesma instância/factory de logger Pino usada pela API, mantendo o mesmo formato de log (`base: { service, env }`) para consistência de observabilidade entre API e worker.
- **`src/config/database.ts`:** o worker não reutiliza a instância `prisma` exportada por este arquivo (ela é inicializada no contexto do processo da API); cria sua própria instância de `PrismaClient` com a mesma `DATABASE_URL`, seguindo a decisão de [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md).

## Critérios de aceite técnicos

- Uma mudança de status que falha ao inserir na outbox resulta em rollback de toda a transação (status não muda).
- Um evento publicado é entregue a todos os endpoints ativos do customer que escutam aquele `toStatus`, e a nenhum outro.
- Uma entrega falha é retentada seguindo exatamente a progressão 1m/5m/30m/2h/12h, e movida para DLQ após a 5ª falha.
- Toda chamada HTTP de entrega inclui `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
- A assinatura HMAC é verificável com a secret atual e, durante o grace period de 24h após rotação, também com a secret anterior.
- O endpoint de replay de DLQ só é acessível por usuários com `role = ADMIN` e registra o `userId` de quem executou o replay.

## Riscos e mitigação

- **Risco:** acoplamento incorreto da publicação do evento na transação de `changeStatus` degrade a performance da transação principal. **Mitigação:** manter `publishWebhookEvent` como operações de insert simples e indexadas, sem chamadas de rede dentro da transação.
- **Risco:** acúmulo de eventos `PENDING` na outbox sob picos de falha de clientes. **Mitigação:** índice em `(status, created_at)`, lotes pequenos de leitura; rate limiting de saída fica como questão em aberto no RFC caso o acúmulo se torne um problema real.
- **Risco:** vazamento de secret HMAC. **Mitigação:** secret por endpoint (não global) e rotação com grace period de 24h (ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)); revisão de segurança dedicada antes do deploy ([09:46] Sofia).

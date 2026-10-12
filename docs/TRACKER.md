# Tracker de Rastreabilidade

Mapeia cada item relevante registrado nos documentos à sua origem na `TRANSCRICAO.md` ou no código fonte.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição | Três clientes B2B pedem notificação em tempo real; Atlas ameaça churn | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (url + eventTypes) por customer | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada automaticamente na criação, devolvida na resposta | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | PATCH para editar webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | DELETE para remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | GET para listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Filtro de eventos: inserir na outbox só se endpoint escuta o status | TRANSCRICAO | [09:33]-[09:34] Marcos, Bruno, Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Não inserir na outbox se nenhum endpoint estiver interessado | TRANSCRICAO | [09:34] Bruno, Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Timestamp, X-Webhook-Id em toda entrega | TRANSCRICAO | [09:25], [09:44]-[09:45] Diego, Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry automático com backoff até 5 tentativas, depois DLQ | TRANSCRICAO | [09:14]-[09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a role ADMIN, com auditoria | TRANSCRICAO | [09:18]-[09:19], [09:35]-[09:36] Diego, Sofia, Larissa |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Rejeitar URL não-HTTPS no cadastro/edição | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de entrega HTTP | TRANSCRICAO | [09:42] Diego, Sofia |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, erro se exceder | TRANSCRICAO | [09:23]-[09:24] Sofia, Diego, Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Worker como processo separado da API | TRANSCRICAO | [09:11] Diego, Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Inserção na outbox atômica com a transação de status | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once, cliente dedup via X-Event-Id | TRANSCRICAO | [09:24]-[09:25] Diego |
| PRD-MET-01 | docs/PRD.md | Métrica | Meta de latência de entrega abaixo de 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-ESCOPO-01 | docs/PRD.md | Trade-off | Fora de escopo: notificação proativa (e-mail) ao cliente | TRANSCRICAO | [09:37]-[09:38] Marcos, Larissa |
| PRD-ESCOPO-02 | docs/PRD.md | Trade-off | Fora de escopo: dashboard visual para o cliente | TRANSCRICAO | [09:39]-[09:40] Larissa, Marcos |
| PRD-ESCOPO-03 | docs/PRD.md | Trade-off | Fora de escopo: rate limiting de saída (observar depois) | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| PRD-ESCOPO-04 | docs/PRD.md | Trade-off | Fora de escopo: múltiplos workers em paralelo / ordering global | TRANSCRICAO | [09:12]-[09:13] Diego, Bruno |
| PRD-ESCOPO-05 | docs/PRD.md | Trade-off | Fora de escopo: arquivamento de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-01 | docs/PRD.md | Risco | Acúmulo de eventos pendentes na outbox | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| PRD-RISCO-02 | docs/PRD.md | Risco | Vazamento de secret comprometendo autenticidade de entregas | TRANSCRICAO | [09:21]-[09:22], [09:46] Sofia, Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Cronograma de 3 sprints pode não cobrir achados tardios de segurança | TRANSCRICAO | [09:46] Sofia, Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Middleware de autenticação JWT e requireRole já existentes | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Logger Pino e middleware de erro centralizado já existentes | CODIGO | src/shared/logger/index.ts |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Alternativa descartada: fila dedicada (Redis Streams) por custo de infra | TRANSCRICAO | [09:06]-[09:07] Larissa, Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Alternativa descartada: disparo síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Alternativa descartada: retry com 3 tentativas | TRANSCRICAO | [09:16] Bruno, Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Alternativa descartada: secret global de HMAC | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Questão em aberto: rate limiting de saída por cliente | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Questão em aberto: notificação proativa de falha ao cliente | TRANSCRICAO | [09:37]-[09:38] Marcos, Larissa |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Questão em aberto: escalabilidade do worker para múltiplas instâncias | TRANSCRICAO | [09:12]-[09:13] Diego, Bruno |
| RFC-OPEN-04 | docs/RFC.md | Restrição | Questão em aberto: dashboard visual para o cliente | TRANSCRICAO | [09:39]-[09:40] Larissa, Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Alteração de OrderService.changeStatus afeta fluxo central de pedidos | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Inserção na outbox dentro da transação de changeStatus, com rollback em falha | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Worker em polling de 2s, lotes ordenados por created_at | TRANSCRICAO | [09:08]-[09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Payload do evento é snapshot renderizado na inserção, não no envio | TRANSCRICAO | [09:51]-[09:52] Larissa, Diego, Bruno |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Decisão | Id dos registros da outbox em UUID, seguindo padrão do projeto | TRANSCRICAO | [09:51] Larissa, Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Decisão | Endpoint POST de cadastro de webhook | TRANSCRICAO | [09:31]-[09:32] Marcos, Bruno |
| FDD-CONTRATO-02 | docs/FDD.md | Decisão | Endpoint GET de listagem de webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Decisão | Endpoint PATCH de edição de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Decisão | Endpoint DELETE de remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Decisão | Endpoint de rotação de secret | TRANSCRICAO | [09:21]-[09:22] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Decisão | Endpoint GET de histórico de deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Decisão | Endpoint POST de replay de DLQ, formato /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18]-[09:19] Diego, Larissa |
| FDD-ERRO-01 | docs/FDD.md | Decisão | Prefixo WEBHOOK_ para códigos de erro do módulo | TRANSCRICAO | [09:28]-[09:29] Bruno, Larissa |
| FDD-ERRO-02 | docs/FDD.md | Restrição | Payload > 64KB vira erro (não truncamento) | TRANSCRICAO | [09:23]-[09:24] Sofia, Diego |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Timeout de 10s por tentativa HTTP do worker | TRANSCRICAO | [09:42] Diego, Sofia |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:15]-[09:17] Diego, Larissa |
| FDD-OBS-01 | docs/FDD.md | Decisão | Reuso do logger Pino existente no worker | TRANSCRICAO | [09:29] Bruno |
| FDD-INTEG-01 | docs/FDD.md | Decisão | changeStatus chama publishWebhookEvent(tx, order, from, to) | TRANSCRICAO | [09:41] Bruno, Diego |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Erros do módulo estendem AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Padrão de erro específico (ex. InvalidStatusTransitionError) usado como referência | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | Middleware de erro centralizado já trata AppError, Zod e Prisma sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | requireRole usado para proteger o replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Novo router de webhooks registrado como os demais módulos | CODIGO | src/routes/index.ts |
| FDD-INTEG-07 | docs/FDD.md | Restrição | changeStatus hoje já faz update de status + history + estoque numa transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição | Transições de status válidas controladas por canTransition | CODIGO | src/modules/orders/order.status.ts |
| FDD-INTEG-09 | docs/FDD.md | Restrição | Padrão de módulo (controller/service/repository/routes/schemas) a seguir | CODIGO | src/modules/customers/customer.routes.ts |
| FDD-INTEG-10 | docs/FDD.md | Restrição | Worker usa PrismaClient próprio, não a instância da API | CODIGO | src/config/database.ts |
| FDD-INTEG-11 | docs/FDD.md | Restrição | authenticate/JWT já usado por todos os módulos autenticados | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-pattern-mysql.md | Decisão | Padrão Outbox no MySQL para publicação de eventos | TRANSCRICAO | [09:06]-[09:08] Diego, Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-pattern-mysql.md | Trade-off | Alternativa: Redis Streams descartada por overengineering | TRANSCRICAO | [09:07] Larissa, Diego |
| ADR-002-DEC | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker roda como processo separado, polling de 2s | TRANSCRICAO | [09:09]-[09:11] Diego, Larissa |
| ADR-002-ALT | docs/adrs/ADR-002-worker-processo-separado-polling.md | Trade-off | Alternativa: trigger de banco descartada (sem notificação de processo externo) | TRANSCRICAO | [09:09] Diego |
| ADR-003-DEC | docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md | Decisão | Retry 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:14]-[09:18] Diego |
| ADR-003-ALT | docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md | Trade-off | Alternativa: 3 tentativas descartada por insuficiente | TRANSCRICAO | [09:16] Bruno, Diego |
| ADR-004-DEC | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:19]-[09:22] Sofia |
| ADR-004-ALT | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Alternativa: secret global descartada por blast radius | TRANSCRICAO | [09:21] Sofia |
| ADR-005-DEC | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:24]-[09:26] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-at-least-once-x-event-id.md | Trade-off | Alternativa: exactly-once descartada por complexidade | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo de webhooks segue padrões existentes do projeto | TRANSCRICAO | [09:27]-[09:30] Bruno, Diego, Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Referência à estrutura de módulo existente | CODIGO | src/modules/orders/ |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Referência à classe AppError e subclasses | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Referência ao logger Pino | CODIGO | src/shared/logger/index.ts |

# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS vai passar a notificar clientes B2B automaticamente, via webhooks outbound, sempre que o status de um pedido deles mudar. Hoje essa notificação não existe: clientes só sabem de mudanças fazendo polling manual em `GET /orders`. A feature adiciona um módulo de configuração de webhooks (`src/modules/webhooks/`) e um worker assíncrono que entrega eventos de mudança de status com garantias de entrega, segurança e resiliência.

## Problema e motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente ser notificados em tempo real quando o status de seus pedidos muda, em vez de fazer polling periódico em `GET /orders`, que é lento e caro de manter do lado deles ([09:00] Marcos). A Atlas Comercial sinalizou risco de migrar para um concorrente se a feature não for entregue até o fim do trimestre ([09:00] Marcos).

## Público-alvo e cenários de uso

- **Clientes B2B integrados via API** (operadores que representam o cliente, autenticados com JWT do OMS) que cadastram um ou mais endpoints de webhook para receber notificações de mudança de status de pedidos específicos do seu `customerId` ([09:31]-[09:32] Marcos, Bruno).
- **Administradores do sistema** (`role = ADMIN`) que reprocessam manualmente entregas que esgotaram as tentativas automáticas (DLQ) ([09:35]-[09:36] Sofia, Larissa).

Cenário principal: um pedido da Atlas muda de `PROCESSING` para `SHIPPED`; o endpoint de webhook cadastrado pela Atlas para esse evento recebe uma notificação HTTP assinada em até ~10 segundos, sem a Atlas precisar fazer polling.

## Objetivos e métricas de sucesso

- Reduzir a necessidade de polling de `GET /orders` pelos clientes B2B integrados a webhooks.
- **Métrica e meta quantitativa:** latência de entrega (do commit da mudança de status até a tentativa de entrega HTTP) abaixo de **10 segundos** no caminho feliz, atendendo ao requisito definido pelos clientes como "tempo real" ([09:02] Marcos). Essa meta é atendida pelo polling de 2s do worker somado ao tempo de processamento, com margem confortável.
- Garantir que nenhuma mudança de status gere evento perdido: todo evento publicado na outbox é entregue pelo menos uma vez (at-least-once) ou termina rastreável em DLQ.
- Entregar a feature completa (incluindo revisão de segurança) em até 3 sprints, conforme estimativa da Tech Lead ([09:45]-[09:46] Larissa).

## Escopo

### Incluso
- Cadastro, edição, remoção e listagem de endpoints de webhook por cliente (`customerId`), com filtro de quais status o endpoint quer receber ([09:31]-[09:33] Marcos, Bruno).
- Geração de secret na criação do endpoint; rotação de secret com grace period de 24h ([09:21]-[09:22] Sofia).
- Publicação transacional do evento (padrão outbox) dentro de `OrderService.changeStatus` ([09:40]-[09:41] Bruno, Diego).
- Worker assíncrono de entrega, com retry/backoff e DLQ ([09:14]-[09:18] Diego).
- Autenticação da entrega via HMAC-SHA256 (`X-Signature`) e idempotência via `X-Event-Id` ([09:19]-[09:25] Sofia, Diego).
- Histórico de deliveries (últimos 100 envios por webhook) consultável pelo cliente ([09:34] Marcos).
- Endpoint administrativo de replay manual de eventos em DLQ, restrito a `role = ADMIN` ([09:18]-[09:19], [09:35]-[09:36]).

### Fora de escopo
- **Notificação proativa ao cliente (ex.: e-mail) quando o webhook dele está falhando repetidamente.** Explicitamente descartado para esta fase pela Tech Lead; fica para uma fase futura, condicionado a medir o impacto ([09:37]-[09:38] Marcos, Larissa).
- **Dashboard visual** para o cliente acompanhar seus webhooks. Considerado projeto separado do time de frontend; esta feature expõe apenas endpoints de API ([09:39]-[09:40] Larissa, Marcos).
- **Rate limiting de saída** por cliente/endpoint. Levantado como preocupação (ex.: 50 pedidos mudando de status em um minuto), mas decidido observar em produção antes de implementar ([09:38]-[09:39] Diego, Larissa).
- **Escalar o worker para múltiplas instâncias em paralelo** (e a garantia de ordering associada). Fica como limitação conhecida para esta entrega ([09:12]-[09:13] Diego, Bruno).
- **Arquivamento de eventos entregues** na outbox após 30 dias. Mencionado como necessidade futura, não desta feature ([09:08] Diego).

## Requisitos funcionais

1. Cliente autenticado pode cadastrar um webhook informando `url` (HTTPS) e lista de status de interesse (`eventTypes`), vinculado a um `customerId` ([09:31]-[09:33] Marcos, Bruno).
2. O sistema gera a secret do webhook automaticamente na criação e a devolve na resposta ([09:31] Marcos).
3. Cliente autenticado pode editar (`PATCH`) a `url` e/ou `eventTypes` de um webhook existente ([09:33] Bruno).
4. Cliente autenticado pode remover (`DELETE`) um webhook ([09:33] Bruno).
5. Cliente autenticado pode listar (`GET`) os webhooks cadastrados de um `customerId` ([09:33] Bruno).
6. Cliente autenticado pode solicitar rotação de secret de um webhook, com a secret anterior permanecendo válida por 24h ([09:21]-[09:22] Sofia).
7. O sistema insere o evento de mudança de status na outbox somente para endpoints cujo `eventTypes` inclua o `toStatus` da transição ([09:33]-[09:34] Marcos, Bruno, Diego).
8. O sistema não insere evento na outbox se nenhum endpoint do customer estiver interessado naquele status, para economizar linhas na tabela ([09:34] Bruno, Diego).
9. O sistema assina toda chamada de entrega com HMAC-SHA256 sobre o corpo da requisição, enviado no header `X-Signature` ([09:20] Sofia).
10. O sistema envia os headers `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id` em toda tentativa de entrega ([09:25], [09:44]-[09:45] Diego, Sofia).
11. O sistema reentrega automaticamente em caso de falha, seguindo backoff de 1m/5m/30m/2h/12h, até 5 tentativas, movendo para DLQ ao esgotar ([09:14]-[09:18] Diego).
12. Administrador (`role = ADMIN`) pode reprocessar manualmente um evento em DLQ via endpoint dedicado, com o replay registrado para auditoria ([09:18]-[09:19], [09:35]-[09:36] Diego, Sofia, Larissa).
13. Cliente autenticado pode consultar o histórico das últimas 100 entregas de um webhook, incluindo status, payload, resposta e tempo de resposta ([09:34] Marcos).
14. O sistema recusa o cadastro ou edição de um webhook com `url` que não seja HTTPS, retornando erro de validação ([09:23] Sofia).

## Requisitos não funcionais

- Timeout de 10 segundos por tentativa de entrega HTTP a um endpoint de cliente ([09:42] Diego, Sofia).
- Limite de 64KB para o payload de um evento; eventos que excedam o limite não são enviados e vão direto para falha/DLQ ([09:23]-[09:24] Sofia, Diego, Larissa).
- Latência mínima de entrega determinada pelo polling do worker: até 2 segundos no pior caso dentro do ciclo de leitura ([09:09]-[09:10] Diego, Marcos, Larissa).
- O worker deve rodar como processo separado da API, para não ser afetado por reinícios de deploy da API ([09:11] Diego, Larissa).
- A inserção do evento na outbox deve ser atômica com a transação de mudança de status: falha na outbox implica rollback de toda a mudança de status ([09:40]-[09:41] Bruno, Diego).
- Entrega garantida como at-least-once, nunca exactly-once; o cliente deve ser capaz de deduplicar via `X-Event-Id` ([09:24]-[09:25] Diego).

## Decisões e trade-offs principais

- Padrão outbox no MySQL em vez de fila dedicada (Redis Streams): menor custo operacional, aceitando não ter infraestrutura de mensageria dedicada ([09:06]-[09:07] Larissa, Diego — ver `docs/adrs/ADR-001-outbox-pattern-mysql.md`).
- At-least-once com dedup no cliente em vez de exactly-once: menor complexidade de implementação, transferindo responsabilidade de deduplicação para o cliente ([09:24]-[09:25] Diego — ver `docs/adrs/ADR-005-at-least-once-x-event-id.md`).
- Secret por endpoint (não global) com rotação e grace period de 24h: maior segurança operacional, ao custo de mais entidades para gerenciar ([09:21]-[09:22] Sofia — ver `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md`).
- 5 tentativas de retry (não 3, não indefinido): equilíbrio entre tolerância a indisponibilidade real do cliente e não deixar eventos presos para sempre ([09:15]-[09:17] Diego, Bruno, Larissa — ver `docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md`).

## Dependências

- Banco de dados MySQL já existente, via Prisma — novas tabelas (`webhook_endpoints`, outbox, deliveries, dead-letter) via migration.
- Middleware de autenticação JWT e `requireRole` já existentes (`src/middlewares/auth.middleware.ts`).
- Logger Pino e middleware de erro centralizado já existentes (`src/shared/logger/index.ts`, `src/middlewares/error.middleware.ts`).
- Disponibilidade da Engenheira de Segurança para revisão dedicada do fluxo de HMAC e geração de secret antes do deploy, com pelo menos 2 dias úteis reservados ([09:46] Sofia).

## Riscos e mitigação

- **Risco:** acúmulo de eventos pendentes na outbox se muitos clientes mudarem de status em rajada ou se um cliente externo ficar indisponível por muito tempo. **Probabilidade:** média. **Impacto:** médio (atraso de entrega, crescimento de tabela). **Mitigação:** índices em `status`/`created_at`, lotes pequenos no worker; rate limiting de saída fica registrado como questão em aberto a avaliar se o problema se concretizar ([09:38]-[09:39] Diego, Larissa).
- **Risco:** vazamento de secret de um cliente comprometendo a autenticidade das entregas para aquele endpoint. **Probabilidade:** baixa, mas já ocorreu antes com um cliente ([09:22] Diego). **Impacto:** alto (entregas forjadas ou integração interrompida). **Mitigação:** secret por endpoint (isola o blast radius) e rotação com grace period de 24h, revisão de segurança dedicada antes do deploy ([09:21]-[09:22], [09:46] Sofia).
- **Risco:** cronograma de 3 sprints não ser suficiente se a revisão de segurança identificar problemas tarde. **Probabilidade:** baixa. **Impacto:** médio (atraso de entrega para a Atlas). **Mitigação:** revisão de segurança reservada com antecedência (2 dias úteis) no fim do ciclo, não como etapa surpresa ([09:46] Sofia, Larissa).

## Critérios de aceitação

- Cliente consegue cadastrar, editar, remover e listar webhooks via API autenticada.
- Mudança de status de um pedido gera entrega HTTP assinada (HMAC-SHA256) para todo endpoint ativo e interessado naquele status, em até ~10 segundos no caminho feliz.
- Falha de entrega é reentregue automaticamente seguindo o backoff definido, e eventos que esgotam as tentativas ficam disponíveis em DLQ para replay manual por `role = ADMIN`.
- Cliente consegue consultar o histórico das últimas 100 entregas de um webhook.
- Nenhum evento é publicado para endpoints que não listam aquele status em `eventTypes`.

## Estratégia de testes e validação

- Testes de integração do fluxo `changeStatus` garantindo que a inserção na outbox é atômica com a mudança de status (incluindo cenário de falha forçada na outbox revertendo o status).
- Testes do worker simulando endpoints lentos/indisponíveis para validar a progressão exata de backoff e a transição para DLQ após 5 tentativas.
- Testes de verificação de assinatura HMAC, incluindo o cenário de grace period com secret antiga ainda válida por 24h após rotação.
- Revisão de segurança dedicada da Engenheira de Segurança sobre geração de secret e fluxo HMAC antes do deploy, conforme solicitado ([09:46] Sofia).
- Testes de contrato dos endpoints HTTP do módulo de webhooks (CRUD, deliveries, replay de DLQ), incluindo controle de acesso por role no replay.

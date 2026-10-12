# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Proposto — aguardando fechamento de implementação |
| Data | 2026-10 (reunião técnica de 09:00) |
| Revisores | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |

## Resumo executivo (TL;DR)

Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) hoje fazem polling manual em `GET /orders` para saber quando o status de um pedido muda, o que é lento e caro para eles — a Atlas já sinalizou risco de churn se isso não for resolvido até o fim do trimestre ([00:00-01:00] Marcos). Propomos um sistema de **webhooks outbound**: quando um pedido muda de status, o OMS registra o evento de forma transacional (padrão **Outbox** em MySQL) e um **worker separado**, em polling de 2 segundos, entrega o evento via HTTP ao(s) endpoint(s) cadastrados pelo cliente, com **autenticação HMAC-SHA256**, **retry com backoff exponencial** e **DLQ** para falhas permanentes. A entrega é **at-least-once**, com deduplicação do lado do cliente via `X-Event-Id`. Estimativa: 3 sprints, incluindo revisão de segurança.

## Contexto e problema

O OMS (Node.js + TypeScript, MySQL via Prisma) não possui hoje nenhum mecanismo de notificação externa, eventos ou webhooks — a aplicação é inteiramente request/response. Clientes B2B que integram com o sistema de pedidos só conseguem saber se o status de um pedido mudou fazendo polling periódico em `GET /orders`, o que:

- Gera custo de integração para os clientes (chamadas repetidas, muitas sem novidade).
- Introduz latência de percepção da mudança de status proporcional ao intervalo de polling do cliente.
- Já gerou reclamação formal de três clientes B2B, com risco de perda de contrato (Atlas Comercial) ([09:00] Marcos).

O requisito de latência aceito pelos clientes é "abaixo de 10 segundos" como "tempo real" — não é necessário push instantâneo ([09:02] Marcos).

## Proposta técnica

A arquitetura proposta tem três componentes:

1. **Publicação transacional (Outbox).** Dentro da mesma transação SQL do método `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), ao lado do update de `orders` e do insert em `order_status_history`, inserimos uma linha em uma nova tabela `webhook_outbox` com o evento já renderizado (snapshot do estado do pedido no momento da mudança). Se a transação falhar, o evento nunca existiu — eliminando qualquer janela de inconsistência entre "status mudou" e "evento foi gerado". Detalhe de decisão em [ADR-001 — Padrão Outbox no MySQL](../adrs/ADR-001-outbox-pattern-mysql.md).

2. **Entrega assíncrona (Worker).** Um processo Node.js separado (`src/worker.ts`), com sua própria instância de `PrismaClient`, faz polling na `webhook_outbox` a cada 2 segundos, busca eventos pendentes mais antigos, resolve os endpoints de webhook cadastrados pelo customer do pedido (filtrados por status de interesse), assina o payload com HMAC-SHA256 e envia a entrega HTTP. Decisão de processo separado + polling em [ADR-002 — Worker em processo separado, polling](../adrs/ADR-002-worker-processo-separado-polling.md).

3. **Resiliência e segurança da entrega.** Falhas de entrega (timeout, erro HTTP, indisponibilidade) acionam retry com backoff exponencial (1m/5m/30m/2h/12h, 5 tentativas); esgotado o retry, o evento vai para uma tabela `webhook_dead_letter`, reprocessável manualmente via endpoint admin. Cada endpoint de webhook tem secret própria para HMAC-SHA256, rotacionável com grace period de 24h. A entrega é at-least-once; o cliente dedupica usando o header `X-Event-Id`. Decisões em [ADR-003 — Retry com backoff exponencial e DLQ](../adrs/ADR-003-retry-backoff-exponencial-dlq.md), [ADR-004 — HMAC-SHA256 com secret por endpoint](../adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) e [ADR-005 — At-least-once com X-Event-Id](../adrs/ADR-005-at-least-once-x-event-id.md).

O módulo segue os padrões já existentes no OMS — estrutura de módulo, `AppError`, logger Pino, middleware de erro centralizado, `requireRole` — detalhado em [ADR-006 — Reuso dos padrões existentes](../adrs/ADR-006-reuso-padroes-existentes.md) e no FDD (`docs/FDD.md`).

O desenho detalhado de contratos HTTP, fluxos passo a passo, matriz de erros e integração ponto a ponto com o código fica no FDD (`docs/FDD.md`), não neste documento.

## Alternativas consideradas

- **Fila/broker dedicado (ex.: Redis Streams) em vez de Outbox no MySQL.** Daria desacoplamento equivalente ou melhor em throughput, mas exigiria subir e operar nova infraestrutura. Para o tamanho atual do time e da carga, foi considerado overengineering frente ao MySQL já existente ([09:06]-[09:07] Larissa, Diego). *Trade-off do descarte:* perde-se escalabilidade de longo prazo em favor de menor custo operacional imediato.
- **Disparo síncrono do webhook dentro da transação de `changeStatus`.** Mais simples de implementar à primeira vista, mas acopla a latência e a disponibilidade de um sistema externo à transação crítica de mudança de status de pedido, sem possibilidade de rollback de um HTTP call já enviado ([09:04] Bruno). *Trade-off do descarte:* perde-se simplicidade de implementação em favor de isolamento de falhas entre domínio interno e consumidores externos.
- **Retry com apenas 3 tentativas.** Mais agressivo em liberar recursos, mas insuficiente para cobrir indisponibilidades reais já observadas em clientes (ex.: manutenção planejada de 2h), matando o evento em ~30 minutos ([09:16] Bruno, Diego). *Trade-off do descarte:* perde-se "liberação rápida" de eventos presos em favor de maior tolerância a indisponibilidade do cliente.
- **Secret global de HMAC compartilhada entre todos os clientes.** Mais simples de gerenciar, mas um único vazamento comprometeria a validação de todos os clientes simultaneamente ([09:21] Sofia). *Trade-off do descarte:* perde-se simplicidade operacional em favor de isolamento de blast radius por endpoint.

## Questões em aberto

- **Rate limiting de saída por cliente.** Se um cliente tiver 50 pedidos mudando de status em um minuto, o worker pode disparar 50 chamadas HTTP seguidas para o mesmo endpoint. O time decidiu não implementar isso agora e observar se vira problema em produção ([09:38]-[09:39] Diego, Larissa).
- **Notificação proativa de falha ao cliente (ex.: e-mail após falhas consecutivas).** Levantado pelo PM, explicitamente descartado para esta fase e adiado para uma fase futura, condicionado a medição de impacto ([09:37]-[09:38] Marcos, Larissa).
- **Escalabilidade do worker para múltiplas instâncias em paralelo.** Hoje a ordenação de eventos por `order_id` depende de haver apenas um worker. Se no futuro for necessário escalar horizontalmente, será preciso particionar por `order_id` ou usar lock pessimista — fora do escopo atual ([09:12]-[09:13] Diego, Bruno).
- **Painel visual (dashboard) para o cliente acompanhar seus webhooks.** Descartado para esta fase; considerado projeto separado do time de frontend, a feature atual expõe apenas endpoints de API ([09:39]-[09:40] Larissa, Marcos).

## Impacto e riscos

- **Impacto em código existente:** altera `OrderService.changeStatus` para inserir na outbox dentro da mesma transação — qualquer regressão nesse método afeta diretamente o fluxo central de pedidos. Mitigação: inserção via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação em curso, minimizando acoplamento ([09:41] Bruno, Diego).
- **Risco de volume na outbox:** sem arquivamento de eventos entregues, a tabela cresce indefinidamente. Arquivamento após ~30 dias foi mencionado mas fica fora do escopo desta entrega ([09:08] Diego).
- **Risco de segurança:** geração e rotação de secret HMAC exigem revisão dedicada da engenharia de segurança antes do deploy (pelo menos 2 dias úteis reservados) ([09:46] Sofia).
- **Risco de cronograma:** prazo de 3 sprints estimado pela Tech Lead, incluindo a revisão de segurança no fim ([09:45]-[09:46] Larissa).

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](../adrs/ADR-001-outbox-pattern-mysql.md) — Padrão Outbox no MySQL
- [ADR-002 — Worker em processo separado, polling](../adrs/ADR-002-worker-processo-separado-polling.md) — Worker em processo separado, polling de 2s
- [ADR-003 — Retry com backoff exponencial e DLQ](../adrs/ADR-003-retry-backoff-exponencial-dlq.md) — Retry com backoff exponencial e DLQ
- [ADR-004 — HMAC-SHA256 com secret por endpoint](../adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) — HMAC-SHA256 com secret por endpoint
- [ADR-005 — At-least-once com X-Event-Id](../adrs/ADR-005-at-least-once-x-event-id.md) — At-least-once com `X-Event-Id`
- [ADR-006 — Reuso dos padrões existentes](../adrs/ADR-006-reuso-padroes-existentes.md) — Reuso dos padrões existentes do projeto

# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito — 2026-10 (reunião técnica de 09:00, decisão em [09:06]-[09:08])

## Contexto

O OMS precisa notificar sistemas externos (clientes B2B) quando o status de um pedido muda. A mudança de status já acontece dentro de uma transação SQL que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos (`src/modules/orders/order.service.ts`, método `changeStatus`).

Disparar a notificação de forma síncrona dentro dessa transação foi descartado: um HTTP call para um cliente lento ou indisponível bloquearia a transação de mudança de status de pedidos não relacionados, e não há como fazer rollback de uma chamada HTTP já enviada ([09:04] Bruno, [09:04] Larissa).

Era necessário um mecanismo que garantisse que o evento só existe se a mudança de status foi efetivamente commitada, sem acoplar a disponibilidade do consumidor externo à disponibilidade da API.

## Decisão

Implementar o padrão **Transactional Outbox**: dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir uma linha em uma nova tabela `webhook_outbox` com o evento já renderizado (snapshot). Um worker separado lê essa tabela de forma assíncrona e dispara as chamadas HTTP.

Se a transação principal falhar e sofrer rollback, o evento nunca existiu. Se ela comitar, o evento está garantidamente registrado. Não há janela de inconsistência entre "status mudou" e "evento foi registrado" ([09:06] Diego).

A tabela usa índice em `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e em `created_at` para o worker buscar pendentes de forma eficiente ([09:08] Diego).

## Alternativas Consideradas

- **Fila dedicada (ex.: Redis Streams / RabbitMQ):** daria desacoplamento equivalente, mas exigiria subir e operar nova infraestrutura. Para um time pequeno, foi considerado overengineering frente ao MySQL já existente e já operado ([09:07] Larissa, [09:07] Diego).
- **Disparo síncrono no `OrderService.changeStatus`:** descartado por acoplar a latência/disponibilidade de clientes externos à transação crítica de domínio, sem possibilidade de rollback de efeito colateral HTTP ([09:04] Bruno).

## Consequências

**Positivas**
- Garantia forte de consistência entre mudança de status e geração do evento, sem transação distribuída.
- Reaproveita a infraestrutura de banco existente (MySQL via Prisma), sem novo componente de infraestrutura.
- Evento é um snapshot ([09:52] Larissa, [09:52] Diego), isolando o payload entregue de mudanças futuras no pedido.

**Negativas / Trade-offs**
- Introduz acoplamento de escrita: a inserção na outbox roda dentro da transação de `changeStatus`; se a outbox falhar ao inserir, toda a mudança de status sofre rollback ([09:40] Bruno, [09:41] Diego) — é o comportamento desejado, mas aumenta a superfície de falha da transação principal.
- A tabela cresce continuamente; arquivamento de linhas entregues após ~30 dias foi mencionado como necessário, mas ficou fora do escopo desta feature ([09:08] Diego).
- Exige um worker de leitura (ver [ADR-002](ADR-002-worker-processo-separado-polling.md)) — não é "fire and forget" como um disparo direto.

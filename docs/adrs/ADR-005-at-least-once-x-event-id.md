# ADR-005: Garantia at-least-once com idempotência via X-Event-Id

## Status

Aceito — 2026-10 (reunião técnica de 09:00, decisão em [09:24]-[09:26])

## Contexto

Com retries e um worker em polling (ver [ADR-002](ADR-002-worker-processo-separado-polling.md) e [ADR-003](ADR-003-retry-backoff-exponencial-dlq.md)), é possível que o mesmo evento seja entregue mais de uma vez ao cliente — por exemplo, se a entrega foi bem-sucedida mas a confirmação/registro de sucesso falhou antes de ser persistido. Era necessário decidir qual garantia de entrega oferecer e como o cliente consegue lidar com entregas duplicadas.

## Decisão

O sistema garante **at-least-once delivery** (pelo menos uma entrega), não exactly-once. Cada evento recebe um **UUID único** (`event_id`) no momento em que entra na outbox, enviado no header **`X-Event-Id`** em toda tentativa de entrega daquele evento. O cliente é responsável por **deduplicar do lado dele** usando esse `event_id` ([09:25] Diego).

Esse padrão foi explicitamente comparado a práticas de mercado (Stripe, GitHub), que adotam a mesma abordagem ([09:25] Diego).

## Alternativas Consideradas

- **Exactly-once delivery:** descartada por exigir coordenação transacional entre os dois lados (OMS e sistema do cliente), elevando significativamente a complexidade de implementação para resolver um problema que at-least-once + dedup já resolve "em 99% dos casos" ([09:25] Diego).

## Consequências

**Positivas**
- Simplicidade de implementação do lado do OMS: não é preciso coordenar confirmação transacional de entrega com o cliente.
- Alinhado a um padrão de mercado já conhecido e documentável para os clientes B2B.

**Negativas / Trade-offs**
- Responsabilidade de deduplicação é transferida para o cliente, exigindo que ele implemente controle de idempotência do lado dele — a Engenheira de Segurança registrou essa preocupação (joga responsabilidade pro cliente), mas o time optou por aceitar o trade-off ([09:25] Sofia, Diego). O PM se compromete a documentar isso de forma destacada no portal de desenvolvedor ([09:26] Marcos).
- Não há garantia de ordenação global entre eventos de pedidos diferentes; a ordenação só é implícita por `order_id` e apenas enquanto o worker for single-instance (ver [ADR-002](ADR-002-worker-processo-separado-polling.md)) — limitação conhecida e documentada, não um requisito dos clientes atuais ([09:12]-[09:14] Diego, Bruno, Marcos).

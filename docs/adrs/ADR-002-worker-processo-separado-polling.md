# ADR-002: Worker de entrega como processo separado, em polling

## Status

Aceito — 2026-10 (reunião técnica de 09:00, decisão em [09:09]-[09:11])

## Contexto

Os eventos registrados em `webhook_outbox` (ver [ADR-001](ADR-001-outbox-pattern-mysql.md)) precisam ser lidos e entregues via HTTP para os endpoints cadastrados pelos clientes. Era necessário decidir dois pontos: (1) como o processo que lê a outbox é notificado de novos eventos, e (2) se esse processo roda dentro da própria API ou separado.

MySQL não possui um mecanismo nativo de notificação de processo externo equivalente ao `LISTEN/NOTIFY` do PostgreSQL. Triggers de banco só executam SQL, não acionam processos externos ([09:09] Diego).

Rodar o worker dentro do mesmo processo da API (`src/server.ts`) faria o worker morrer a cada reinício/deploy da API ([09:11] Diego).

## Decisão

O worker roda como **processo Node.js separado** (`src/worker.ts`, novo script `npm run worker`), conectando ao mesmo banco MySQL com uma instância própria de `PrismaClient` — pois `PrismaClient` é por processo, não pode ser compartilhado entre processos Node distintos ([09:11] Larissa, [09:30] Bruno).

O worker busca eventos pendentes em **polling, a cada 2 segundos**, em lotes pequenos ordenados por `created_at`, processa e marca como entregue ou falho ([09:09] Diego).

A latência mínima de entrega passa a ser de até 2 segundos no pior caso, o que atende ao requisito de "abaixo de 10 segundos" definido pelos clientes B2B como "tempo real" ([09:02] Marcos, [09:10] Larissa).

## Alternativas Consideradas

- **Trigger de banco acionando o worker diretamente:** descartado porque MySQL não suporta notificação de processo externo nativamente; qualquer solução (escrever em arquivo, chamar um endpoint a partir de um trigger) foi considerada artificial e frágil ([09:09] Diego).
- **Worker embutido no processo da API:** descartado porque reinícios/deploys da API matariam o worker, interrompendo o processamento da fila sem necessidade ([09:11] Diego).

## Consequências

**Positivas**
- Independência de ciclo de vida: deploy ou crash da API não interrompe o processamento de eventos pendentes.
- Simplicidade operacional: não requer broker de mensagens nem mecanismo de push do banco.
- Latência previsível e suficiente para o requisito do cliente (abaixo de 10s).

**Negativas / Trade-offs**
- Latência mínima fixa de ~2 segundos mesmo em baixa carga, por não ser reativo.
- Exige gerenciar mais um processo em produção (deploy, monitoramento, restart automático).
- Com um único worker, o processamento é sequencial; ordering entre eventos de pedidos diferentes fica implícito a essa decisão de single-worker (ver [ADR-005](ADR-005-at-least-once-x-event-id.md) e a limitação de ordering documentada no FDD).

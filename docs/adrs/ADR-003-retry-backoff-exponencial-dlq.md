# ADR-003: Retry com backoff exponencial e Dead Letter Queue dedicada

## Status

Aceito — 2026-10 (reunião técnica de 09:00, decisão em [09:14]-[09:18])

## Contexto

Clientes externos podem estar temporariamente indisponíveis (manutenção planejada, instabilidade, etc.). O sistema precisa de uma estratégia de reentrega que tolere indisponibilidades realistas sem manter eventos pendurados indefinidamente. Já houve um caso de cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

Era necessário decidir: número de tentativas, progressão do backoff, e onde ficam os eventos que esgotaram as tentativas.

## Decisão

- **5 tentativas no total**, com **backoff exponencial**: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas entre tentativas — cobrindo uma janela de falha de até ~15 horas entre a primeira falha e a última tentativa ([09:15]-[09:17] Diego, Larissa).
- Esgotadas as 5 tentativas, o evento é movido para uma tabela separada **`webhook_dead_letter`**, com payload, motivo da falha e timestamp, mantendo a tabela `webhook_outbox` principal limpa para leitura do worker ([09:18] Diego).
- Reprocessamento de itens em DLQ é **manual**, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento como pendente na outbox (ver [ADR-006](ADR-006-reuso-padroes-existentes.md) para o controle de acesso desse endpoint) ([09:18] Diego, [09:19] Larissa).

## Alternativas Consideradas

- **3 tentativas:** considerado mais agressivo, mas insuficiente — mataria o evento em ~30 minutos, não cobrindo indisponibilidades reais já observadas em clientes (ex.: manutenção planejada de 2h) ([09:16] Bruno propôs, [09:16] Diego descartou).
- **Retry indefinido:** descartado por manter eventos pendurados para sempre caso o cliente nunca mais responda, sem sinal claro de falha permanente ([09:15] Diego).
- **Marcar como "failed" na própria tabela `webhook_outbox`** em vez de tabela separada: descartado em favor de uma tabela dedicada, por manter a outbox principal mais limpa para o worker e por servir melhor como evidência de debug/reprocessamento ([09:18] Diego).

## Consequências

**Positivas**
- Tolera indisponibilidades realistas de clientes (até ~15h) sem exigir intervenção humana imediata.
- DLQ dedicada preserva histórico de falhas para auditoria e debug, sem poluir a tabela operacional principal.
- Reprocessamento manual evita reentregas automáticas e descontroladas de eventos já declarados como falha permanente.

**Negativas / Trade-offs**
- Eventos que esgotam o retry exigem intervenção manual via endpoint admin — não há reentrega automática após a DLQ.
- Clientes com indisponibilidade maior que ~15h perdem o evento automaticamente (fica em DLQ até replay manual), o que foi considerado aceitável pelo PM ([09:17] Marcos).
- Não há notificação proativa (e-mail, alerta) quando um webhook entra em DLQ — isso foi explicitamente adiado para fase futura ([09:37]-[09:38] Marcos, Larissa).

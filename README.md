# Sistema de Webhooks de Notificação de Pedidos — Design Docs

> O enunciado original deste desafio está preservado em [`docs/ENUNCIADO.md`](docs/ENUNCIADO.md). Este README documenta o processo de produção da documentação, não o enunciado em si.

## Sobre o desafio

O desafio consistiu em transformar a transcrição literal de uma reunião técnica (`TRANSCRICAO.md`) e o código de um Order Management System existente em um pacote completo de design docs — PRD, RFC, FDD, ADRs e um tracker de rastreabilidade — para a feature "Sistema de Webhooks de Notificação de Pedidos". A reunião já tinha fechado as decisões técnicas (outbox, retry, HMAC, etc.); o trabalho aqui não foi decidir a arquitetura, mas extrair com fidelidade o que foi decidido, descartado ou adiado, e documentá-lo em diferentes alturas (produto, arquitetura, decisão pontual, implementação), sem inventar requisitos que não tivessem origem identificável na transcrição ou no código.

A IA (Claude Code) foi a ferramenta principal de produção: leu a transcrição e o código-fonte do OMS, propôs a estrutura de cada documento e gerou o conteúdo. Meu papel foi definir o escopo de cada entrega, apontar a IA para os trechos certos da transcrição, revisar cada documento linha a linha contra a fonte, e exigir correções quando algo ficava genérico, duplicado entre documentos, ou sem rastreabilidade clara.

## Ferramentas de IA utilizadas

- **Claude Code** — usado para ler o repositório inteiro (código em `src/`, `prisma/schema.prisma`) e a transcrição completa, e gerar os seis documentos (PRD, RFC, FDD, 6 ADRs, Tracker) a partir desse contexto combinado, em uma única sessão com acesso direto ao filesystem do projeto.

## Workflow adotado

Segui a ordem sugerida no enunciado original, que faz sentido porque cada documento depende do anterior:

1. **Leitura integral da transcrição e do código relevante primeiro.** Antes de gerar qualquer documento, pedi à IA para ler `TRANSCRICAO.md` por completo e os arquivos centrais do domínio de pedidos (`order.service.ts`, `order.status.ts`, classes de erro, middlewares de auth/erro, logger). Isso evitou que os documentos citassem arquivos ou padrões que não existem no repositório.
2. **ADRs primeiro.** Identifiquei as 6 decisões arquiteturais discutidas na reunião (outbox, worker/polling, retry+DLQ, HMAC, at-least-once, reuso de padrões) e pedi um ADR por decisão, cada um linkando a alternativa descartada com o trade-off explícito citado na própria call.
3. **RFC em cima dos ADRs.** O RFC consolidou a proposta em nível de arquitetura, referenciando os ADRs já escritos em vez de repetir o conteúdo deles, e isolou as alternativas descartadas e questões em aberto que a transcrição deixava claras (rate limiting, notificação proativa, escalabilidade do worker, dashboard).
4. **FDD depois do RFC.** Com a decisão e a proposta fechadas, o FDD desceu ao nível de implementação: fluxos passo a passo, contratos HTTP com payload de exemplo, matriz de erros `WEBHOOK_*`, e a seção obrigatória de integração com arquivos reais do código (`order.service.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `routes/index.ts`, `database.ts`).
5. **PRD por último entre os grandes documentos.** Com RFC, FDD e ADRs prontos, o PRD foi essencialmente uma consolidação em nível de produto/negócio, com cuidado extra em não deixar passar nenhum item citado nos documentos técnicos que não aparecesse no PRD em forma de requisito ou critério de aceite.
6. **Tracker em paralelo/no fim.** Montei o tracker revisando os quatro documentos técnicos linha a linha, extraindo cada afirmação factual e localizando o timestamp exato na transcrição ou o caminho de arquivo no código. Qualquer item para o qual eu não conseguia apontar uma origem foi removido do documento correspondente antes de entrar no tracker.
7. **README por último.** Escrito depois de todo o processo concluído, para poder descrever com precisão as iterações reais.

## Prompts customizados

Dois exemplos representativos dos prompts usados ao longo do processo (adaptados, não literais, para preservar o fluxo real de iteração):

```
Leia TRANSCRICAO.md por completo e os arquivos em src/modules/orders/,
src/shared/errors/ e src/middlewares/. Depois, liste separadamente:
(1) decisões arquiteturais fechadas na reunião, com o timestamp e quem decidiu;
(2) requisitos funcionais explícitos; (3) itens explicitamente descartados
ou adiados, com o timestamp; (4) pontos técnicos secundários (formato de
payload, headers, timeouts) que podem virar ADR ou só FDD.
Não gere nenhum documento ainda — só a lista com a origem de cada item.
```

```
Gere o ADR-003 sobre a política de retry e DLQ. A seção "Alternativas
Consideradas" precisa citar pelo menos uma alternativa que foi
literalmente proposta e rejeitada na reunião (não uma alternativa
genérica de livro-texto), com o trade-off exato que motivou o descarte,
citando o timestamp. Não repita no ADR o nível de detalhe de contratos
HTTP — isso é problema do FDD.
```

## Iterações e ajustes

O processo levou essencialmente 3 ciclos principais até fechar:

1. **Primeira geração dos ADRs vinha genérica demais.** A primeira tentativa de ADR sobre o padrão outbox descrevia "outbox pattern" em termos de livro-texto, sem citar a tabela `webhook_outbox`, os índices mencionados por Diego em [09:08], ou o motivo real do descarte do Redis Streams (time pequeno, overengineering). Corrigi pedindo explicitamente que cada seção "Alternativas Consideradas" e "Consequências" citasse o timestamp da transcrição que sustenta a afirmação — isso forçou reescrita de praticamente todos os ADRs.
2. **RFC duplicava o nível de detalhe do FDD na primeira versão.** A primeira versão do RFC incluía exemplos de payload JSON e a matriz de erros `WEBHOOK_*` completa — conteúdo que pertence ao FDD. Precisei reescrever a seção "Proposta técnica" do RFC para ficar em nível de visão geral (outbox + worker + resiliência/segurança, cada um em um parágrafo), movendo todo o detalhamento de contrato para o FDD e linkando os ADRs em vez de repetir o raciocínio deles.
3. **Tracker com itens sem origem rastreável.** Ao montar o tracker, identifiquei linhas do PRD (ex.: uma menção genérica a "SLA de disponibilidade do webhook") para as quais eu não conseguia apontar nem timestamp nem arquivo de código — sinal de que tinham sido inferidas, não extraídas. Removi esses itens do PRD antes de fechar o tracker, seguindo a dica do próprio enunciado de que a ausência de "Localização" é sinal de alucinação.

## Como navegar a entrega

Ordem de leitura sugerida:

1. [`docs/PRD.md`](docs/PRD.md) — por que a feature existe, para quem, e o que fica fora.
2. [`docs/RFC.md`](docs/RFC.md) — a proposta técnica de alto nível, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — as 6 decisões arquiteturais isoladas, cada uma com contexto, alternativa e trade-off:
   - `ADR-001-outbox-pattern-mysql.md`
   - `ADR-002-worker-processo-separado-polling.md`
   - `ADR-003-retry-backoff-exponencial-dlq.md`
   - `ADR-004-hmac-sha256-secret-por-endpoint.md`
   - `ADR-005-at-least-once-x-event-id.md`
   - `ADR-006-reuso-padroes-existentes.md`
4. [`docs/FDD.md`](docs/FDD.md) — fluxos, contratos HTTP, matriz de erros e integração detalhada com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — para verificar, item a item, a origem (transcrição ou código) de qualquer afirmação dos documentos anteriores.
6. [`TRANSCRICAO.md`](TRANSCRICAO.md) — a fonte primária, para quem quiser conferir o contexto completo de qualquer timestamp citado.

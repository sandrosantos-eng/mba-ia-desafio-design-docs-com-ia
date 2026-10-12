# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito — 2026-10 (reunião técnica de 09:00, decisão em [09:19]-[09:22])

## Contexto

O sistema vai enviar dados de pedidos para endpoints HTTP fora da infraestrutura da empresa (webhooks outbound). O cliente que recebe precisa conseguir validar que a requisição realmente veio do OMS e que o payload não foi adulterado em trânsito ([09:19] Sofia).

Já houve um incidente anterior em que um cliente vazou uma secret em log de aplicação dele ([09:22] Diego), reforçando a necessidade de isolamento de blast radius entre endpoints/clientes e de um mecanismo de rotação.

## Decisão

- Cada chamada de webhook é assinada com **HMAC-SHA256** sobre o corpo (body) do request. A assinatura vai no header `X-Signature` ([09:20] Sofia, Bruno).
- Cada **endpoint de webhook cadastrado** (não cada cliente, não uma secret global da plataforma) tem sua **própria secret**, armazenada junto com `url`, `customer_id` e estado ativo na configuração do webhook ([09:21] Sofia, Bruno).
- A secret é **rotacionável** via API. Ao rotacionar, a secret antiga permanece válida em paralelo por **24 horas** (grace period), para o cliente migrar os sistemas dele sem downtime; depois disso, a secret antiga é invalidada ([09:21]-[09:22] Sofia).

## Alternativas Consideradas

- **Secret única/global da plataforma:** descartada explicitamente — se vazasse, comprometeria a validação de todos os clientes de uma vez. Secret por endpoint limita o blast radius de um vazamento a um único cadastro ([09:21] Sofia).
- **Rotação sem grace period (invalidação imediata da secret antiga):** implícita como alternativa mais simples, mas rejeitada porque quebraria a integração do cliente no instante da rotação, sem janela para ele atualizar a secret do lado dele — motivada pelo caso real de vazamento de secret em log de cliente ([09:22] Diego, Sofia).

## Consequências

**Positivas**
- Blast radius de uma secret vazada fica restrito a um único endpoint/cliente.
- Rotação com grace period de 24h permite remediar um vazamento sem quebrar a integração do cliente afetado.
- HMAC-SHA256 é padrão de mercado amplamente suportado por bibliotecas client-side, reduzindo o esforço de integração dos clientes B2B.

**Negativas / Trade-offs**
- Durante o grace period de 24h, a aplicação precisa validar a assinatura contra duas secrets (atual e anterior) simultaneamente, aumentando levemente a complexidade da verificação (não aplicável no lado do cliente, só na geração/gestão interna da secret).
- A revisão de segurança da geração de secret e do fluxo de HMAC foi marcada como obrigatória antes do deploy, reservando ao menos dois dias úteis da engenheira de segurança ([09:46] Sofia) — isso é uma dependência de cronograma, não só uma decisão técnica.
- TLS obrigatório (HTTPS) nas URLs cadastradas é tratado apenas como validação de schema (Zod), não como ADR separado, mas é pré-condição para a garantia de integridade do HMAC fazer sentido ([09:23] Sofia).

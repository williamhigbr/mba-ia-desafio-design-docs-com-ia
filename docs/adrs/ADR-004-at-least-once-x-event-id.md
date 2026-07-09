# ADR-004 — Garantia At-Least-Once com Idempotência via X-Event-Id

## Status

Aceito

## Contexto

Em qualquer sistema de mensageria distribuído, a garantia de entrega pode ser classificada
em três categorias: at-most-once (no máximo uma entrega, possível perda), at-least-once
(mínimo uma entrega, possível duplicata) e exactly-once (exatamente uma entrega, sem perda
nem duplicata).

O sistema de webhooks usa retries automáticos para resiliência (ADR-002). Isso significa
que, em caso de falha com resposta ambígua do servidor do cliente (timeout, 500 com
processamento parcial), o worker tentará reenviar o mesmo evento. O cliente pode receber
o mesmo evento mais de uma vez — isso é uma consequência direta da estratégia de retry.

A questão é qual das três garantias adotar:

- **At-most-once** eliminaria retries, sacrificando resiliência.
- **Exactly-once** exigiria coordenação bidirecional entre o OMS e o sistema receptor do
  cliente: o receptor teria que reportar de volta ao OMS que processou o evento, e o OMS
  teria que manter estado suficiente para saber que nunca mais precisa reenviar. Isso
  duplica a complexidade de integração e introduz novos pontos de falha.
- **At-least-once** mantém a resiliência via retry e delega a deduplicação ao cliente,
  fornecendo um identificador único de evento (`X-Event-Id`) que o cliente pode usar para
  detectar e descartar duplicatas.

O padrão at-least-once com `X-Event-Id` é o modelo adotado pelas plataformas de referência
do mercado — Stripe, GitHub, Shopify — tornando-o familiar para as equipes de integração
dos clientes B2B.

## Decisão

O sistema adota garantia de entrega at-least-once. Cada evento recebe um UUID único
(`event_id`) no momento da inserção na `webhook_outbox`. Esse UUID é enviado no header
`X-Event-Id` em todas as tentativas de entrega do mesmo evento. O cliente é responsável
por usar o `X-Event-Id` para deduplicação de sua parte. A plataforma documenta esse
comportamento no portal do cliente.

## Alternativas Consideradas

### Alternativa A — Garantia exactly-once

O OMS manteria estado do processamento por evento no lado do cliente: após confirmação de
recebimento (ex: resposta `200 OK`), marcaria o evento como definitivamente entregue e
nunca mais o reenviaria, mesmo em retries de outros componentes.

**Por que descartada:** exigiria que o cliente expusesse algum mecanismo de confirmação
além do simples `200 OK` (ou que o OMS mantivesse registro por cliente de eventos
confirmados), criando coordenação bidirecional entre dois sistemas controlados por partes
diferentes. A complexidade de implementação é significativamente maior, e os casos de
falha na confirmação seriam difíceis de tratar. O padrão de mercado (Stripe, GitHub) optou
deliberadamente por at-least-once porque os benefícios do exactly-once não compensam o
custo. Descartada por Diego em `[09:25]`.

### Alternativa B — At-most-once (sem retry)

Tentar a entrega uma vez; se falhar, descartar o evento sem reprocessamento.

**Por que descartada:** sacrifica completamente a resiliência. Um timeout de rede ou
reinicialização momentânea do servidor do cliente resultaria em perda definitiva do evento.
Inaceitável para clientes B2B que dependem das notificações para acionar fluxos de negócio
(separação de mercadoria, atualização de ERP). Não foi proposta formalmente na reunião;
descartada implicitamente pela adoção dos retries em `[09:15]–[09:17]`.

## Consequências

### Positivas

- Modelo simples de implementar tanto na plataforma quanto no lado do cliente.
- Idêntico ao padrão adotado por Stripe e GitHub: equipes de integração já familiarizadas
  com o conceito de `X-Event-Id` e deduplicação.
- Mantém compatibilidade total com a estratégia de retries (ADR-002): cada tentativa
  reutiliza o mesmo `X-Event-Id`, facilitando deduplicação no receptor.
- O UUID da outbox (`event_id`) serve tanto como identificador de evento para deduplicação
  quanto como chave de rastreabilidade em logs e histórico de entregas.

### Negativas / Trade-offs

- O cliente é responsável por implementar deduplicação se não quiser processar duplicatas.
  Sem isso, um retry pode causar processamento duplo de uma ação de negócio.
- A plataforma precisa documentar claramente o comportamento (Marcos se comprometeu com
  isso em `[09:26]`).
- Em cenários de alta falha seguida de replay manual da DLQ, o cliente pode receber o mesmo
  evento com dias de diferença — o `X-Event-Id` continua sendo o mesmo, facilitando
  identificação, mas o cliente precisa estar preparado para esse padrão.

## Referências

- Transcrição: `[09:24]–[09:26] Diego / Larissa` — decisão por at-least-once e X-Event-Id
- Transcrição: `[09:25] Diego` — descarte do exactly-once por complexidade bidirecional
- Transcrição: `[09:26] Marcos` — comprometimento com documentação no portal do cliente
- Transcrição: `[09:44]–[09:45] Diego / Sofia` — confirmação do header X-Event-Id no payload final
- Código: `prisma/schema.prisma` — campo `eventId` em `WebhookOutbox` (`@unique`, UUID, gerado na inserção)

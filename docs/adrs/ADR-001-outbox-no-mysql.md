# ADR-001 — Padrão Outbox no MySQL para Disparo de Webhooks

## Status

Aceito

## Contexto

O sistema precisa notificar clientes B2B toda vez que um pedido muda de status. O ponto
natural de integração é o método `changeStatus` em
`src/modules/orders/order.service.ts`, que já executa uma transação SQL composta: valida
a transição, ajusta estoque, atualiza o pedido e insere o histórico. Em produção, essa
transação envolve ao menos três tabelas (`orders`, `order_status_history`,
`stock_quantities`).

A questão central é: como garantir que o evento de webhook seja disparado **exatamente
quando** a mudança de status for confirmada, sem acoplar o ciclo de vida da transação ao
comportamento do endpoint HTTP do cliente?

Disparar o HTTP de forma síncrona dentro da transação criaria um acoplamento direto: se o
cliente estiver lento ou offline, a transação ficaria aberta por segundos ou travaria
completamente, bloqueando o `changeStatus` para qualquer outro pedido. Disparar após o
`commit` da transação, de forma assíncrona na mesma thread, cria uma janela de falha: se
a API reiniciar entre o `commit` e o disparo, o evento se perde sem rastro.

O padrão Outbox resolve esse problema inserindo o evento de webhook na mesma transação
SQL que confirma a mudança de status. A tabela `webhook_outbox` passa a ser parte do
`$transaction` do Prisma. Se a transação confirmar, o evento existe. Se der rollback,
o evento some junto. Um worker separado lê e despacha os eventos de forma assíncrona,
completamente fora do fluxo da API.

## Decisão

Adotado o padrão Outbox com tabela `webhook_outbox` no mesmo banco MySQL. A inserção na
outbox ocorre dentro do `$transaction` do `changeStatus`, via função
`publishWebhookEvent(tx, order, fromStatus, toStatus)`. O worker consome a outbox de forma
assíncrona e independente da API.

## Alternativas Consideradas

### Alternativa A — Disparo HTTP síncrono dentro da transação de `changeStatus`

O webhook seria disparado como chamada HTTP diretamente dentro do bloco
`this.prisma.$transaction(...)` do `changeStatus`. Garantiria que o evento só sai se a
transação confirmar.

**Por que descartada:** a transação já é pesada (atualiza pedido, histórico, estoque).
Adicionar um `fetch()` síncrono manteria o lock aberto enquanto aguarda resposta do
cliente. Um cliente lento (timeout configurado em 10s) travaria a transação por 10
segundos. Se o cliente estiver offline, a única saída seria rollback da mudança de status
— o que reverte uma operação de negócio para compensar uma falha de notificação, um
trade-off inaceitável. Descartada por Bruno e Diego em `[09:04]–[09:06]`.

### Alternativa B — Fila externa (Redis Streams ou equivalente)

Publicar o evento em uma fila externa (Redis Streams, RabbitMQ, SQS) após o `commit` da
transação. O worker consumiria dessa fila em vez de fazer polling no banco.

**Por que descartada:** exigiria subir nova infraestrutura (Redis ou broker) em um time
pequeno. O volume atual de eventos não justifica a complexidade operacional adicional. O
MySQL existente resolve o problema com o padrão Outbox sem nenhuma dependência nova.
Descartada por Diego e Larissa em `[09:07]`.

## Consequências

### Positivas

- Atomicidade garantida: evento de webhook nunca é criado sem a mudança de status
  correspondente, e vice-versa.
- Nenhuma infraestrutura nova: usa o MySQL já em produção.
- Desacoplamento total: a API retorna imediatamente após o `commit`; o worker opera
  de forma independente.
- Rastreabilidade: a outbox serve como log de eventos pendentes, processados e com falha.

### Negativas / Trade-offs

- Latência não é zero: o evento é processado pelo worker em polling de 2 segundos, o
  que introduz uma latência mínima garantida de ~0–2s (dentro do requisito de <10s).
- Operação adicional na transação: cada `changeStatus` passa a incluir um `INSERT` na
  `webhook_outbox`, aumentando levemente o custo da transação.
- Crescimento da tabela: linhas entregues precisam ser arquivadas periodicamente (previsto
  para fase futura, conforme `[09:08] Diego`).
- Worker em polling gera carga constante de leitura no banco; mitigado pelo índice em
  `(status, next_retry_at)`.

## Referências

- Transcrição: `[09:06] Diego` — proposta do padrão Outbox
- Transcrição: `[09:07] Larissa` — confirmação da decisão e rejeição de infra nova
- Transcrição: `[09:04]–[09:06] Bruno / Diego` — descarte do disparo síncrono
- Transcrição: `[09:40]–[09:41] Bruno / Diego / Larissa` — confirmação de que `publishWebhookEvent` fica dentro do `$transaction`
- Código: `src/modules/orders/order.service.ts` — método `changeStatus` e uso de `$transaction`
- Código: `prisma/schema.prisma` — modelo base para o novo modelo `WebhookOutbox`

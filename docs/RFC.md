# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo     | Valor |
|-----------|-------|
| Autor     | Diego (Eng. Sênior — Plataforma) |
| Status    | Em revisão |
| Data      | 2026-07-09 |
| Revisores | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Sofia (Eng. de Segurança) |

---

## TL;DR

Propomos um sistema de webhooks outbound que notifica clientes B2B sobre mudanças de status
de pedidos. A solução usa o padrão **Outbox no MySQL**: dentro da mesma transação SQL que
confirma a mudança de status, um evento é inserido em uma tabela `webhook_outbox`. Um
**worker em processo separado** consome essa tabela em polling de 2 segundos e faz as
chamadas HTTP para os endpoints dos clientes. Cada entrega é assinada com **HMAC-SHA256**
usando um segredo único por endpoint. A garantia de entrega é **at-least-once**, com
`X-Event-Id` para deduplicação pelo cliente. A solução não exige nova infraestrutura e
reutiliza todos os padrões do projeto existente.

---

## Contexto e Problema

Três clientes B2B estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — dependem
de saber em tempo quase real quando o status de um pedido muda para acionar fluxos internos
(separação de mercadoria, atualização de ERP, notificação de transportadora). Hoje, a única
alternativa disponível é polling periódico na API do OMS, o que gera carga desnecessária,
latência imprevisível e acoplamento operacional entre os sistemas. `[09:02] Marcos`

O OMS não tem nenhum mecanismo de notificação externa. Não há eventos, filas, webhooks ou
qualquer forma de push. Essa lacuna está causando fricção de integração e foi apontada como
risco de churn pelos três clientes citados. `[09:00] Marcos`

A feature precisa preencher exatamente essa lacuna: permitir que clientes cadastrem
endpoints HTTP e recebam notificações automáticas quando pedidos transitarem entre estados
— sem que isso impacte a performance ou a confiabilidade do fluxo principal de mudança de
status.

---

## Proposta Técnica

### Inserção atômica via Outbox

O ponto de integração com o sistema existente é o método `changeStatus` em
`src/modules/orders/order.service.ts`. Esse método já executa uma `$transaction` Prisma
que atualiza o pedido, debita ou repõe estoque e insere o histórico de status. Dentro dessa
mesma transação, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` consultará
os endpoints webhook ativos do customer que tenham o novo status no filtro e inserirá um
evento na tabela `webhook_outbox` para cada um. Se a transação der rollback por qualquer
motivo, o evento some junto — sem eventos órfãos.

### Worker assíncrono em processo separado

Um processo Node.js independente (`src/worker.ts`, novo) faz polling na `webhook_outbox`
a cada 2 segundos, buscando eventos com `status = 'PENDING'` e `nextRetryAt <= now()`.
Para cada evento, o worker faz um HTTP POST para o endpoint do cliente com timeout de 10
segundos. A separação em processo distinto garante que um deploy ou crash da API não
interrompa entregas de eventos já enfileirados. `[09:11] Diego`

### Resiliência com retry e DLQ

Falhas de entrega (timeout, 4xx, 5xx) disparam uma política de retry com backoff exponencial
em 5 tentativas distribuídas ao longo de aproximadamente 15 horas (1m / 5m / 30m / 2h / 12h).
Após esgotadas as tentativas, o evento é movido para `webhook_dead_letter`. Um endpoint admin
(`POST /admin/webhooks/dead-letter/:id/replay`) permite reprocessamento manual com role
`ADMIN`. Ver [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md).

### Autenticação e segurança

Cada entrega é assinada com HMAC-SHA256 calculado sobre o corpo da requisição usando um
segredo único por endpoint, gerado pela plataforma na criação. O segredo é retornado ao
cliente apenas no momento da criação e pode ser rotacionado via endpoint dedicado, com
grace period de 24 horas para que o cliente atualize sua configuração sem downtime. TLS é
obrigatório: URLs com `http://` são recusadas na validação. `[09:20]–[09:22] Sofia / Larissa`.
Ver [ADR-003](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md).

### Garantia de entrega

A entrega é **at-least-once**: o worker tentará reenviar em caso de falha, podendo o cliente
receber duplicatas. O header `X-Event-Id` (UUID gerado na inserção da outbox, imutável entre
retries) é o mecanismo padrão para deduplicação no lado do cliente — o mesmo padrão adotado
por Stripe e GitHub. `[09:24]–[09:26] Diego / Larissa`. Ver
[ADR-004](./adrs/ADR-004-at-least-once-x-event-id.md).

### Encaixe na arquitetura existente

O módulo de webhooks segue os padrões da codebase sem exceção: estrutura
`src/modules/webhooks/` espelhando `src/modules/orders/`, erros com prefixo `WEBHOOK_`
estendendo `AppError`, logger Pino existente, `requireRole` do middleware de autenticação.
Nenhuma nova dependência de infraestrutura. `[09:28]–[09:30] Bruno / Diego / Larissa`. Ver
[ADR-001](./adrs/ADR-001-outbox-no-mysql.md), [ADR-005](./adrs/ADR-005-worker-separado-polling.md)
e [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md).

---

## Alternativas Consideradas

### Disparo síncrono do webhook dentro da transação de `changeStatus`

A transição de status poderia disparar o HTTP call para o cliente diretamente dentro do
bloco `$transaction`, garantindo que a notificação só sai se a transação confirmar.

**Por que descartada:** a transação de `changeStatus` já envolve atualização de pedido,
histórico e estoque. Adicionar um `fetch()` síncrono manteria o lock de banco aberto
enquanto aguarda o endpoint do cliente — até 10 segundos no pior caso. Um cliente offline
exigiria rollback da mudança de status para compensar a falha de notificação, revertendo
uma operação de negócio válida. Inaceitável. `[09:04]–[09:06] Bruno / Diego`

### Fila externa (Redis Streams ou equivalente) como intermediário

Eventos de webhook poderiam ser publicados em uma fila externa (Redis Streams, RabbitMQ,
SQS) após o commit, com um worker consumindo dessa fila.

**Por que descartada:** exigiria subir e operar nova infraestrutura em um time pequeno,
com prazo de 3 sprints. O volume atual de eventos não justifica a complexidade adicional.
O MySQL existente resolve o problema com o padrão Outbox sem nenhuma dependência nova.
`[09:07] Diego / Larissa`

---

## Questões em Aberto

### Rate limiting de envio por cliente

Se um cliente tiver muitos pedidos mudando de status em curto espaço de tempo, o worker
pode disparar dezenas de chamadas HTTP em sequência para o mesmo endpoint, sobrecarregando
o servidor do cliente.

**Levantado por:** Diego `[09:38]–[09:39]`  
**Status:** explicitamente adiado. Larissa registrou como "observar e decidir depois".  
**Próximo passo:** monitorar volume de eventos por cliente em produção; implementar
rate limiting por webhook na fase 2 se os dados indicarem necessidade.

### Controle de acesso para CRUD de configuração de webhook

Os endpoints de criação, edição e remoção de webhooks aceitam qualquer usuário autenticado
(`authenticate` sem `requireRole`). Para operações de produção, pode ser desejável restringir
a configuração de webhooks a roles específicas.

**Levantado por:** Sofia `[09:37]`  
**Status:** explicitamente adiado. Sofia sinalizou "mais pra frente a gente pode endurecer".  
**Próximo passo:** revisão da política de acesso na fase 2, após observar quem de fato
opera os webhooks em produção.

---

## Impacto e Riscos

**Acúmulo na outbox durante indisponibilidade do worker:** se o worker ficar offline por
tempo prolongado (ex: falha de deploy), eventos acumulam na `webhook_outbox`. Quando o
worker volta, processa tudo em ordem, mas pode haver um pico de chamadas HTTP saindo
simultaneamente para os clientes. Mitigação: monitorar `webhook_outbox_pending_total` e
alertar se acima de threshold.

**Ordering não garantida com múltiplos workers:** a solução atual usa single-worker, o que
garante ordering implícita por `order_id`. Se o volume crescer e exigir escalar horizontalmente
para múltiplos workers em paralelo, a ordering global não será mais garantida sem mecanismo
de particionamento ou lock pessimista — problema explicitamente adiado para fase futura.
`[09:13] Diego`

**Grace period na rotação de secret:** durante as 24h de grace period, dois segredos são
válidos em paralelo. Um segredo comprometido permanece válido por até 24h após a rotação.
A janela é intencional para evitar downtime, mas deve ser documentada como risco de segurança
residual.

---

## Decisões Relacionadas

- [ADR-001 — Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com Backoff e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003 — HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-Least-Once com X-Event-Id](./adrs/ADR-004-at-least-once-x-event-id.md)
- [ADR-005 — Worker Separado em Polling](./adrs/ADR-005-worker-separado-polling.md)
- [ADR-006 — Reuso dos Padrões Existentes](./adrs/ADR-006-reuso-padroes-existentes.md)

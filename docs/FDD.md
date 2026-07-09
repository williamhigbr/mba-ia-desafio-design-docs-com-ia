# FDD — Sistema de Webhooks de Notificação de Pedidos

> **Documento de referência:** [RFC](./RFC.md) | **Decisões:** [ADRs](./adrs/)  
> Todo item rastreável ao `docs/context/TRANSCRIPT_CONTEXT.md` ou `docs/context/CODE_MAP.md`.

---

## 1. Contexto e Motivação Técnica

O OMS não possui nenhum mecanismo de notificação externa. Clientes B2B integrados à
plataforma recorrem a polling periódico na API para detectar mudanças de status de pedidos,
o que gera carga desnecessária, latência imprevisível e acoplamento operacional.

O ponto de integração central é o método `changeStatus` em
`src/modules/orders/order.service.ts`. Esse método executa uma `$transaction` Prisma
composta que atualiza o status do pedido, ajusta estoque e insere o histórico. É nesse
ponto que o sistema de webhooks deve se ancorar: inserindo um evento na `webhook_outbox`
dentro da mesma transação, garantindo atomicidade sem impactar o fluxo existente.
`[09:40]–[09:41] Bruno / Diego / Larissa`

Um worker em processo separado (`src/worker.ts`, novo) consome a outbox de forma
assíncrona, fazendo as chamadas HTTP para os endpoints cadastrados pelos clientes.

---

## 2. Objetivos Técnicos

- Latência de notificação < 10 segundos após confirmação da mudança de status `[09:02] Marcos`
- Garantia at-least-once de entrega com `X-Event-Id` para deduplicação `[09:25] Diego`
- Zero impacto na latência da transação de `changeStatus` (webhook é assíncrono)
- Isolamento de falha: crash ou deploy da API não interrompe entrega de eventos pendentes `[09:11] Diego`
- Sem nova infraestrutura: MySQL existente, sem Redis, sem filas externas `[09:07] Diego`
- Reuso de todos os padrões da codebase: `AppError`, Pino, `requireRole`, estrutura de módulo `[09:28]–[09:30] Larissa`

---

## 3. Escopo e Exclusões

### Arquivos novos a criar

| Arquivo | Descrição |
|---|---|
| `src/worker.ts` (novo) | Entry point do processo worker |
| `src/modules/webhooks/webhook.routes.ts` (novo) | Rotas Express do módulo |
| `src/modules/webhooks/webhook.controller.ts` (novo) | Controllers dos endpoints |
| `src/modules/webhooks/webhook.service.ts` (novo) | Lógica de negócio |
| `src/modules/webhooks/webhook.repository.ts` (novo) | Acesso ao Prisma |
| `src/modules/webhooks/webhook.schemas.ts` (novo) | Schemas Zod de validação |
| `src/modules/webhooks/webhook.processor.ts` (novo) | Lógica de processamento da outbox |
| `src/modules/webhooks/webhook.errors.ts` (novo) | Classes de erro `WEBHOOK_*` |

### Em escopo

- CRUD de endpoints webhook (`POST`, `GET`, `PATCH`, `DELETE`)
- Rotação de secret com grace period de 24h `[09:21] Sofia`
- Histórico de entregas: `GET /webhooks/:id/deliveries` `[09:34] Marcos`
- Replay manual de DLQ: `POST /admin/webhooks/dead-letter/:id/replay` `[09:18]–[09:19] Diego`
- Worker com polling, retry e DLQ

### Fora de escopo (decisão da reunião)

- Webhooks inbound `[09:02]–[09:03] Marcos / Sofia`
- Notificação por email em caso de falha `[09:37] Larissa`
- Dashboard visual para o cliente `[09:39]–[09:40] Larissa`
- Rate limiting de envio por cliente `[09:38]–[09:39] Diego / Larissa` — adiado para fase 2
- Arquivamento automático de eventos entregues na outbox `[09:08] Diego` — adiado
- Múltiplos workers em paralelo / particionamento da outbox `[09:13] Diego` — adiado

---

## 4. Fluxos Detalhados

### 4a. Criação do evento na outbox

```
1. API recebe PATCH /api/v1/orders/:id/status
2. OrderService.changeStatus(id, input, userId) abre this.prisma.$transaction(async (tx) => {
   a. Carrega order com items via tx
   b. Valida transição: canTransition(fromStatus, toStatus)
   c. Se shouldDebitStock: debita estoque via tx
   d. Se shouldReplenishStock: repõe estoque via tx
   e. tx.order.update({ status: toStatus })
   f. tx.orderStatusHistory.create({ fromStatus, toStatus, changedById })
   g. publishWebhookEvent(tx, order, fromStatus, toStatus):
      i.  Busca webhooks ativos do customer com toStatus no statusFilter
      ii. Para cada webhook encontrado:
          - Serializa payload como snapshot (não renderizado no envio)
          - Insere em webhook_outbox:
            { id: uuid(), webhookId, eventId: uuid(), eventType: "order.status_changed",
              payload: <snapshot JSON>, status: "PENDING", attempts: 0,
              nextRetryAt: now(), createdAt: now() }
   h. tx.order.findUnique — retorna order atualizada
3. $transaction commita (ou faz rollback de tudo, incluindo o evento da outbox)
})
```

**Snapshot do payload serializado na inserção** (não no momento do envio):
```json
{
  "event_id": "<eventId>",
  "event_type": "order.status_changed",
  "timestamp": "<ISO 8601>",
  "data": {
    "order_id": "<uuid>",
    "order_number": "ORD-000042",
    "customer_id": "<uuid>",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "total_cents": 15900
  }
}
```
O campo `items` não é incluído — cliente busca via `GET /orders/:id` se precisar.
`[09:43]–[09:45] Diego / Sofia`

---

### 4b. Processamento pelo worker

```
1. src/worker.ts inicia, cria PrismaClient próprio via createPrismaClient()
2. Loop a cada 2 segundos (WEBHOOK_WORKER_POLL_INTERVAL_MS, padrão 2000):
   a. SELECT webhook_outbox
      WHERE status = 'PENDING' AND nextRetryAt <= now()
      ORDER BY createdAt ASC
      LIMIT <batch_size>
   b. Para cada evento no batch:
      i.   Marca status = 'PROCESSING' (evita processamento duplo)
      ii.  Busca configuração do webhook (url, secret, active, statusFilter)
      iii. Se webhook.active = false:
           → move para DLQ com failureReason = "webhook_inactive"
           → marca outbox.status = 'FAILED'
           → log warn { webhookId, eventId }
           → continua para próximo evento
      iv.  Verifica tamanho do payload: se > 64KB → WEBHOOK_PAYLOAD_TOO_LARGE
           → move para DLQ com motivo do erro
      v.   Serializa headers:
           X-Event-Id: <eventId>
           X-Webhook-Id: <webhookId>
           X-Timestamp: <unix timestamp>
           X-Signature: sha256=<HMAC-SHA256(body, secret)>
           Content-Type: application/json
      vi.  HTTP POST <webhook.url> com timeout de 10s
      vii. Se resposta 2xx:
           → webhook_outbox.status = 'DELIVERED', processedAt = now()
           → insere em webhook_deliveries { success: true, httpStatusCode, durationMs }
           → log info { webhookId, eventId, attemptNumber, httpStatusCode, durationMs }
      viii.Se falha (timeout, 4xx, 5xx, erro de rede):
           → insere em webhook_deliveries { success: false, httpStatusCode?, durationMs }
           → log warn { webhookId, eventId, attemptNumber, error, nextRetryAt }
           → fluxo de retry (4c)
```

---

### 4c. Retry com backoff exponencial

```
Tabela de nextRetryAt por tentativa (attempts após a falha):

  attempts = 1 → nextRetryAt = now() + 1 minuto
  attempts = 2 → nextRetryAt = now() + 5 minutos
  attempts = 3 → nextRetryAt = now() + 30 minutos
  attempts = 4 → nextRetryAt = now() + 2 horas
  attempts = 5 → nextRetryAt = now() + 12 horas

Ao falhar:
  1. Incrementa webhook_outbox.attempts
  2. Se attempts < 5:
     → Calcula nextRetryAt conforme tabela acima
     → webhook_outbox.status = 'PENDING'
  3. Se attempts >= 5:
     → Fluxo DLQ (4d)
```
`[09:15]–[09:17] Diego / Larissa`

---

### 4d. Dead Letter Queue (DLQ)

```
Ao mover para DLQ:
  1. INSERT webhook_dead_letter:
     { id: uuid(), webhookId, eventId, payload: <mesmo snapshot>,
       failureReason: <mensagem de erro>, failedAt: now() }
  2. UPDATE webhook_outbox SET status = 'FAILED'
  3. log.error { webhookId, eventId, totalAttempts, failureReason }

Reprocessamento via endpoint admin:
  POST /admin/webhooks/dead-letter/:id/replay
  (requer authenticate + requireRole('ADMIN'))

  1. Busca dead_letter pelo :id
  2. Se já foi reprocessado (replayedAt != null) → 409 WEBHOOK_ALREADY_REQUEUED
  3. INSERT webhook_outbox:
     { id: uuid(), webhookId, eventId: <mesmo eventId>, eventType, payload,
       status: 'PENDING', attempts: 0, nextRetryAt: now() }
  4. UPDATE webhook_dead_letter SET replayedAt = now()
  5. log.info { adminUserId, eventId, deadLetterId }
  6. Retorna 200 { message, outboxId, eventId }
```
`[09:17]–[09:19] Diego / Larissa / Sofia`

---

## 5. Contratos Públicos (Endpoints HTTP)

Todos os endpoints usam `authenticate` como primeiro middleware. `[09:37] Sofia`

---

### POST /api/v1/webhooks
Cadastra novo endpoint webhook. A `secret` é gerada pela plataforma e retornada **apenas nesta resposta**.

**Auth:** `authenticate` (qualquer role autenticada)

**Request body:**
```json
{
  "customerId": "3f2a1b4c-0001-0001-0001-000000000001",
  "url": "https://cliente.com/webhooks/orders",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Response 201:**
```json
{
  "id": "7e8d9c0a-0002-0002-0002-000000000002",
  "customerId": "3f2a1b4c-0001-0001-0001-000000000001",
  "url": "https://cliente.com/webhooks/orders",
  "secret": "whsec_a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "createdAt": "2026-07-09T23:00:00.000Z"
}
```

**Status codes:**
- `201 Created` — criado com sucesso
- `400 WEBHOOK_INVALID_URL` — URL não-HTTPS ou malformada
- `400 VALIDATION_ERROR` — campos obrigatórios ausentes ou tipo inválido
- `400 WEBHOOK_INVALID_STATUS_FILTER` — status inválido no filtro
- `401 UNAUTHORIZED` — token ausente ou inválido
- `404 WEBHOOK_CUSTOMER_NOT_FOUND` — `customerId` não existe

---

### GET /api/v1/webhooks
Lista webhooks. **A `secret` nunca é retornada em listagens.**

**Auth:** `authenticate`

**Query params:** `customerId` (obrigatório), `page` (padrão 1), `pageSize` (padrão 20)

**Response 200:**
```json
{
  "data": [
    {
      "id": "7e8d9c0a-0002-0002-0002-000000000002",
      "customerId": "3f2a1b4c-0001-0001-0001-000000000001",
      "url": "https://cliente.com/webhooks/orders",
      "statusFilter": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-07-09T23:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**Status codes:**
- `200 OK`
- `400 VALIDATION_ERROR` — `customerId` ausente
- `401 UNAUTHORIZED`

---

### PATCH /api/v1/webhooks/:id
Atualiza configuração. Todos os campos são opcionais.

**Auth:** `authenticate`

**Request body:**
```json
{
  "url": "https://cliente.com/webhooks/orders-v2",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response 200:**
```json
{
  "id": "7e8d9c0a-0002-0002-0002-000000000002",
  "customerId": "3f2a1b4c-0001-0001-0001-000000000001",
  "url": "https://cliente.com/webhooks/orders-v2",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-07-09T23:00:00.000Z",
  "updatedAt": "2026-07-09T23:30:00.000Z"
}
```

**Status codes:**
- `200 OK`
- `400 WEBHOOK_INVALID_URL`
- `400 VALIDATION_ERROR`
- `401 UNAUTHORIZED`
- `404 WEBHOOK_NOT_FOUND`

---

### DELETE /api/v1/webhooks/:id
Remove endpoint webhook.

**Auth:** `authenticate`

**Response:** `204 No Content` (sem body)

**Status codes:**
- `204 No Content`
- `401 UNAUTHORIZED`
- `404 WEBHOOK_NOT_FOUND`

---

### GET /api/v1/webhooks/:id/deliveries
Histórico de tentativas de entrega. Máximo 100 itens por página. `[09:34] Marcos`

**Auth:** `authenticate`

**Query params:** `page` (padrão 1), `pageSize` (padrão 20, máx 100)

**Response 200:**
```json
{
  "data": [
    {
      "id": "a1b2c3d4-0003-0003-0003-000000000003",
      "eventId": "b2c3d4e5-0004-0004-0004-000000000004",
      "attemptNumber": 1,
      "httpStatusCode": 200,
      "durationMs": 342,
      "success": true,
      "attemptedAt": "2026-07-09T23:02:05.000Z"
    },
    {
      "id": "c3d4e5f6-0005-0005-0005-000000000005",
      "eventId": "d4e5f6a7-0006-0006-0006-000000000006",
      "attemptNumber": 1,
      "httpStatusCode": 503,
      "durationMs": 10001,
      "success": false,
      "attemptedAt": "2026-07-09T23:02:10.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 47, "totalPages": 3 }
}
```

**Status codes:**
- `200 OK`
- `401 UNAUTHORIZED`
- `404 WEBHOOK_NOT_FOUND`

---

### POST /api/v1/webhooks/:id/rotate-secret
Rotaciona a secret. A anterior permanece válida por 24h. `[09:21] Sofia`

**Auth:** `authenticate`

**Request body:** nenhum

**Response 200:**
```json
{
  "newSecret": "whsec_f0e9d8c7b6a5f4e3d2c1b0a9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0",
  "oldSecretExpiresAt": "2026-07-10T23:00:00.000Z"
}
```

**Status codes:**
- `200 OK`
- `401 UNAUTHORIZED`
- `404 WEBHOOK_NOT_FOUND`

---

### POST /admin/webhooks/dead-letter/:id/replay
Reprocessa evento da DLQ. `[09:18]–[09:19] Diego / Larissa`

**Auth:** `authenticate` + `requireRole('ADMIN')` `[09:36] Sofia / Larissa`

**Request body:** nenhum

**Response 200:**
```json
{
  "message": "Event requeued for delivery",
  "outboxId": "e5f6a7b8-0007-0007-0007-000000000007",
  "eventId": "d4e5f6a7-0006-0006-0006-000000000006"
}
```

**Status codes:**
- `200 OK`
- `401 UNAUTHORIZED`
- `403 FORBIDDEN` — usuário com role `OPERATOR`
- `404 WEBHOOK_DELIVERY_NOT_FOUND`
- `409 WEBHOOK_ALREADY_REQUEUED` — evento já foi recolocado na fila

---

## 6. Headers do Request de Webhook (Payload enviado ao cliente)

O worker envia um `HTTP POST` para o endpoint do cliente com os seguintes headers:

```
POST https://cliente.com/webhooks/orders
Content-Type: application/json
X-Event-Id: d4e5f6a7-0006-0006-0006-000000000006
X-Webhook-Id: 7e8d9c0a-0002-0002-0002-000000000002
X-Timestamp: 1752105600
X-Signature: sha256=3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b
```

**Cálculo do `X-Signature`:**
```
HMAC-SHA256(body_serializado_como_string_utf8, secret_do_endpoint)
Prefixo obrigatório: "sha256="
Implementação: Node.js nativo crypto.createHmac('sha256', secret).update(body).digest('hex')
```

**Body JSON:**
```json
{
  "event_id": "d4e5f6a7-0006-0006-0006-000000000006",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-09T23:02:05.000Z",
  "data": {
    "order_id": "f5a6b7c8-0008-0008-0008-000000000008",
    "order_number": "ORD-000042",
    "customer_id": "3f2a1b4c-0001-0001-0001-000000000001",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "total_cents": 15900
  }
}
```

O campo `items` não é incluído. `[09:43] Diego`  
O payload é serializado como **snapshot** no momento da inserção na outbox, não no momento
do envio. `[09:52] Larissa / Diego`

---

## 7. Matriz de Erros

Todos os erros do módulo usam prefixo `WEBHOOK_` e estendem `AppError` de
`src/shared/errors/app-error.ts`. `[09:29] Diego`

| Código | HTTP | Classe base | Quando ocorre |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Webhook ID inexistente em qualquer operação |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL não-HTTPS ou malformada na criação/edição |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `BadRequestError` | Status inválido no filtro (não pertence ao enum `OrderStatus`) |
| `WEBHOOK_INACTIVE` | 422 | `UnprocessableEntityError` | Tentativa de entrega para webhook com `active = false` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload acima de 64KB `[09:24] Diego / Larissa` |
| `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | `NotFoundError` | ID de dead_letter inexistente no replay |
| `WEBHOOK_ALREADY_REQUEUED` | 409 | `ConflictError` | Evento da DLQ já foi recolocado na fila (replayedAt != null) |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `NotFoundError` | `customerId` não existe ao cadastrar webhook |

**Exemplo de classe de erro:**
```typescript
// src/modules/webhooks/webhook.errors.ts
import { NotFoundError, BadRequestError } from '../../shared/errors/http-errors.js';

export class WebhookNotFoundError extends NotFoundError {
  constructor(id: string) {
    super(`Webhook ${id} not found`, 'WEBHOOK_NOT_FOUND');
  }
}

export class WebhookInvalidUrlError extends BadRequestError {
  constructor() {
    super('Webhook URL must use HTTPS', 'WEBHOOK_INVALID_URL');
  }
}
```

O `errorMiddleware` existente em `src/middlewares/error.middleware.ts` captura essas classes
automaticamente — sem alteração necessária. `[09:29]–[09:30] Diego / Larissa`

---

## 8. Estratégias de Resiliência

### Timeouts

- HTTP call do worker: **10 segundos** (`WEBHOOK_HTTP_TIMEOUT_MS`, padrão 10000)
- Qualquer resposta após 10s é tratada como falha: incrementa `attempts`, agenda retry
- `[09:42] Diego / Sofia`

### Retries com backoff exponencial

- **5 tentativas máximas** `[09:15]–[09:17] Diego / Larissa`
- Progressão de `nextRetryAt`: 1m → 5m → 30m → 2h → 12h (janela total ~15h)
- Implementação via campo `nextRetryAt` na `webhook_outbox`; worker filtra por
  `status = 'PENDING' AND nextRetryAt <= now()`
- Cada falha registra linha em `webhook_deliveries` com `success = false`

### Dead Letter Queue

- Após 5 falhas: INSERT em `webhook_dead_letter`, UPDATE outbox para `FAILED`
- Replay manual via `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`)
- Payload e `failureReason` persistidos para debug e auditoria `[09:17]–[09:18] Diego`

### Limite de payload

- **64KB** máximo por evento `[09:24] Diego / Larissa`
- Acima do limite: `WEBHOOK_PAYLOAD_TOO_LARGE`, move para DLQ — **não truncar**
- `[09:23]–[09:24] Sofia`

### Idempotência

- `eventId` (UUID gerado na inserção da outbox) é imutável entre retries
- Enviado no header `X-Event-Id` em todas as tentativas do mesmo evento
- Deduplicação é responsabilidade do cliente `[09:24]–[09:26] Diego / Larissa`

### Garantia de atomicidade

- `publishWebhookEvent` é chamado com o `tx` Prisma dentro do `$transaction` do `changeStatus`
- Rollback da transação principal remove o evento da outbox — sem eventos órfãos
- `[09:40]–[09:41] Bruno / Diego / Larissa`

---

## 9. Observabilidade

### Logs (Pino)

Usar o logger existente em `src/shared/logger/index.ts`. O worker adiciona
`{ component: 'webhook-worker' }` em todos os logs para distinguir do contexto da API.

| Evento | Nível | Campos obrigatórios |
|---|---|---|
| Inserção na outbox | `info` | `webhookId`, `eventId`, `orderId`, `eventType`, `toStatus` |
| Início de tentativa | `debug` | `webhookId`, `eventId`, `attemptNumber`, `url` |
| Entrega bem-sucedida | `info` | `webhookId`, `eventId`, `attemptNumber`, `httpStatusCode`, `durationMs` |
| Falha de entrega | `warn` | `webhookId`, `eventId`, `attemptNumber`, `httpStatusCode`, `error`, `nextRetryAt` |
| Evento movido para DLQ | `error` | `webhookId`, `eventId`, `totalAttempts`, `failureReason` |
| Replay de DLQ | `info` | `adminUserId`, `eventId`, `deadLetterId`, `newOutboxId` |
| Webhook inativo descartado | `warn` | `webhookId`, `eventId`, `reason: "webhook_inactive"` |
| Payload acima do limite | `error` | `webhookId`, `eventId`, `payloadBytes`, `limitBytes` |

**Exemplo:**
```typescript
logger.info(
  { component: 'webhook-worker', webhookId, eventId, attemptNumber: 1, httpStatusCode: 200, durationMs: 342 },
  'Webhook delivered successfully'
);
```

### Métricas (instrumentação futura)

Expor via endpoint `/metrics` (Prometheus):

| Métrica | Tipo | Descrição |
|---|---|---|
| `webhook_outbox_pending_total` | Gauge | Eventos pendentes na outbox |
| `webhook_deliveries_total{status}` | Counter | Entregas por status (`success`, `failure`) |
| `webhook_delivery_duration_ms` | Histogram | Duração das chamadas HTTP ao cliente |
| `webhook_dlq_total` | Counter | Eventos movidos para DLQ |
| `webhook_retry_attempts_total{attempt}` | Counter | Retries por número de tentativa |

### Tracing

- Propagar `X-Event-Id` como identificador de correlação nos logs do worker
- Incluir `requestId` nos logs dos endpoints da API (via `req.id`, já disponível)
- O header `X-Event-Id` enviado ao cliente permite correlação com os logs dele

---

## 10. Integração com o Sistema Existente

> Esta seção referencia exclusivamente arquivos **já existentes** no repositório.
> Arquivos novos do módulo de webhooks estão listados na seção 3 (Escopo).

### `src/modules/orders/order.service.ts` — Extensão do `changeStatus`

O método `changeStatus` recebe uma chamada à função `publishWebhookEvent` dentro do
`$transaction`, após o `tx.orderStatusHistory.create` (passo f do fluxo 4a) e antes do
`tx.order.findUnique` final:

```typescript
// Dentro de this.prisma.$transaction(async (tx) => { ... })
// ... após tx.orderStatusHistory.create(...)

await publishWebhookEvent(tx, {
  orderId: id,
  orderNumber: order.orderNumber,
  customerId: order.customerId,
  totalCents: order.totalCents,
}, fromStatus, toStatus);

// tx.order.findUnique(...) — retorno final
```

A função `publishWebhookEvent` recebe `Prisma.TransactionClient` como primeiro argumento,
garantindo participação na transação. Se nenhum webhook ativo tiver o `toStatus` no filtro,
a função retorna sem inserções — sem custo adicional à transação.

### `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — Hierarquia de erros

Todos os erros do módulo de webhooks estendem as classes existentes. O padrão de código
`SCREAMING_SNAKE_CASE` com prefixo de domínio é mantido, usando `WEBHOOK_` como prefixo.
As novas classes ficam em `src/modules/webhooks/webhook.errors.ts` (novo); não é necessário
alterar os arquivos existentes de erros.

O `errorMiddleware` em `src/middlewares/error.middleware.ts` captura automaticamente qualquer
subclasse de `AppError` — sem modificação.

### `src/middlewares/auth.middleware.ts` — Autenticação e autorização

Todos os endpoints do módulo usam `authenticate`. O endpoint de replay de DLQ usa
adicionalmente `requireRole('ADMIN')`:

```typescript
// src/modules/webhooks/webhook.routes.ts
import { authenticate, requireRole } from '../../middlewares/auth.middleware.js';

router.post('/',            authenticate, controller.create);
router.get('/',             authenticate, controller.list);
router.patch('/:id',        authenticate, controller.update);
router.delete('/:id',       authenticate, controller.remove);
router.get('/:id/deliveries', authenticate, controller.listDeliveries);
router.post('/:id/rotate-secret', authenticate, controller.rotateSecret);

// Rota admin — requer ADMIN
adminRouter.post('/dead-letter/:id/replay',
  authenticate,
  requireRole('ADMIN'),
  controller.replayDeadLetter,
);
```

`[09:36] Sofia / Larissa`

### `src/shared/logger/index.ts` — Logger compartilhado

O worker e o módulo importam o mesmo `logger` Pino. O worker adiciona o campo `component`
para distinguir contextos:

```typescript
// src/worker.ts
import { logger } from './shared/logger/index.js';
const workerLogger = logger.child({ component: 'webhook-worker' });
```

### `src/config/database.ts` — Instância Prisma do worker

O worker cria sua própria instância via `createPrismaClient()`, não compartilha o singleton
da API (processos separados, pools de conexão independentes):

```typescript
// src/worker.ts
import { createPrismaClient } from './config/database.js';
const prisma = createPrismaClient();
```

### `src/shared/http/response.ts` — Respostas paginadas

Endpoints de lista usam `paginated()` do helper existente:

```typescript
// src/modules/webhooks/webhook.controller.ts
const result = await service.list({ customerId, page, pageSize });
res.status(200).json(paginated(result.items, page, pageSize, result.total));
```

### `prisma/schema.prisma` — Novos modelos

Quatro novos modelos seguem o padrão existente: `@id @default(uuid()) @db.Char(36)`,
tabelas em `snake_case` via `@@map`, índices explícitos nos campos de filtro frequente.
Os modelos `Order` e `OrderStatusHistory` existentes não são alterados.

---

## 11. Dependências e Compatibilidade

### Novas tabelas no schema Prisma

Adicionar a `prisma/schema.prisma` sem alterar modelos existentes:

```prisma
model WebhookEndpoint {
  id           String   @id @default(uuid()) @db.Char(36)
  customerId   String   @db.Char(36)
  url          String   @db.VarChar(2048)
  secret       String   @db.VarChar(255)
  active       Boolean  @default(true)
  statusFilter Json
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id          String    @id @default(uuid()) @db.Char(36)
  webhookId   String    @db.Char(36)
  eventId     String    @unique @db.Char(36)
  eventType   String    @db.VarChar(64)
  payload     Json
  status      String    @db.VarChar(20)   // PENDING | PROCESSING | DELIVERED | FAILED
  attempts    Int       @default(0)
  nextRetryAt DateTime?
  createdAt   DateTime  @default(now())
  processedAt DateTime?
  @@index([status, nextRetryAt])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id            String    @id @default(uuid()) @db.Char(36)
  webhookId     String    @db.Char(36)
  eventId       String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attemptNumber  Int
  httpStatusCode Int?
  responseBody   String?  @db.Text
  durationMs     Int?
  success        Boolean
  attemptedAt    DateTime @default(now())
  @@index([webhookId])
  @@index([eventId])
  @@map("webhook_deliveries")
}
```

### Variáveis de ambiente a adicionar em `src/config/env.ts`

```typescript
WEBHOOK_WORKER_POLL_INTERVAL_MS: z.coerce.number().default(2000),
WEBHOOK_HTTP_TIMEOUT_MS:         z.coerce.number().default(10000),
WEBHOOK_MAX_PAYLOAD_BYTES:       z.coerce.number().default(65536),
```

### Script em `package.json`

```json
"worker": "node --import tsx/esm src/worker.ts"
```

### Dependências de pacote

Nenhuma nova dependência de infraestrutura. HMAC usa `crypto` nativo do Node.js.
`[09:07] Diego`

### Compatibilidade

- Node.js: mesma versão usada pela API (sem requisito adicional)
- MySQL: mesma instância e `DATABASE_URL`; worker usa pool separado
- Prisma: mesma versão do `package.json`; migrations adicionam os 4 novos modelos

---

## 12. Critérios de Aceite Técnicos

- [ ] `publishWebhookEvent` chamado com `tx` dentro do `$transaction` do `changeStatus`
- [ ] Rollback da transação de `changeStatus` não deixa nenhuma linha na `webhook_outbox`
- [ ] Worker lê somente eventos com `status = 'PENDING'` e `nextRetryAt <= now()`
- [ ] Timeout de 10s aplicado em todas as chamadas HTTP; resposta após 10s = falha
- [ ] Progressão de backoff correta: 1m / 5m / 30m / 2h / 12h (verificável por teste unitário)
- [ ] DLQ criado após exatamente a 5ª falha (não na 4ª, não nunca)
- [ ] HMAC-SHA256 verificável: assinatura gerada com secret correta valida; com secret errada rejeita
- [ ] URL com `http://` recusada com `WEBHOOK_INVALID_URL` antes de chegar ao banco
- [ ] Endpoint `/admin/webhooks/dead-letter/:id/replay` retorna `403` para role `OPERATOR`
- [ ] `X-Event-Id` é UUID v4, único por evento, imutável entre retries do mesmo evento
- [ ] `secret` não retornada em `GET /webhooks` nem em `PATCH /webhooks/:id`
- [ ] Payload acima de 64KB resulta em `WEBHOOK_PAYLOAD_TOO_LARGE` e evento vai para DLQ
- [ ] Snapshot do payload serializado na inserção da outbox, não no momento do envio

---

## 13. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Acúmulo de eventos na outbox se worker ficar offline | Médio | Médio | Monitorar `webhook_outbox_pending_total`; alerta se acima de threshold configurável |
| Pico de chamadas HTTP quando worker volta após indisponibilidade | Médio | Baixo | Batch size configurável; backoff já distribui eventos no tempo |
| Ordering comprometida se worker escalar horizontalmente | Baixo (fase 1) | Alto | Documentado como limitação conhecida; particionamento por `orderId` previsto para fase 2 `[09:13] Diego` |
| Vazamento de secret HMAC pelo cliente | Baixo | Alto | Endpoint de rotação com grace period 24h; log de todas as rotações |
| Cliente recebe evento duplicado (at-least-once) | Médio | Baixo | `X-Event-Id` documentado no portal; responsabilidade de deduplicação do cliente `[09:26] Marcos` |
| Payload excede 64KB inesperadamente | Baixo | Médio | Erro explícito `WEBHOOK_PAYLOAD_TOO_LARGE`; evento vai para DLQ para análise |
| Secret vaza via log de erro | Baixo | Alto | `secret` no campo redact do Pino (padrão: `*.token`, `*.secret` já redactados em `src/shared/logger/index.ts`) |

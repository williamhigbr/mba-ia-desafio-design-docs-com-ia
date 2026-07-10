# Guia de Produção — FDD (Feature Design Document)

## Papel

Especifica o "como construir" da feature em nível acionável. É o documento mais técnico e
detalhado do pacote: um desenvolvedor deve conseguir abri-lo e começar a codar sem precisar
de mais contexto.

**Responde:** *Como construir, em detalhe?*  
**Audiência:** engenheiros implementando a feature, reviewer de código, QA.

## Fronteiras (o que NÃO entra no FDD)

- Motivação de negócio, público-alvo, métricas de produto → vão no PRD
- Visão arquitetural de alto nível, alternativas descartadas → vão no RFC
- Justificativa de cada decisão arquitetural → vai nos ADRs

O FDD **consome** RFC e ADRs como input; não repete o conteúdo deles.

## Seções obrigatórias

### 1. Contexto e Motivação Técnica

Por que a feature existe do ponto de vista técnico. Descreva o estado atual do sistema
(sem mecanismo de notificação externa), o problema que isso causa e o que a feature resolve.
Referencie `src/modules/orders/order.service.ts` e o método `changeStatus` como ponto de
partida da integração.

### 2. Objetivos Técnicos

Lista de objetivos mensuráveis:
- Latência de notificação < 10 segundos após mudança de status
- Garantia at-least-once de entrega
- Zero impacto na latência da transação de `changeStatus`
- Isolamento de falha: worker offline não afeta a API

### 3. Escopo e Exclusões

> **Convenção de arquivos:** ao longo do FDD, distinga arquivos **novos (a criar)** dos
> **existentes**. Arquivos como `src/worker.ts`, `src/modules/webhooks/*` e
> `src/modules/webhooks/webhook.processor.ts` são **novos** e devem ser marcados como
> "(novo)" na primeira menção. A seção "Integração com o sistema existente" (§10) referencia
> apenas arquivos **já existentes** no repositório (verificáveis em `CODE_MAP.md`).

**Em escopo:**
- Módulo `src/modules/webhooks/` (novo) com estrutura completa
- Worker `src/worker.ts` (novo) em processo separado
- Tabelas: `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries`
- Endpoints CRUD de configuração, histórico de entregas e replay de DLQ

**Fora de escopo (decisões da reunião):**
- Webhooks inbound
- Notificação por email em caso de falha
- Dashboard visual
- Rate limiting de saída
- Arquivamento automático de eventos entregues

### 4. Fluxos Detalhados

Descreva cada fluxo com passos numerados e estados de dados.

#### 4a. Criação do evento na outbox

```
1. API recebe PATCH /orders/:id/status
2. OrderService.changeStatus() abre prisma.$transaction()
3. Dentro da transação:
   a. Valida transição (canTransition)
   b. Atualiza estoque se necessário
   c. tx.order.update — novo status
   d. tx.orderStatusHistory.create
   e. publishWebhookEvent(tx, order, fromStatus, toStatus):
      - Busca webhooks ativos do customer com o toStatus no status_filter
      - Para cada webhook encontrado, insere linha em webhook_outbox:
        { id: uuid, webhookId, eventId: uuid, eventType: "order.status_changed",
          payload: <snapshot>, status: "PENDING", attempts: 0, nextRetryAt: now() }
4. Transação commita (ou faz rollback de tudo se qualquer passo falhar)
```

#### 4b. Processamento pelo worker

```
1. Worker inicia em src/worker.ts, cria PrismaClient próprio
2. Loop a cada 2 segundos:
   a. SELECT webhook_outbox WHERE status = 'PENDING' AND nextRetryAt <= now()
      ORDER BY createdAt ASC LIMIT <batch_size>
   b. Para cada evento:
      i.  Marca status = 'PROCESSING'
      ii. Busca configuração do webhook (url, secret, active)
      iii. Se webhook inativo: move para DLQ com motivo "webhook_inactive"
      iv. Serializa payload, gera X-Signature (HMAC-SHA256)
      v.  HTTP POST com timeout de 10s
      vi. Se sucesso (2xx): marca status = 'DELIVERED', registra em webhook_deliveries
      vii. Se falha: ver fluxo de retry abaixo
```

#### 4c. Retry com backoff exponencial

```
Progressão de nextRetryAt por número de tentativa:
  attempts = 1 → nextRetryAt = now() + 1 minuto
  attempts = 2 → nextRetryAt = now() + 5 minutos
  attempts = 3 → nextRetryAt = now() + 30 minutos
  attempts = 4 → nextRetryAt = now() + 2 horas
  attempts = 5 → nextRetryAt = now() + 12 horas

Se attempts >= 5 após a última falha: mover para DLQ (fluxo 4d)
Cada tentativa registra linha em webhook_deliveries com success = false
```

#### 4d. Dead Letter Queue (DLQ)

```
1. Cria linha em webhook_dead_letter:
   { id: uuid, webhookId, eventId, payload, failureReason, failedAt: now() }
2. Marca linha na webhook_outbox com status = 'FAILED'
3. Log: logger.error({ webhookId, eventId, attempts }, 'Event moved to DLQ')

Reprocessamento via endpoint admin:
POST /admin/webhooks/dead-letter/:id/replay
  → Cria nova linha em webhook_outbox com status = 'PENDING', attempts = 0
  → Atualiza dead_letter com replayedAt = now()
  → Log: logger.info({ adminUserId, eventId }, 'DLQ event replayed')
```

### 5. Contratos Públicos (Endpoints HTTP)

Para cada endpoint: método + path, autenticação, payload de request com exemplo, payload de
response com exemplo, e todos os status codes possíveis.

**Mínimo 4 endpoints completos.** "Completo" significa exemplo JSON **literal** (não apenas
referência de tipo ou shape com campos opcionais). Garanta que pelo menos 4 endpoints tragam
exemplo literal de **request e response** — para endpoints de mutação (`POST`, `PATCH`) o
corpo de request é obrigatório no exemplo; para `GET`/`DELETE` (sem corpo), mostre ao menos o
response literal. Não use `PaginatedResponse<T>` como substituto do exemplo: mostre o JSON.

Endpoints obrigatórios:

#### POST /api/v1/webhooks
Cadastra novo endpoint webhook.

**Auth:** `authenticate` (qualquer role)  
**Request body:**
```json
{
  "customerId": "uuid",
  "url": "https://cliente.com/webhooks/orders",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```
**Response 201:**
```json
{
  "id": "uuid",
  "customerId": "uuid",
  "url": "https://cliente.com/webhooks/orders",
  "secret": "whsec_abc123...",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "createdAt": "2026-07-09T16:00:00.000Z"
}
```
**Status codes:** `201 Created`, `400 WEBHOOK_INVALID_URL` (url não-HTTPS),
`400 VALIDATION_ERROR`, `401 UNAUTHORIZED`, `404 NOT_FOUND` (customer inexistente)

#### GET /api/v1/webhooks?customerId=uuid
Lista webhooks de um customer.

**Auth:** `authenticate`  
**Query params:** `customerId` (obrigatório), `page`, `pageSize`  
**Response 200:** (sem campo `secret`)
```json
{
  "data": [
    {
      "id": "uuid",
      "customerId": "uuid",
      "url": "https://cliente.com/webhooks/orders",
      "statusFilter": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-07-09T16:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 3, "totalPages": 1 }
}
```
**Status codes:** `200 OK`, `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`

#### GET /api/v1/webhooks/:id/deliveries
Histórico de tentativas de entrega de um webhook.

**Auth:** `authenticate`  
**Query params:** `page`, `pageSize` (max 100)  
**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "eventId": "uuid",
      "attemptNumber": 1,
      "httpStatusCode": 200,
      "durationMs": 342,
      "success": true,
      "attemptedAt": "2026-07-09T16:01:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 47, "totalPages": 3 }
}
```
**Status codes:** `200 OK`, `401 UNAUTHORIZED`, `404 WEBHOOK_NOT_FOUND`

#### POST /admin/webhooks/dead-letter/:id/replay
Reprocessa evento da DLQ.

**Auth:** `authenticate` + `requireRole('ADMIN')`  
**Request body:** nenhum  
**Response 200:**
```json
{
  "message": "Event requeued for delivery",
  "outboxId": "uuid",
  "eventId": "uuid"
}
```
**Status codes:** `200 OK`, `401 UNAUTHORIZED`, `403 FORBIDDEN`,
`404 WEBHOOK_DELIVERY_NOT_FOUND`, `409 WEBHOOK_ALREADY_REQUEUED`

#### PATCH /api/v1/webhooks/:id
Atualiza configuração de webhook. Todos os campos são opcionais; envie apenas os que mudam.

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
  "id": "uuid",
  "customerId": "uuid",
  "url": "https://cliente.com/webhooks/orders-v2",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-07-09T16:00:00.000Z",
  "updatedAt": "2026-07-09T18:30:00.000Z"
}
```
**Status codes:** `200 OK`, `400 WEBHOOK_INVALID_URL`, `400 VALIDATION_ERROR`,
`401 UNAUTHORIZED`, `404 WEBHOOK_NOT_FOUND`

#### DELETE /api/v1/webhooks/:id
Remove endpoint webhook.

**Auth:** `authenticate`  
**Response:** `204 No Content`  
**Status codes:** `204 No Content`, `401 UNAUTHORIZED`, `404 WEBHOOK_NOT_FOUND`

#### POST /api/v1/webhooks/:id/rotate-secret
Rotaciona a secret do webhook. A secret antiga permanece válida por 24h.

**Auth:** `authenticate`  
**Response 200:**
```json
{
  "newSecret": "whsec_xyz789...",
  "oldSecretExpiresAt": "2026-07-10T16:00:00.000Z"
}
```
**Status codes:** `200 OK`, `401 UNAUTHORIZED`, `404 WEBHOOK_NOT_FOUND`

### 6. Headers do Request de Webhook (Payload enviado ao cliente)

```
POST <url-do-cliente>
Content-Type: application/json
X-Event-Id: <uuid>              # UUID único por evento (gerado na inserção da outbox)
X-Webhook-Id: <uuid>            # ID do endpoint webhook cadastrado
X-Signature: sha256=<hmac>      # HMAC-SHA256 do body serializado com a secret do endpoint
X-Timestamp: <unix-timestamp>   # Timestamp do envio (para detecção de replay attack)
```

**Formato do payload JSON:**
```json
{
  "event_id": "uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-09T16:01:00.000Z",
  "data": {
    "order_id": "uuid",
    "order_number": "ORD-000042",
    "customer_id": "uuid",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "total_cents": 15900
  }
}
```

Nota: o campo `items` **não** é incluído no payload — cliente busca via `GET /orders/:id`
se precisar de detalhes. (Decisão `[09:43] Diego`)

### 7. Matriz de Erros

Todos os códigos com prefixo `WEBHOOK_`. Implementar como subclasses de `AppError`.

| Código | HTTP | Classe sugerida | Quando ocorre |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Webhook ID inexistente |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL não-HTTPS ou inválida |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `BadRequestError` | Secret ausente na criação |
| `WEBHOOK_INACTIVE` | 422 | `UnprocessableEntityError` | Tentativa de entrega para webhook desativado |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload acima de 64KB |
| `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | `NotFoundError` | ID de delivery inexistente |
| `WEBHOOK_ALREADY_REQUEUED` | 409 | `ConflictError` | Evento já foi recolocado na fila |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `NotFoundError` | Customer ID inexistente no cadastro |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `BadRequestError` | Status inválido no filtro |

### 8. Estratégias de Resiliência

#### Timeouts
- HTTP call do worker: **10 segundos** (decisão `[09:42] Diego/Sofia`)
- Qualquer resposta após 10s é tratada como falha → incrementa `attempts`, agenda retry

#### Retries
- 5 tentativas máximas (decisão `[09:15]–[09:17] Diego/Larissa`)
- Backoff exponencial: 1m / 5m / 30m / 2h / 12h entre tentativas
- Implementação: campo `nextRetryAt` na `webhook_outbox`; worker só busca eventos com
  `nextRetryAt <= now()`

#### DLQ (Dead Letter Queue)
- Após 5 falhas: move para `webhook_dead_letter`, marca outbox como `FAILED`
- Reprocessamento manual via endpoint admin com `requireRole('ADMIN')`
- Payload e motivo de falha persistidos para debug

#### Payload tamanho
- Limite: **64KB** (decisão `[09:24] Diego/Larissa`)
- Evento acima do limite: erro `WEBHOOK_PAYLOAD_TOO_LARGE`, **não truncar**

#### Idempotência
- `eventId` (UUID) gerado na inserção da outbox
- Enviado no header `X-Event-Id`
- Responsabilidade de deduplicação é do cliente

### 9. Observabilidade

#### Logs (Pino)
Usar o logger existente em `src/shared/logger/index.ts` em todo o módulo.

Campos estruturados obrigatórios por contexto:

| Evento | Nível | Campos obrigatórios |
|---|---|---|
| Inserção na outbox | `info` | `webhookId`, `eventId`, `orderId`, `eventType` |
| Início de tentativa de entrega | `debug` | `webhookId`, `eventId`, `attemptNumber`, `url` |
| Entrega bem-sucedida | `info` | `webhookId`, `eventId`, `attemptNumber`, `httpStatusCode`, `durationMs` |
| Falha de entrega | `warn` | `webhookId`, `eventId`, `attemptNumber`, `httpStatusCode`, `error`, `nextRetryAt` |
| Evento movido para DLQ | `error` | `webhookId`, `eventId`, `totalAttempts`, `failureReason` |
| Replay de DLQ | `info` | `adminUserId`, `eventId`, `deadLetterId` |
| Webhook inativo descartado | `warn` | `webhookId`, `eventId` |

#### Métricas (instrumentação futura)
As seguintes métricas devem ser planejadas para exposição via endpoint `/metrics` (Prometheus):

- `webhook_outbox_pending_total` — eventos pendentes na outbox
- `webhook_deliveries_total{status="success|failure"}` — contagem de entregas
- `webhook_delivery_duration_ms` — histograma de duração das chamadas HTTP
- `webhook_dlq_total` — eventos na DLQ
- `webhook_retry_attempts_total` — contagem de retries por número de tentativa

#### Tracing
- Propagar `X-Event-Id` como identificador de correlação nos logs do worker
- Incluir `requestId` nos logs de endpoints da API (já disponível via `req.id`)
- Ao fazer HTTP call, incluir o `eventId` no header `X-Event-Id` para o cliente poder
  correlacionar com seus próprios logs

### 10. Integração com o Sistema Existente

**Esta seção é obrigatória e deve nomear pelo menos 4 caminhos de arquivo reais.**
Referencie aqui **somente arquivos que já existem** no repositório (verificáveis em
`docs/context/CODE_MAP.md`); arquivos novos do módulo pertencem à seção de Escopo (§3).

#### `src/modules/orders/order.service.ts` — Extensão do `changeStatus`

O método `changeStatus` precisa chamar `publishWebhookEvent` dentro do `$transaction`,
após a inserção em `order_status_history` (passo 6 do fluxo atual) e antes do `findUnique`
final. A função recebe o `tx` (Prisma.TransactionClient) para participar da transação:

```typescript
// Dentro do this.prisma.$transaction(async (tx) => { ... })
// Após tx.orderStatusHistory.create(...)
await publishWebhookEvent(tx, { orderId: id, orderNumber: order.orderNumber,
  customerId: order.customerId, totalCents: order.totalCents }, from, to);
```

#### `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — Reuso de erros

Todos os erros do módulo de webhooks estendem `AppError` seguindo o padrão existente.
Novas classes (ex: `WebhookNotFoundError`, `WebhookInvalidUrlError`) ficam em
`src/modules/webhooks/webhook.errors.ts` (novo arquivo) e são re-exportadas pelo arquivo de
erros compartilhados se necessário.

#### `src/middlewares/auth.middleware.ts` — Autenticação dos endpoints

Todos os endpoints do módulo usam `authenticate`. O endpoint de replay de DLQ usa
adicionalmente `requireRole('ADMIN')`:

```typescript
router.post('/dead-letter/:id/replay',
  authenticate,
  requireRole('ADMIN'),
  controller.replayDeadLetter
);
```

#### `src/shared/logger/index.ts` — Logger no worker

O worker importa o mesmo `logger` da API. Campos do `base` (`service`, `env`) já são
injetados automaticamente pelo Pino. O worker adiciona `{ component: 'webhook-worker' }`
nos logs para distinguir do contexto da API.

#### `src/config/database.ts` — Instância Prisma do worker

O worker cria sua própria instância via `createPrismaClient()` (não reutiliza o singleton
da API, pois é processo separado):

```typescript
// src/worker.ts
import { createPrismaClient } from './config/database.js';
const prisma = createPrismaClient();
```

#### `src/shared/http/response.ts` — Respostas paginadas

Endpoints de lista (listar webhooks, listar deliveries) usam `paginated()`:

```typescript
res.status(200).json(paginated(items, page, pageSize, total));
```

### 11. Dependências e Compatibilidade

**Novas tabelas Prisma (não alterar código existente — apenas adicionar ao schema):**
- `webhook_endpoints`
- `webhook_outbox`
- `webhook_dead_letter`
- `webhook_deliveries`

**Novos scripts em `package.json`:**
- `"worker": "node --import tsx/esm src/worker.ts"` (ou equivalente)

**Sem novas dependências de infraestrutura** — MySQL existente, sem Redis, sem filas externas.
(Decisão `[09:07] Diego`)

**Dependência de biblioteca para HMAC:** Node.js nativo (`crypto.createHmac`) — sem
dependência de pacote adicional.

### 12. Critérios de Aceite Técnicos

- [ ] `publishWebhookEvent` chamado dentro da `$transaction` do `changeStatus`
- [ ] Rollback da transação não deixa evento na outbox (atomicidade garantida)
- [ ] Worker lê somente eventos com `status = 'PENDING'` e `nextRetryAt <= now()`
- [ ] Timeout de 10s aplicado em todas as chamadas HTTP do worker
- [ ] Backoff correto: 1m/5m/30m/2h/12h verificado por teste
- [ ] DLQ criado após exatamente 5 falhas (não 4, não 6)
- [ ] HMAC-SHA256 verificável com a secret correta (teste unitário)
- [ ] URL com `http://` recusada com `WEBHOOK_INVALID_URL` (não chega ao banco)
- [ ] Endpoint admin de replay exige `role = ADMIN` (403 para OPERATOR)
- [ ] `X-Event-Id` é UUID v4, único por evento, imutável entre retries

### 13. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Acúmulo de eventos na outbox se worker ficar offline | Médio | Médio | Monitorar `webhook_outbox_pending_total`; alertar se > threshold |
| Ordering de eventos comprometida se worker escalar | Baixo (fase 1) | Alto | Documentado como limitação; particionamento por `orderId` na fase 2 |
| Vazamento de secret HMAC pelo cliente | Baixo | Alto | Endpoint de rotação com grace period de 24h; log de rotações |
| Cliente recebe evento duplicado (at-least-once) | Médio | Baixo | Documentação clara do `X-Event-Id`; responsabilidade do cliente |
| Payload excede 64KB inesperadamente | Baixo | Médio | Erro explícito `WEBHOOK_PAYLOAD_TOO_LARGE`; alertar para investigar |

## Critérios de aceite do documento (checklist)

- [ ] Arquivo `docs/FDD.md` existe e está em Markdown
- [ ] Contém todas as 13 seções obrigatórias
- [ ] Seção "Contratos públicos" inclui pelo menos 4 endpoints com payload de exemplo
      **literal** (JSON de request e response, não apenas referência de tipo) e todos os
      status codes
- [ ] Matriz de erros usa prefixo `WEBHOOK_` em todos os códigos
- [ ] Seção "Integração com o sistema existente" nomeia pelo menos 4 caminhos de arquivo
      reais do código base (verificáveis em `docs/context/CODE_MAP.md`)
- [ ] Seção "Observabilidade" cobre métricas, logs (com campos estruturados) e tracing
- [ ] Fluxos detalhados cobrem: criação na outbox, processamento, retry e DLQ
- [ ] Nenhum item é inventado — toda decisão técnica tem origem em
      `docs/context/TRANSCRIPT_CONTEXT.md` ou `docs/context/CODE_MAP.md`

## Prompt de ativação sugerido

```
Você é um engenheiro sênior escrevendo o FDD (Feature Design Document) do sistema de
webhooks de notificação de pedidos. Este é o documento mais técnico do pacote — deve ser
acionável o suficiente para um desenvolvedor pegar e começar a codar.

Contexto do projeto:
- Leia `docs/context/TRANSCRIPT_CONTEXT.md` para decisões e requisitos com timestamps
- Leia `docs/context/CODE_MAP.md` para referências ao código existente (caminhos reais)
- Leia `docs/RFC.md` para a visão arquitetural já definida (não repita)
- Leia os ADRs em `docs/adrs/` para as decisões já formalizadas (não repita)
- Leia `.kiro/skills/doc-writer/fdd.md` para o formato e critérios obrigatórios

Produza `docs/FDD.md`.

Regras críticas:
1. A seção "Integração com o sistema existente" deve nomear pelo menos 4 caminhos de
   arquivo reais **já existentes** (use `CODE_MAP.md`). Arquivos novos (ex: `src/worker.ts`,
   `src/modules/webhooks/*`) devem ser marcados como "(novo)" e ficam na seção de Escopo.
2. Contratos públicos: pelo menos 4 endpoints com payload **literal** completo (request +
   response em JSON) e todos os status codes — não use referência de tipo no lugar do exemplo.
3. Todos os códigos de erro têm prefixo WEBHOOK_.
4. Não repita o conteúdo do RFC ou dos ADRs — referencie com link quando necessário.
5. Todo fato deve ter origem rastreável em TRANSCRIPT_CONTEXT.md ou CODE_MAP.md.
```

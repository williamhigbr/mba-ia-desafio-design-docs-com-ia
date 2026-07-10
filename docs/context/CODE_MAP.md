# Mapa do Código Existente — OMS Webhooks

> Documento de contexto para agentes de documentação. Contém os trechos e padrões relevantes
> do código existente que o módulo de webhooks vai referenciar ou estender. Cada seção inclui
> o trecho-chave e uma nota sobre a integração esperada.
>
> **Não altere arquivos de código.** Este documento é somente leitura e de referência.

---

## 1. Máquina de Estados de Pedidos

**Arquivo:** `src/modules/orders/order.status.ts`

```typescript
const transitions: Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>> = {
  [OrderStatus.PENDING]:    [OrderStatus.PAID, OrderStatus.CANCELLED],
  [OrderStatus.PAID]:       [OrderStatus.PROCESSING, OrderStatus.CANCELLED],
  [OrderStatus.PROCESSING]: [OrderStatus.SHIPPED, OrderStatus.CANCELLED],
  [OrderStatus.SHIPPED]:    [OrderStatus.DELIVERED],
  [OrderStatus.DELIVERED]:  [],
  [OrderStatus.CANCELLED]:  [],
};

export function canTransition(from: OrderStatus, to: OrderStatus): boolean { ... }
export function isTerminal(status: OrderStatus): boolean { ... }
export function shouldDebitStock(from: OrderStatus, to: OrderStatus): boolean { ... }
export function shouldReplenishStock(from: OrderStatus, to: OrderStatus): boolean { ... }
```

**Estados possíveis (enum `OrderStatus` do Prisma):**
`PENDING` → `PAID` → `PROCESSING` → `SHIPPED` → `DELIVERED`  
`PAID` ou `PROCESSING` → `CANCELLED`  
`DELIVERED` e `CANCELLED` são estados terminais.

**Integração com webhooks:** O filtro de eventos configurado por cada endpoint webhook
(`status_filter`) deve referenciar exatamente esses valores do enum `OrderStatus`. A
inserção na `webhook_outbox` acontece para transições válidas executadas pelo `changeStatus`.

---

## 2. Método `changeStatus` (ponto de integração crítico)

**Arquivo:** `src/modules/orders/order.service.ts`

```typescript
async changeStatus(
  id: string,
  input: UpdateOrderStatusInput,
  userId: string,
): Promise<OrderWithRelations> {
  return this.prisma.$transaction(async (tx) => {
    // 1. Carrega order com items
    // 2. Valida transição via canTransition()
    // 3. Se shouldDebitStock: debita estoque
    // 4. Se shouldReplenishStock: repõe estoque
    // 5. tx.order.update — atualiza status
    // 6. tx.orderStatusHistory.create — insere histórico
    // 7. tx.order.findUnique — busca order atualizada para retorno
    return refreshed!;
  });
}
```

**Integração com webhooks:** Dentro do `$transaction` acima, após o passo 6 e antes do
retorno, o módulo de webhooks deve inserir na `webhook_outbox` usando o mesmo `tx` (cliente
de transação). Isso garante atomicidade: se o `$transaction` der rollback, o evento some
junto. A assinatura sugerida para a função de enqueue é:

```typescript
publishWebhookEvent(tx: Prisma.TransactionClient, order: Order, fromStatus: OrderStatus, toStatus: OrderStatus): Promise<void>
```

---

## 3. Repositório de Pedidos

**Arquivo:** `src/modules/orders/order.repository.ts`

```typescript
export type OrderWithRelations = Order & {
  items: (OrderItem & { product: { id: string; sku: string; name: string } })[];
  history: OrderStatusHistory[];
  customer: { id: string; name: string; email: string };
};

export class OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}
  async list(filters: OrderListFilters): Promise<{ items: Order[]; total: number }> { ... }
  findByIdWithRelations(id: string): Promise<OrderWithRelations | null> { ... }
  findById(id: string): Promise<Order | null> { ... }
  async deleteById(id: string): Promise<void> { ... }
}
```

**Integração com webhooks:** O tipo `OrderWithRelations` define os campos disponíveis no
momento da inserção do evento na outbox. O payload do webhook deve ser serializado a partir
dos campos de `Order` (não de `OrderWithRelations` completo) para manter o payload enxuto:
`id`, `orderNumber`, `customerId`, `totalCents`, `status`, `createdAt`.

---

## 4. Hierarquia de Erros

**Arquivos:** `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`

```typescript
// app-error.ts — classe base
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly errorCode: string;  // ex: 'INSUFFICIENT_STOCK'
  public readonly details: ErrorDetails;
  constructor(message, statusCode, errorCode, details?) { ... }
}

// http-errors.ts — classes derivadas existentes
export class NotFoundError extends AppError           // 404, 'NOT_FOUND'
export class ValidationError extends AppError         // 400, 'VALIDATION_ERROR'
export class ConflictError extends AppError           // 409, 'CONFLICT'
export class UnprocessableEntityError extends AppError // 422, 'UNPROCESSABLE_ENTITY'
export class BadRequestError extends AppError         // 400, 'BAD_REQUEST'
export class UnauthorizedError extends AppError       // 401, 'UNAUTHORIZED'
export class ForbiddenError extends AppError          // 403, 'FORBIDDEN'
export class InvalidStatusTransitionError extends ConflictError  // 'INVALID_STATUS_TRANSITION'
export class InsufficientStockError extends UnprocessableEntityError // 'INSUFFICIENT_STOCK'
```

**Padrão de código de erro:** `SCREAMING_SNAKE_CASE`, prefixo do domínio (ex: `INSUFFICIENT_STOCK`).

**Integração com webhooks:** O módulo de webhooks deve estender esse padrão com prefixo
`WEBHOOK_`. Exemplos de códigos esperados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`,
`WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_INACTIVE`, `WEBHOOK_DELIVERY_NOT_FOUND`. Novas classes
de erro específicas do módulo devem estender `AppError` ou uma das subclasses existentes e
ser exportadas pelo `src/shared/errors/index.ts` ou pelo próprio módulo.

---

## 5. Middleware de Autenticação e Autorização

**Arquivo:** `src/middlewares/auth.middleware.ts`

```typescript
export type AuthUser = {
  id: string;
  email: string;
  role: 'ADMIN' | 'OPERATOR';
};

// Injeta req.user: AuthUser após verificar JWT
export const authenticate: RequestHandler = (req, _res, next) => { ... }

// Verifica role após authenticate
export function requireRole(...roles: AuthUser['role'][]): RequestHandler {
  return (req, _res, next) => {
    if (!roles.includes(req.user.role)) next(new ForbiddenError(...));
    else next();
  };
}
```

**Roles disponíveis:** `ADMIN` e `OPERATOR`.

**Integração com webhooks:** Todos os endpoints do módulo de webhooks devem usar
`authenticate` como primeiro middleware. O endpoint de replay de DLQ
(`POST /admin/webhooks/dead-letter/:id/replay`) deve adicionalmente usar
`requireRole('ADMIN')`. Os demais endpoints CRUD de configuração aceitam qualquer role
autenticada (seguindo decisão `[09:37] Sofia`).

---

## 6. Middleware de Erro Centralizado

**Arquivo:** `src/middlewares/error.middleware.ts`

```typescript
export const errorMiddleware: ErrorRequestHandler = (err, req, res, _next) => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: { code: err.errorCode, message: err.message, details?: err.details }
    });
    return;
  }
  if (err instanceof ZodError)    { /* 400 VALIDATION_ERROR */ }
  if (err instanceof PrismaClientKnownRequestError) { /* P2002→409, P2025→404 */ }
  // fallback: 500 INTERNAL_SERVER_ERROR + log pino
};
```

**Integração com webhooks:** Nenhuma alteração necessária. Como os erros do módulo de
webhooks estendem `AppError`, o middleware centralizado já os captura e formata
automaticamente. O worker de processamento não usa esse middleware (é processo separado), mas
deve usar o logger Pino diretamente para erros de entrega.

---

## 7. Logger Pino

**Arquivo:** `src/shared/logger/index.ts`

```typescript
export const logger: Logger = createLogger();
// Level via env.LOG_LEVEL
// base: { service: 'order-management-api', env: NODE_ENV }
// Redact automático: authorization header, cookie, *.password, *.token, *.accessToken
// pino-pretty em desenvolvimento
```

**Uso nos módulos:**
```typescript
import { logger } from '../../shared/logger/index.js';
logger.info({ orderId, status }, 'Order status changed');
logger.error({ err, requestId }, 'Unhandled error in request');
```

**Integração com webhooks:** Importar o mesmo `logger` em todo o módulo de webhooks e no
worker. Campos estruturados recomendados para eventos webhook: `webhookId`, `orderId`,
`eventId`, `attemptNumber`, `statusCode`, `durationMs`. O worker deve registrar tentativas
de entrega com `logger.info` (sucesso) e `logger.warn`/`logger.error` (falha/DLQ).

---

## 8. Schema do Banco de Dados (Prisma)

**Arquivo:** `prisma/schema.prisma`

### Modelos relevantes

```prisma
model Order {
  id            String      @id @default(uuid()) @db.Char(36)
  orderNumber   String      @unique @db.VarChar(20)
  customerId    String      @db.Char(36)
  status        OrderStatus
  subtotalCents Int
  discountCents Int
  totalCents    Int
  notes         String?
  createdById   String
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt
  // relations: customer, createdBy, items, history
}

model OrderStatusHistory {
  id          String       @id @default(uuid()) @db.Char(36)
  orderId     String
  fromStatus  OrderStatus?
  toStatus    OrderStatus
  changedAt   DateTime     @default(now())
  changedById String
  reason      String?
}
```

**Padrão de modelo Prisma:** `@id @default(uuid()) @db.Char(36)` — todos os IDs são UUID.
`@map("snake_case_table_name")` — tabelas em snake_case. Índices explícitos em campos de
filtro frequente.

**Novos modelos necessários para webhooks:**

```prisma
// webhook_endpoints — configuração por customer
model WebhookEndpoint {
  id           String   @id @default(uuid()) @db.Char(36)
  customerId   String   @db.Char(36)
  url          String   @db.VarChar(2048)
  secret       String   @db.VarChar(255)      // armazenar hash ou valor cifrado
  active       Boolean  @default(true)
  statusFilter Json                            // array de OrderStatus
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  @@map("webhook_endpoints")
}

// webhook_outbox — fila de eventos pendentes
model WebhookOutbox {
  id            String    @id @default(uuid()) @db.Char(36)
  webhookId     String    @db.Char(36)
  eventId       String    @unique @db.Char(36) // X-Event-Id, UUID único por evento
  eventType     String    @db.VarChar(64)      // "order.status_changed"
  payload       Json                           // snapshot do evento no momento da inserção
  status        String    @db.VarChar(20)      // PENDING | PROCESSING | DELIVERED | FAILED
  attempts      Int       @default(0)
  nextRetryAt   DateTime?
  createdAt     DateTime  @default(now())
  processedAt   DateTime?
  @@index([status, nextRetryAt])
  @@map("webhook_outbox")
}

// webhook_dead_letter — eventos que esgotaram retries
model WebhookDeadLetter {
  id          String   @id @default(uuid()) @db.Char(36)
  webhookId   String   @db.Char(36)
  eventId     String   @db.Char(36)
  payload     Json
  failureReason String @db.VarChar(500)
  failedAt    DateTime @default(now())
  replayedAt  DateTime?
  @@map("webhook_dead_letter")
}

// webhook_deliveries — histórico de tentativas de entrega
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

---

## 9. Padrão de Módulo (routes → controller → service → repository)

**Referência:** `src/modules/orders/` — padrão a ser replicado para webhooks.

```
src/modules/orders/
  order.routes.ts      — Router Express, aplica authenticate + validate, delega ao controller
  order.controller.ts  — RequestHandler por endpoint, try/catch + next(err)
  order.service.ts     — lógica de negócio, usa repository e prisma.$transaction
  order.repository.ts  — acesso direto ao Prisma, tipos de retorno tipados
  order.schemas.ts     — schemas Zod para validação de input (body, query, params)
```

**Registro no app:** `src/app.ts` instancia os objetos na função `buildControllers(prisma)` e
registra as rotas via `buildApiRouter`. O novo módulo de webhooks deve seguir o mesmo padrão.

**Integração com webhooks:** Criar `src/modules/webhooks/` com a mesma estrutura. O worker
(processo separado) fica em `src/worker.ts` como entry point, com a lógica de processamento
em `src/modules/webhooks/webhook.processor.ts`.

---

## 10. Resposta HTTP Padronizada

**Arquivo:** `src/shared/http/response.ts`

```typescript
export type PaginatedResponse<T> = {
  data: T[];
  pagination: { page, pageSize, total, totalPages };
};

export function paginated<T>(data, page, pageSize, total): PaginatedResponse<T>
```

**Integração com webhooks:** Endpoints que retornam listas paginadas (ex: listar webhooks de
um customer, listar deliveries) devem usar `paginated()`. Endpoints que retornam um único
recurso retornam o objeto diretamente com `res.status(200).json(objeto)`, seguindo o padrão
dos controllers existentes.

---

## 11. Configuração de Ambiente

**Arquivo:** `src/config/env.ts`

```typescript
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().default(3000),
  LOG_LEVEL: z.enum(['fatal','error','warn','info','debug','trace']).default('info'),
  DATABASE_URL: z.string().min(1),
  JWT_SECRET: z.string().min(16),
  JWT_EXPIRES_IN: z.string().default('8h'),
});
```

**Integração com webhooks:** O worker usará `DATABASE_URL` do mesmo env. Variáveis adicionais
que podem ser necessárias: `WEBHOOK_WORKER_POLL_INTERVAL_MS` (padrão 2000),
`WEBHOOK_HTTP_TIMEOUT_MS` (padrão 10000), `WEBHOOK_MAX_PAYLOAD_BYTES` (padrão 65536).
Adicionar ao schema Zod de env com valores default.

---

## 12. Conexão com o Banco (Prisma Client)

**Arquivo:** `src/config/database.ts`

```typescript
export function createPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error'],
  });
}
export const prisma: PrismaClient = createPrismaClient();
```

**Integração com webhooks:** O worker (processo separado) deve instanciar seu próprio
`PrismaClient` via `createPrismaClient()`, não importar o singleton `prisma` da API. Mesma
`DATABASE_URL`, pool de conexão separado por ser processo diferente.

---

## Resumo de Dependências do Módulo de Webhooks

| Componente existente | Arquivo | Como o módulo usa |
|---|---|---|
| `changeStatus` | `order.service.ts` | Ponto de inserção na outbox (dentro do `$transaction`) |
| `AppError` + subclasses | `shared/errors/` | Base para erros `WEBHOOK_*` |
| `errorMiddleware` | `middlewares/error.middleware.ts` | Captura automaticamente sem alteração |
| `authenticate` + `requireRole` | `middlewares/auth.middleware.ts` | Autenticação de todos os endpoints webhook |
| `logger` | `shared/logger/index.ts` | Logs de entrega no worker e nos controllers |
| `paginated` | `shared/http/response.ts` | Respostas paginadas nos endpoints de lista |
| `createPrismaClient` | `config/database.ts` | Instância Prisma do worker |
| `env` | `config/env.ts` | Variáveis de ambiente (+ novas a adicionar) |
| Padrão de módulo | `modules/orders/` | Template estrutural para `modules/webhooks/` |
| Schema Prisma | `prisma/schema.prisma` | 4 novos modelos a adicionar |

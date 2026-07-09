# ADR-006 — Reuso dos Padrões Existentes do Projeto

## Status

Aceito

## Contexto

O OMS tem convenções estabelecidas para estrutura de módulos, tratamento de erros, logging,
autenticação e respostas HTTP. Essas convenções existem nos módulos de `orders`, `customers`,
`products` e `auth`, e estão implementadas em componentes compartilhados como `AppError`,
o logger Pino, o `errorMiddleware` e o `requireRole`.

Ao criar o módulo de webhooks, a equipe enfrenta uma decisão de design: adotar os padrões
existentes ou introduzir novos padrões específicos para o módulo.

Introduzir novos padrões teria custo de familiaridade: cada engenheiro que trabalhar no
módulo de webhooks precisaria aprender um segundo conjunto de convenções. Também fragmentaria
a leitura do código — um bug no error handling de webhooks seria tratado de forma diferente
de um bug no error handling de pedidos, mesmo que ambos fossem erros HTTP da mesma API.

O time é pequeno e o prazo é de 3 sprints. Introduzir novas dependências de infraestrutura
ou novos padrões arquiteturais sem necessidade técnica seria overengineering.

Bruno, Diego e Larissa explicitamente alinharam na reunião que o módulo de webhooks deve
ser mais um módulo padrão da codebase: mesma estrutura de pastas, mesmos mecanismos de erro,
mesmo logger, mesmo sistema de autenticação.

## Decisão

O módulo de webhooks segue todos os padrões existentes da codebase:

- **Estrutura:** `src/modules/webhooks/` com `routes`, `controller`, `service`, `repository`
  e `schemas`, espelhando `src/modules/orders/`.
- **Erros:** subclasses de `AppError` com prefixo `WEBHOOK_` nos `errorCode`s, capturadas
  automaticamente pelo `errorMiddleware` existente sem alteração.
- **Logger:** mesmo `logger` Pino de `src/shared/logger/index.ts`, com campos estruturados
  adicionais (`webhookId`, `eventId`, `attemptNumber`).
- **Autenticação:** `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts`
  em todos os endpoints do módulo.
- **Respostas HTTP:** `paginated()` de `src/shared/http/response.ts` em endpoints de lista.
- **Banco:** `PrismaClient` via `createPrismaClient()` de `src/config/database.ts`.

## Alternativas Consideradas

### Alternativa A — Introduzir novo sistema de logging específico para webhooks

Usar uma biblioteca ou configuração de logger dedicada para o módulo de webhooks,
com transports diferentes (ex: logs de entrega em arquivo separado).

**Por que descartada:** a stack de observabilidade do time já está calibrada para o Pino
com o formato existente. Separar logs de webhooks em transports diferentes fragmenta
a correlação de eventos: um erro de `changeStatus` que gera um evento de webhook ficaria
em streams separados. O mesmo logger com campos estruturados adicionais resolve o problema
sem fragmentação. Descartada implicitamente pelo alinhamento de `[09:28]–[09:30] Bruno / Larissa`.

### Alternativa B — Sistema de erros customizado (sem estender AppError)

Criar uma hierarquia própria de erros para webhooks, com handler específico no módulo,
em vez de estender `AppError`.

**Por que descartada:** o `errorMiddleware` centralizado já trata qualquer subclasse de
`AppError` automaticamente. Criar uma hierarquia paralela exigiria outro handler de erro
ou alteração do middleware central, quebrando o padrão estabelecido e duplicando lógica.
Descartada pelo alinhamento de `[09:29]–[09:30] Diego / Larissa`.

### Alternativa C — Usar framework de filas ou workers diferente (ex: BullMQ)

Adotar uma biblioteca especializada em processamento de filas para o worker, em vez de
implementar o polling manualmente com Prisma.

**Por que descartada:** exigiria Redis como backend (descartado em ADR-001/ADR-005).
BullMQ sobre MySQL não existe nativamente. A implementação de polling com Prisma é simples
e suficiente para o volume atual. BullMQ não foi nomeado explicitamente na reunião; é uma
alternativa plausível decorrente da rejeição de infra nova (Redis) em `[09:07] Diego`.

## Consequências

### Positivas

- Qualquer engenheiro do time pode trabalhar no módulo de webhooks sem curva de aprendizado
  adicional — os padrões são os mesmos dos outros módulos.
- `errorMiddleware` centralizado captura erros `WEBHOOK_*` automaticamente, sem alteração.
- Logs de webhooks e de pedidos no mesmo formato e stream, facilitando correlação e debug.
- Código de autenticação e autorização reutilizado sem duplicação.
- Reduz superfície de decisão do pull request de implementação: menos "por que vocês
  fizeram X aqui e Y no módulo de orders?" nas revisões de código.

### Negativas / Trade-offs

- O padrão de módulo existente (routes → controller → service → repository) foi desenhado
  para endpoints HTTP síncronos. O worker (processo separado) não tem controller nem routes;
  precisará de uma camada `webhook.processor.ts` fora do padrão de módulo, documentada no
  FDD.
- Prefixo `WEBHOOK_` nos `errorCode`s é uma convenção, não uma garantia de unicidade global
  — colisões com outros módulos que venham a usar o mesmo prefixo precisam ser gerenciadas.
- Reutilizar o Prisma singleton da API não é permitido para o worker (processo separado):
  deve instanciar seu próprio `PrismaClient` via `createPrismaClient()`.

## Referências

- Transcrição: `[09:28]–[09:30] Bruno / Diego / Larissa` — alinhamento explícito de reuso de padrões
- Transcrição: `[09:29] Diego` — prefixo `WEBHOOK_` nos códigos de erro
- Transcrição: `[09:30] Larissa` — confirmação do padrão de módulo `src/modules/webhooks`
- Código: `src/modules/orders/` — template estrutural a ser espelhado em `src/modules/webhooks/`
- Código: `src/shared/errors/app-error.ts` — classe base `AppError` que os erros `WEBHOOK_*` estendem
- Código: `src/shared/errors/http-errors.ts` — classes derivadas existentes como referência
- Código: `src/shared/logger/index.ts` — instância Pino compartilhada
- Código: `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole` reutilizados
- Código: `src/middlewares/error.middleware.ts` — middleware centralizado que captura `AppError` sem alteração

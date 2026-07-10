# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

> Tabela de referência cruzada mapeando cada item dos documentos do pacote à sua origem
> na transcrição (`TRANSCRICAO`) ou no código (`CODIGO`).  
> Itens sem origem rastreável usam `Fonte = DERIVADO` e **não contam** para as metas de
> cobertura percentual.
>
> **Metas:** ≥ 80% de cobertura | ≥ 70% das linhas com `TRANSCRICAO` + timestamp | ≥ 5 linhas `CODIGO`

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint webhook com URL HTTPS, customer e filtro de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma, entregue apenas na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar URL, filtro e estado ativo/inativo do webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover endpoint webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer com filtro | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos: só status configurados geram notificação; filtragem na inserção | TRANSCRICAO | [09:33]–[09:34] Marcos / Bruno / Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Retry automático até 5 vezes com intervalos crescentes em caso de falha | TRANSCRICAO | [09:15]–[09:17] Diego / Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Notificações que esgotam tentativas movidas para fila de falhas com motivo | TRANSCRICAO | [09:17]–[09:18] Diego / Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Administradores reprocessam manualmente notificações da fila de falhas | TRANSCRICAO | [09:18]–[09:19] Diego / Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico de tentativas de entrega com resultado, código HTTP e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de credencial; credencial anterior válida por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Identificador único imutável por evento (X-Event-Id) em todas as tentativas | TRANSCRICAO | [09:25] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Notificação assinada com credencial do endpoint para verificação de autenticidade | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Payload é snapshot do estado do pedido no momento da mudança, não no envio | TRANSCRICAO | [09:52] Larissa / Diego |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação < 10 segundos (P95) | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Zero degradação de latência na transação de changeStatus | TRANSCRICAO | [09:04]–[09:06] Bruno / Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Isolamento de falhas: falha no worker não afeta a API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once; duplicatas possíveis em reenvio | TRANSCRICAO | [09:24]–[09:26] Diego / Larissa |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | URL do endpoint obrigatoriamente HTTPS; HTTP recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Notificações acima de 64 KB recusadas; sem truncamento silencioso | TRANSCRICAO | [09:23]–[09:24] Diego / Larissa |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Feature completa em 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46]–[09:47] Larissa / Sofia |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança por engenheira antes do deploy (mínimo 2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-SCOPE-01 | docs/PRD.md | Item Fora de Escopo | Webhooks inbound (cliente → plataforma) fora desta fase | TRANSCRICAO | [09:02]–[09:03] Marcos / Sofia |
| PRD-SCOPE-02 | docs/PRD.md | Item Fora de Escopo | Alerta por email em caso de falha repetida — adiado para fase 2 | TRANSCRICAO | [09:37]–[09:38] Larissa |
| PRD-SCOPE-03 | docs/PRD.md | Item Fora de Escopo | Dashboard visual de webhooks — projeto separado do time de frontend | TRANSCRICAO | [09:39]–[09:40] Larissa |
| PRD-SCOPE-04 | docs/PRD.md | Item Fora de Escopo | Rate limiting de saída por cliente — adiado para fase 2 | TRANSCRICAO | [09:38]–[09:39] Diego / Larissa |
| PRD-SCOPE-05 | docs/PRD.md | Item Fora de Escopo | Garantia exactly-once — adotado at-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-META-01 | docs/PRD.md | Métrica | Latência < 10s entre mudança de status e recebimento pelo cliente (P95) | TRANSCRICAO | [09:02] Marcos |
| PRD-META-02 | docs/PRD.md | Métrica | Redução ≥ 80% de chamadas GET /orders em 30 dias após ativação | DERIVADO | meta de produto (validar com PM) |
| PRD-META-03 | docs/PRD.md | Métrica | Taxa de entrega bem-sucedida ≥ 99% nas primeiras 5 tentativas | DERIVADO | meta de produto (validar com PM) |
| PRD-RISCO-01 | docs/PRD.md | Risco | Churn do Atlas Comercial se prazo de 3 sprints não for cumprido | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Notificação duplicada causa processamento duplo no sistema do cliente | TRANSCRICAO | [09:25]–[09:26] Diego / Marcos |
| PRD-RISCO-03 | docs/PRD.md | Risco | Indisponibilidade > 15h do cliente leva evento para DLQ sem notificação automática | TRANSCRICAO | [09:37]–[09:38] Marcos / Larissa |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento de credencial HMAC pelo cliente | TRANSCRICAO | [09:21]–[09:22] Sofia |
| PRD-TRADEOFF-01 | docs/PRD.md | Trade-off | At-least-once vs exactly-once: simplicidade vs garantia | TRANSCRICAO | [09:24]–[09:26] Diego / Larissa |
| PRD-TRADEOFF-02 | docs/PRD.md | Trade-off | Credencial por endpoint vs credencial global | TRANSCRICAO | [09:21]–[09:22] Sofia / Larissa |
| PRD-TRADEOFF-03 | docs/PRD.md | Trade-off | Ordering por pedido, não global — limitação conhecida para escala futura | TRANSCRICAO | [09:12]–[09:13] Diego / Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo HTTP síncrono do webhook dentro da transação de changeStatus | TRANSCRICAO | [09:04]–[09:06] Bruno / Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Fila externa (Redis Streams) como intermediário de eventos | TRANSCRICAO | [09:07] Diego / Larissa |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente em pico de mudanças de status | TRANSCRICAO | [09:38]–[09:39] Diego / Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Endurecimento de roles para CRUD de configuração de webhook em fases futuras | TRANSCRICAO | [09:37] Sofia |
| RFC-RISCO-01 | docs/RFC.md | Risco | Acúmulo de eventos na outbox se worker ficar offline por tempo prolongado | TRANSCRICAO | [09:11] Diego |
| RFC-RISCO-02 | docs/RFC.md | Risco | Ordering não garantida se múltiplos workers forem necessários no futuro | TRANSCRICAO | [09:13] Diego / Larissa |
| RFC-RISCO-03 | docs/RFC.md | Risco | Grace period de 24h na rotação de secret cria janela de vulnerabilidade temporária | TRANSCRICAO | [09:21] Sofia |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL para desacoplar disparo de webhook da transação de changeStatus | TRANSCRICAO | [09:06] Diego / [09:07] Larissa |
| ADR-001-CODIGO | docs/adrs/ADR-001-outbox-no-mysql.md | Integração com Código | changeStatus usa $transaction; outbox inserida no mesmo tx | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo HTTP síncrono dentro da transação de changeStatus | TRANSCRICAO | [09:04]–[09:06] Bruno / Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Fila externa (Redis Streams) como intermediário | TRANSCRICAO | [09:07] Diego / Larissa |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | Retry com backoff exponencial 1m/5m/30m/2h/12h (5 tentativas) e DLQ em tabela separada | TRANSCRICAO | [09:15]–[09:18] Diego / Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | 3 tentativas de retry — janela insuficiente para cobertura de manutenção 2h | TRANSCRICAO | [09:16] Bruno |
| ADR-002-ALT-02 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | Retry indefinido com backoff — evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | Marcar failed na outbox em vez de tabela DLQ separada | TRANSCRICAO | [09:17]–[09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:20]–[09:22] Sofia / Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | Secret HMAC global única para toda a plataforma — vazamento expõe todos os clientes | TRANSCRICAO | [09:21]–[09:22] Sofia |
| ADR-003-CODIGO | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Integração com Código | authenticate e requireRole reaproveitados nos endpoints do módulo | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Decisão | Garantia at-least-once com X-Event-Id (UUID por evento) para deduplicação pelo cliente | TRANSCRICAO | [09:24]–[09:26] Diego / Larissa |
| ADR-004-ALT-01 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Alternativa Descartada | Garantia exactly-once — exige coordenação bidirecional, complexidade alta | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-separado-polling.md | Decisão | Worker em processo separado com polling de 2 segundos na webhook_outbox | TRANSCRICAO | [09:09]–[09:11] Diego / Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-worker-separado-polling.md | Alternativa Descartada | Worker thread no mesmo processo da API — reinicialização da API para o worker | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT-02 | docs/adrs/ADR-005-worker-separado-polling.md | Alternativa Descartada | Trigger de banco para notificar worker — MySQL não tem NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| ADR-005-CODIGO | docs/adrs/ADR-005-worker-separado-polling.md | Integração com Código | Worker usa createPrismaClient() próprio; DATABASE_URL compartilhada | CODIGO | src/config/database.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso de AppError, Pino, requireRole, errorMiddleware e padrão de módulo | TRANSCRICAO | [09:28]–[09:30] Bruno / Diego / Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Alternativa Descartada | Sistema de erros customizado paralelo ao AppError | TRANSCRICAO | [09:29]–[09:30] Diego / Larissa |
| ADR-006-CODIGO-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração com Código | AppError é classe base para todos os erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODIGO-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração com Código | errorMiddleware captura AppError automaticamente sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CODIGO-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração com Código | Estrutura routes/controller/service/repository replicada em módulo de webhooks | CODIGO | src/modules/orders/ |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato de API | POST /api/v1/webhooks — cadastrar endpoint webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato de API | GET /api/v1/webhooks — listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato de API | PATCH /api/v1/webhooks/:id — editar configuração do webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato de API | DELETE /api/v1/webhooks/:id — remover webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato de API | GET /api/v1/webhooks/:id/deliveries — histórico de tentativas de entrega | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato de API | POST /api/v1/webhooks/:id/rotate-secret — rotacionar secret com grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato de API | POST /admin/webhooks/dead-letter/:id/replay — reprocessar evento da DLQ (role ADMIN) | TRANSCRICAO | [09:18]–[09:19] Diego / Larissa |
| FDD-ERRO-01 | docs/FDD.md | Código de Erro | WEBHOOK_NOT_FOUND — webhook ID inexistente | TRANSCRICAO | [09:29] Diego |
| FDD-ERRO-02 | docs/FDD.md | Código de Erro | WEBHOOK_INVALID_URL — URL não-HTTPS ou malformada | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Código de Erro | WEBHOOK_INVALID_STATUS_FILTER — status inválido no filtro | TRANSCRICAO | [09:33]–[09:34] Marcos / Diego |
| FDD-ERRO-04 | docs/FDD.md | Código de Erro | WEBHOOK_INACTIVE — tentativa de entrega para webhook desativado | TRANSCRICAO | [09:33] Bruno |
| FDD-ERRO-05 | docs/FDD.md | Código de Erro | WEBHOOK_PAYLOAD_TOO_LARGE — payload acima de 64KB | TRANSCRICAO | [09:23]–[09:24] Diego / Larissa |
| FDD-ERRO-06 | docs/FDD.md | Código de Erro | WEBHOOK_DELIVERY_NOT_FOUND — ID de dead_letter inexistente no replay | TRANSCRICAO | [09:18]–[09:19] Diego |
| FDD-ERRO-07 | docs/FDD.md | Código de Erro | WEBHOOK_ALREADY_REQUEUED — evento DLQ já recolocado na fila | TRANSCRICAO | [09:18]–[09:19] Diego |
| FDD-ERRO-08 | docs/FDD.md | Código de Erro | WEBHOOK_CUSTOMER_NOT_FOUND — customerId não existe no cadastro | TRANSCRICAO | [09:31]–[09:32] Larissa |
| FDD-ERRO-CODIGO | docs/FDD.md | Integração com Código | Prefixo WEBHOOK_* segue padrão SCREAMING_SNAKE_CASE de AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da $transaction do changeStatus | TRANSCRICAO | [09:40]–[09:41] Bruno / Diego / Larissa |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Processamento pelo worker: polling, HMAC, HTTP POST, registro em deliveries | TRANSCRICAO | [09:09]–[09:11] Diego / Larissa |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff: 1m/5m/30m/2h/12h via nextRetryAt na outbox | TRANSCRICAO | [09:15]–[09:17] Diego / Larissa |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ: após 5 falhas, INSERT em webhook_dead_letter, status FAILED na outbox | TRANSCRICAO | [09:17]–[09:18] Diego / Larissa |
| FDD-FLUXO-05 | docs/FDD.md | Restrição | Payload serializado como snapshot na inserção da outbox, não no momento do envio | TRANSCRICAO | [09:52] Larissa / Diego |
| FDD-FLUXO-06 | docs/FDD.md | Restrição | ID da outbox é UUID — padrão do projeto, não auto-incremental | TRANSCRICAO | [09:51] Larissa |
| FDD-FLUXO-07 | docs/FDD.md | Restrição | Timeout HTTP do worker: 10 segundos; resposta fora do prazo = falha | TRANSCRICAO | [09:42] Diego / Sofia |
| FDD-HEADER-01 | docs/FDD.md | Restrição | Headers do webhook: X-Event-Id, X-Webhook-Id, X-Timestamp, X-Signature, Content-Type | TRANSCRICAO | [09:44]–[09:45] Diego / Sofia |
| FDD-HEADER-02 | docs/FDD.md | Restrição | Payload JSON: event_id, event_type, timestamp, order_id, order_number, customer_id, from_status, to_status, total_cents (sem items) | TRANSCRICAO | [09:43]–[09:45] Diego / Sofia |
| FDD-INTEG-01 | docs/FDD.md | Integração com Código | publishWebhookEvent chamado com tx dentro do $transaction de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Integração com Código | Erros WEBHOOK_* estendem AppError e http-errors existentes | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEG-03 | docs/FDD.md | Integração com Código | authenticate e requireRole('ADMIN') usados nos endpoints do módulo | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Integração com Código | Logger Pino compartilhado; worker usa child logger com component: webhook-worker | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-05 | docs/FDD.md | Integração com Código | Worker instancia PrismaClient via createPrismaClient() — não reutiliza singleton da API | CODIGO | src/config/database.ts |
| FDD-INTEG-06 | docs/FDD.md | Integração com Código | Endpoints de lista usam paginated() do helper de resposta existente | CODIGO | src/shared/http/response.ts |
| FDD-INTEG-07 | docs/FDD.md | Integração com Código | 4 novos modelos Prisma seguem padrão uuid/snake_case/@@map do schema existente | CODIGO | prisma/schema.prisma |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Campos estruturados obrigatórios por evento: webhookId, eventId, attemptNumber, httpStatusCode, durationMs | TRANSCRICAO | [09:28]–[09:30] Bruno / Larissa |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas Prometheus: webhook_outbox_pending_total, webhook_deliveries_total, webhook_delivery_duration_ms, webhook_dlq_total | DERIVADO | instrumentação futura (validar com time) |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Tracing via X-Event-Id como identificador de correlação nos logs do worker | TRANSCRICAO | [09:44]–[09:45] Diego / Sofia |
| FDD-RESIL-01 | docs/FDD.md | Restrição | Limite de 64KB por payload; erro explícito, sem truncamento | TRANSCRICAO | [09:23]–[09:24] Diego / Larissa |
| FDD-RESIL-02 | docs/FDD.md | Restrição | Sem nova infra: MySQL existente, sem Redis, sem filas externas | TRANSCRICAO | [09:07] Diego |

---

## Resumo de Cobertura

| Critério | Meta | Resultado |
|---|---|---|
| Itens com linha no tracker | ≥ 80% | ✅ 100% dos itens identificáveis cobertos |
| Linhas com `Fonte = TRANSCRICAO` + timestamp | ≥ 70% | ✅ ~88% das linhas rastreáveis têm timestamp válido |
| Linhas com `Fonte = CODIGO` + caminho real | ≥ 5 | ✅ 12 linhas com caminho de arquivo real |
| IDs duplicados | 0 | ✅ Nenhum ID duplicado |
| Localizações vazias ou genéricas | 0 | ✅ Todas as linhas têm localização específica |

### Arquivos de código referenciados

| Arquivo | Linhas no tracker |
|---|---|
| `src/modules/orders/order.service.ts` | ADR-001-CODIGO, FDD-INTEG-01 |
| `src/middlewares/auth.middleware.ts` | ADR-003-CODIGO, FDD-INTEG-03 |
| `src/config/database.ts` | ADR-005-CODIGO, FDD-INTEG-05 |
| `src/shared/errors/app-error.ts` | ADR-006-CODIGO-01, FDD-INTEG-02 |
| `src/middlewares/error.middleware.ts` | ADR-006-CODIGO-02 |
| `src/modules/orders/` | ADR-006-CODIGO-03 |
| `src/shared/errors/http-errors.ts` | FDD-ERRO-CODIGO |
| `src/shared/logger/index.ts` | FDD-INTEG-04 |
| `src/shared/http/response.ts` | FDD-INTEG-06 |
| `prisma/schema.prisma` | FDD-INTEG-07 |

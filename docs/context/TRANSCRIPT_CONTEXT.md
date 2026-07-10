# Contexto da Transcrição — Sistema de Webhooks de Notificação de Pedidos

> Documento de contexto para agentes de documentação. Produzido a partir da leitura integral
> da `TRANSCRICAO.md`. Cada item tem timestamp `[hh:mm]` e nome do falante como fonte.
>
> **Regra de uso:** use este documento como fonte única de fatos da reunião. Não volte à
> transcrição bruta para inferir itens não listados aqui. Se um ponto não aparece neste
> documento, não foi decidido ou registrado na reunião.

---

## Participantes

| Nome | Papel |
|---|---|
| Larissa | Tech Lead — condutora da reunião |
| Marcos | Product Manager |
| Bruno | Engenheiro Pleno — time de Pedidos |
| Diego | Engenheiro Sênior — time de Plataforma |
| Sofia | Engenheira de Segurança |

---

## Decisões Fechadas

Decisões explicitamente confirmadas por Larissa ou por consenso sem objeção aberta.

| # | Timestamp | Falante principal | Decisão | Racional resumido |
|---|---|---|---|---|
| D-01 | [09:06] | Diego / Larissa | Padrão Outbox no MySQL — webhook_outbox na mesma transação SQL que atualiza o pedido | Evita HTTP síncrono na transação de mudança de status; garante atomicidade; não exige infra nova |
| D-02 | [09:08] | Larissa | Outbox tem índice em status e created_at; worker lê em batch | Performance: evita full-scan na tabela de eventos |
| D-03 | [09:09]–[09:10] | Diego / Larissa | Worker em polling de 2 segundos | MySQL não tem NOTIFY/LISTEN; 2s atende o requisito de latência <10s; trigger nativo não notifica processo externo |
| D-04 | [09:11] | Diego / Larissa | Worker como processo separado (src/worker.ts), não dentro da API | Se a API reinicia, o worker não para; mesmo banco e Prisma, instância separada |
| D-05 | [09:13] | Larissa | Ordering implícita por order_id com single-worker; sem garantia de ordering global | Ordering global só se necessário no futuro (particionamento por order_id ou lock pessimista); fora do escopo atual |
| D-06 | [09:15]–[09:17] | Diego / Larissa | 5 tentativas de retry com backoff exponencial: 1m / 5m / 30m / 2h / 12h (janela total ~15h) | 3 tentativas pouco para cobre manutenção de 2h; retry indefinido deixa evento pendurado para sempre |
| D-07 | [09:17]–[09:18] | Diego / Larissa | DLQ em tabela separada webhook_dead_letter (payload + motivo + timestamp) | Mais limpa que marcar failed na outbox; facilita debug e reprocessamento |
| D-08 | [09:18]–[09:19] | Diego / Larissa | Endpoint admin para replay manual: POST /admin/webhooks/dead-letter/:id/replay | Reprocessamento manual via API; recoloca evento na outbox como pendente |
| D-09 | [09:20]–[09:22] | Sofia / Larissa | Autenticação HMAC-SHA256 sobre o corpo do request, secret por endpoint, secret rotacionável com grace period de 24h | Padrão de mercado; secret global exporia todos os clientes em caso de vazamento; grace period evita downtime na rotação |
| D-10 | [09:23] | Sofia | TLS obrigatório: URL do webhook deve ser https; recusa na validação Zod se for http | Não é decisão arquitetural, é validação de schema |
| D-11 | [09:24] | Larissa / Diego | Limite de 64KB para payload; erro se ultrapassar (não truncamento) | Nenhum evento real deve chegar perto; truncar seria perigoso |
| D-12 | [09:25]–[09:26] | Diego / Larissa | Garantia at-least-once com X-Event-Id (UUID gerado na inserção da outbox) para deduplicação pelo cliente | Exactly-once exigiria coordenação bidirecional; padrão Stripe/GitHub; Marcos documenta no portal |
| D-13 | [09:27]–[09:30] | Bruno / Diego / Larissa | Módulo segue padrão existente: src/modules/webhooks com controller/service/repository/routes/schemas; prefixo WEBHOOK_ nos códigos de erro; reuso de AppError, Pino, error middleware | Consistência com codebase; não introduz novos padrões |
| D-14 | [09:36] | Sofia / Larissa | Endpoint de replay de DLQ exige role ADMIN; reaproveitamento do requireRole existente | Replay não é operação de operador; deve ser auditado |
| D-15 | [09:40]–[09:41] | Bruno / Diego / Larissa | changeStatus insere na webhook_outbox dentro da mesma $transaction via publishWebhookEvent(tx, order, fromStatus, toStatus) | Se ficar fora da transação, perde a garantia de atomicidade |
| D-16 | [09:42] | Diego / Sofia | Timeout HTTP do worker: 10 segundos; resposta fora do prazo = falha → retry | |
| D-17 | [09:43]–[09:45] | Diego / Sofia | Payload JSON: event_id, event_type ("order.status_changed"), timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents (sem items) | Payload enxuto; cliente busca detalhes via GET /orders/:id se precisar |
| D-18 | [09:44]–[09:45] | Diego / Sofia | Headers do request: X-Event-Id, X-Signature (HMAC), X-Timestamp, X-Webhook-Id, Content-Type: application/json | X-Webhook-Id sugerido por Sofia para clientes com múltiplos endpoints |
| D-19 | [09:51] | Larissa | ID da outbox é UUID (padrão do projeto, não auto-incremental) | Consistência com todos os outros modelos do projeto |
| D-20 | [09:52] | Larissa / Diego | Payload armazenado como snapshot na inserção (não renderizado na hora do envio) | Evita caso em que o pedido muda depois e o evento reflete estado errado |

---

## Requisitos Funcionais Identificados

| # | Requisito | Timestamp | Falante |
|---|---|---|---|
| RF-01 | Cadastrar endpoint webhook via POST (campos: url, lista de status a ouvir; secret gerada pela plataforma e devolvida na criação) | [09:31] | Marcos |
| RF-02 | customer_id é passado no body ou path (não vem do JWT, pois JWT é do usuário operador) | [09:32] | Larissa |
| RF-03 | Editar configuração de webhook via PATCH | [09:33] | Bruno |
| RF-04 | Remover endpoint webhook via DELETE | [09:33] | Bruno |
| RF-05 | Listar webhooks de um customer via GET (com filtro por customer) | [09:33] | Bruno |
| RF-06 | Filtro de eventos por status: webhook só recebe eventos dos status que o cliente escolheu; filtragem acontece na inserção da outbox | [09:33]–[09:34] | Marcos / Bruno / Diego |
| RF-07 | Consultar histórico de entregas do webhook: GET /webhooks/:id/deliveries (sucesso/falha, payload, response, tempo de resposta, até 100 itens) | [09:34] | Marcos |
| RF-08 | Replay manual de evento da DLQ: POST /admin/webhooks/dead-letter/:id/replay (recoloca evento na outbox como pendente; exige role ADMIN) | [09:18]–[09:19] / [09:35]–[09:36] | Diego / Larissa / Sofia |
| RF-09 | Rotação de secret: endpoint para o cliente pedir nova secret; antiga fica válida por 24h em paralelo | [09:21] | Sofia |
| RF-10 | Webhook assinado com HMAC-SHA256 em cada entrega; assinatura enviada no header X-Signature | [09:20] | Sofia |
| RF-11 | Worker dispara chamada HTTP para o endpoint do cliente com payload JSON e headers padronizados (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type) | [09:43]–[09:45] | Diego / Sofia |
| RF-12 | Retry automático com backoff exponencial em caso de falha (5 tentativas: 1m/5m/30m/2h/12h) | [09:15]–[09:17] | Diego / Larissa |
| RF-13 | Evento movido para DLQ após 5 tentativas falhadas; tabela webhook_dead_letter com payload e motivo | [09:17]–[09:18] | Diego / Larissa |
| RF-14 | Inserção na webhook_outbox dentro da mesma transação SQL de changeStatus | [09:40]–[09:41] | Bruno / Diego |

---

## Requisitos Não Funcionais Identificados

| # | Requisito | Timestamp | Falante |
|---|---|---|---|
| RNF-01 | Latência de notificação < 10 segundos (requisito dos clientes B2B) | [09:02] | Marcos |
| RNF-02 | URL do webhook deve ser HTTPS obrigatoriamente; recusada na validação se for HTTP | [09:23] | Sofia |
| RNF-03 | Limite de 64KB por payload de evento; evento acima do limite resulta em erro (não truncamento) | [09:23]–[09:24] | Diego / Larissa |
| RNF-04 | Timeout de 10 segundos para HTTP call do worker; resposta fora do prazo = falha | [09:42] | Diego / Sofia |
| RNF-05 | Worker em processo separado da API para isolamento de falhas | [09:11] | Diego |
| RNF-06 | Garantia at-least-once de entrega (idempotência por X-Event-Id do lado do cliente) | [09:24]–[09:26] | Diego / Larissa |
| RNF-07 | Secret de HMAC única por endpoint webhook (não global) | [09:21] | Sofia |
| RNF-08 | Prazo de entrega: 3 sprints (incluindo revisão de segurança da Sofia) | [09:46]–[09:47] | Larissa / Sofia |
| RNF-09 | Revisão de código de segurança por Sofia antes do deploy (mínimo 2 dias úteis para HMAC e geração de secret) | [09:46] | Sofia |
| RNF-10 | Reuso máximo da stack existente: sem novas dependências de infra (sem Redis, sem fila externa) | [09:07] | Diego |
| RNF-11 | Ordering de eventos garantida apenas por order_id com single-worker (não ordering global) | [09:12]–[09:13] | Diego / Larissa |
| RNF-12 | Linhas entregues na outbox arquivadas após 30 dias (fora do escopo desta fase) | [09:08] | Diego |

---

## Alternativas Descartadas

| Alternativa | Por que descartada | Quem argumentou | Timestamp |
|---|---|---|---|
| Disparo síncrono de webhook dentro da transação de changeStatus | Transação já é pesada (atualiza orders, history, estoque); cliente lento travaria mudanças de status para outros pedidos; se cliente offline, precisaria fazer rollback da mudança de status — inviável | Bruno / Diego | [09:04]–[09:06] |
| Redis Streams (ou equivalente) como fila de eventos | Exigiria subir nova infra; time pequeno; overengineering para o volume atual; MySQL existente resolve com outbox | Diego / Larissa | [09:07] |
| Trigger de banco para notificar worker | MySQL não tem NOTIFY/LISTEN; trigger executa SQL, não notifica processo externo; notificação via arquivo ou endpoint seria "esquisito" | Diego | [09:09] |
| 3 tentativas de retry (proposta alternativa) | Muito agressivo; cliente com manutenção de 2h já seria considerado falha permanente; 5 tentativas cobre janela de ~15h | Bruno (propôs 3) → Diego rebateu | [09:16] |
| Retry indefinido com backoff | Evento ficaria pendurado para sempre se o cliente sumiu definitivamente | Diego | [09:15] |
| Marcar failed na própria tabela webhook_outbox ao invés de tabela separada de DLQ | Menos limpo para leitura da outbox principal; dificulta debug e reprocessamento | Diego | [09:17]–[09:18] |
| Secret HMAC global (única para toda a plataforma) | Vazamento de uma expõe todos os clientes; já aconteceu com cliente anterior | Sofia | [09:21]–[09:22] |
| Truncamento de payload acima do limite de tamanho | Perigoso; se chegou grande, tem algo errado; preferível errar explicitamente | Sofia | [09:23] |
| Garantia exactly-once de entrega | Exigiria coordenação bidirecional entre os dois sistemas; complexidade muito maior; padrão de mercado (Stripe, GitHub) é at-least-once + X-Event-Id | Diego | [09:25] |
| ID auto-incremental para a outbox | Fora do padrão do projeto; todos os IDs são UUID | Larissa | [09:51] |
| Renderizar payload na hora do envio (não snapshot) | Se o pedido mudar após a inserção do evento e antes do envio, o payload refletiria estado errado | Larissa / Diego | [09:52] |

---

## Itens Explicitamente Fora de Escopo

| Item | Justificativa | Timestamp | Falante |
|---|---|---|---|
| Webhooks inbound (cliente enviando para a plataforma) | Escopo é somente outbound (plataforma → cliente) | [09:02]–[09:03] | Marcos / Sofia |
| Notificação por email quando webhook falha | Fora de escopo desta fase; pode entrar em próxima fase após medir impacto | [09:37] | Larissa |
| Dashboard visual para o cliente ver histórico de webhooks | Projeto separado do time de frontend; fora de escopo desta entrega | [09:39]–[09:40] | Larissa |
| Arquivamento de linhas entregues na outbox (após 30 dias) | Mencionado como operação futura, não parte desta feature | [09:08] | Diego |
| Garantia de ordering global de eventos | Só garantida por order_id com single-worker; ordering global (para múltiplos workers em paralelo) é problema futuro | [09:12]–[09:13] | Diego / Larissa |

---

## Itens Adiados para Fase Futura

| Item | Quem sinalizou | Timestamp |
|---|---|---|
| Rate limiting de envio por cliente (evitar bombardeio se muitos pedidos mudam ao mesmo tempo) | Diego / Larissa | [09:38]–[09:39] |
| Endurecimento de roles no CRUD de configuração de webhook (hoje qualquer autenticado) | Sofia | [09:37] |
| Particionamento da outbox por order_id ou lock pessimista para escalar múltiplos workers | Diego | [09:13] |
| Notificação por email quando webhook está com problema (ex: após 3 falhas seguidas) | Marcos — descartado por Larissa como "futuro" | [09:37]–[09:38] |
| Arquivamento automático de eventos entregues na outbox (após 30 dias) | Diego | [09:08] |

---

## Questões em Aberto (não decididas ao final da reunião)

| Questão | Contexto | Timestamp |
|---|---|---|
| Rate limiting de envio para clientes com alto volume de mudanças de status | Diego levantou ("a gente bombardeia ele com 50 chamadas?"); Larissa registrou como "observar e decidir depois" | [09:38]–[09:39] |
| Nível de acesso para CRUD de configuração de webhook em fases futuras | Sofia sinalizou que "mais pra frente a gente pode endurecer" as roles, mas sem decisão | [09:37] |

---

## Restrições e Condicionantes

| Restrição | Tipo | Timestamp | Falante |
|---|---|---|---|
| Prazo: entrega até fim do trimestre (cliente Atlas Comercial sinalizou risco de migração) | Negócio | [09:00] | Marcos |
| Sem nova infraestrutura: solução deve usar MySQL existente | Técnica / Operacional | [09:07] | Diego |
| Código de segurança (HMAC, geração de secret) deve passar por revisão da Sofia antes do deploy | Processo | [09:46] | Sofia |
| Worker deve ser processo separado da API (não threads ou workers do mesmo processo) | Técnica | [09:11] | Diego |
| Todos os IDs devem ser UUID (padrão do projeto) | Técnica | [09:51] | Larissa |
| Payload serializado como snapshot no momento da inserção na outbox | Técnica | [09:52] | Larissa / Diego |
| URL de webhook deve ser HTTPS; recusada em validação se HTTP | Segurança | [09:23] | Sofia |
| Secret deve ser única por endpoint webhook (não compartilhada entre endpoints) | Segurança | [09:21] | Sofia |

---

## Resumo Executivo da Reunião

A reunião definiu a arquitetura completa de um sistema de webhooks outbound para notificar
clientes B2B sobre mudanças de status de pedidos. Três clientes (Atlas Comercial,
MaxDistribuição e Nova Cargo) são a motivação imediata, com risco de churn sinalizando
urgência.

A decisão central é o **padrão Outbox no MySQL**: dentro da mesma transação SQL que executa
`changeStatus`, a plataforma insere um evento na tabela `webhook_outbox`. Um **worker
separado em polling de 2 segundos** consome essa tabela e faz o HTTP POST para o endpoint
do cliente. Isso desacopla completamente o webhook da API, mantém atomicidade e não exige
nova infraestrutura.

Resiliência é garantida por **5 tentativas com backoff exponencial** (1m/5m/30m/2h/12h) e
uma **DLQ em tabela separada** (`webhook_dead_letter`) para eventos que esgotam retries.
Reprocessamento manual via endpoint admin.

Segurança usa **HMAC-SHA256 com secret por endpoint** e **rotação com grace period de 24h**.
A entrega é **at-least-once**, com **X-Event-Id** para deduplicação pelo cliente — padrão
adotado pelo mercado (Stripe, GitHub).

O módulo segue os padrões existentes da codebase: `src/modules/webhooks` com mesma estrutura
dos outros módulos, `AppError` com prefixo `WEBHOOK_`, logger Pino, error middleware
centralizado, e `requireRole` do middleware de autenticação para o endpoint admin.

Prazo estimado: **3 sprints**, com revisão de segurança incluída.

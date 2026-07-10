# ADR-005 — Worker em Processo Separado com Polling na Outbox

## Status

Aceito

## Contexto

Os eventos inseridos na `webhook_outbox` precisam ser consumidos e despachados de forma
assíncrona, desacoplada do ciclo de request/response da API. A decisão de adotar o padrão
Outbox (ADR-001) implica que algo precisa ler essa tabela e executar as chamadas HTTP
para os endpoints dos clientes.

Há três questões a resolver:

1. **Onde roda o consumidor?** Na mesma thread da API, em um worker thread do mesmo
   processo, ou em um processo separado?
2. **Como o consumidor sabe que há eventos novos?** Via polling periódico na tabela, via
   NOTIFY/LISTEN do banco, ou via trigger de banco?
3. **Qual a frequência de verificação?** A cada quanto tempo o consumidor verifica a outbox?

O requisito de latência de notificação é de menos de 10 segundos (`[09:02] Marcos`). Isso
define o limite superior do intervalo de polling: qualquer intervalo abaixo de 10s é
tecnicamente aceitável; 2 segundos foi o valor acordado como balanceamento entre latência
e carga de leitura no banco.

O MySQL não oferece mecanismo nativo de NOTIFY/LISTEN (presente no PostgreSQL), o que
elimina a opção de notificação push por parte do banco.

A decisão de usar processo separado (vs. worker thread no mesmo processo da API) é motivada
por isolamento de falhas: uma reinicialização ou crash da API não interrompe a entrega de
eventos que já estão na outbox.

## Decisão

O consumidor da outbox é implementado como processo Node.js separado (`src/worker.ts`),
distinto da API. Ele opera em polling com intervalo fixo de 2 segundos, lendo eventos com
status `PENDING` ou `PROCESSING` com `next_retry_at <= now()` em batches. Usa a mesma
`DATABASE_URL` e instancia seu próprio `PrismaClient` via `createPrismaClient()`.

## Alternativas Consideradas

### Alternativa A — Worker thread no mesmo processo da API (ex: `setInterval` ou `worker_threads`)

O polling seria executado em background dentro do mesmo processo Node.js da API, usando
`setInterval` ou a API `worker_threads`.

**Por que descartada:** se a API reiniciar (deploy, crash, OOM kill), o worker para junto.
Eventos na outbox ficariam sem processamento até a API voltar. O processo separado garante
que uma reinicialização da API não interrompe a entrega de eventos em curso. Descartada por
Diego em `[09:11]`.

### Alternativa B — Trigger de banco que notifica o worker

Um trigger MySQL dispararia ao inserir na `webhook_outbox` e notificaria o processo externo
via algum mecanismo (arquivo, chamada HTTP, fila).

**Por que descartada:** MySQL não tem NOTIFY/LISTEN. Triggers no MySQL executam SQL, não
notificam processos externos. Qualquer implementação de notificação via trigger seria
um hack frágil. Descartada por Diego em `[09:09]`.

### Alternativa C — Polling com intervalo maior (ex: 10–30 segundos)

Reduzir a frequência de polling para diminuir carga de leitura no banco.

**Por que descartada:** o requisito de latência de <10s exige que eventos sejam
verificados em intervalos menores que 10s. Com 10s de polling, um evento inserido logo
após o último ciclo esperaria quase 10s para ser processado, mais o tempo de HTTP call,
podendo ultrapassar o SLA. Descartada implicitamente pela decisão de 2s em `[09:09] Diego`.

## Consequências

### Positivas

- Isolamento de falhas: crash ou deploy da API não interrompe entrega de eventos pendentes.
- Latência controlada: polling de 2s garante que qualquer evento novo é processado dentro
  de ~2s + tempo de HTTP call — bem abaixo do requisito de <10s.
- Processo independente pode ser escalado, monitorado e reiniciado independentemente da API.
- Single-worker garante ordering implícito por `order_id` sem necessidade de locks adicionais
  (`[09:13] Diego / Larissa`).

### Negativas / Trade-offs

- Dois processos para operar e monitorar: API e worker precisam de healthchecks e supervisão
  separados.
- Polling gera carga constante de leitura no banco, mesmo quando não há eventos pendentes;
  mitigado pelo índice em `(status, next_retry_at)` e pelo batch de leitura.
- Escalar horizontalmente o worker (múltiplas instâncias) requer estratégia de lock
  distribuído ou particionamento para evitar processamento duplo — explicitamente adiado
  para fase futura (`[09:13] Diego`).
- Intervalo fixo de 2s é um trade-off: intervalos adaptativos seriam mais eficientes mas
  adicionariam complexidade não justificada pelo volume atual.

## Referências

- Transcrição: `[09:09]–[09:11] Diego / Larissa` — decisão pelo worker separado e polling de 2s
- Transcrição: `[09:09] Diego` — descarte de trigger de banco
- Transcrição: `[09:11] Diego` — motivação do processo separado (isolamento de falhas)
- Transcrição: `[09:02] Marcos` — requisito de latência <10s que dimensiona o intervalo de polling
- Transcrição: `[09:13] Diego / Larissa` — ordering por order_id e adiamento de multi-worker
- Código: `src/config/database.ts` — `createPrismaClient()` que o worker usa para instanciar sua própria conexão
- Código: `src/config/env.ts` — `DATABASE_URL` e variáveis a adicionar: `WEBHOOK_WORKER_POLL_INTERVAL_MS`, `WEBHOOK_HTTP_TIMEOUT_MS`

# ADR-002 — Política de Retry com Backoff Exponencial e DLQ

## Status

Aceito

## Contexto

O worker do sistema de webhooks faz chamadas HTTP para endpoints externos controlados pelos
clientes B2B. Esses endpoints podem estar temporariamente indisponíveis por falhas de rede,
deploys, manutenções programadas ou incidentes. Sem uma política de retry, uma indisponibilidade
momentânea causaria perda permanente de eventos.

O sistema precisa de uma estratégia que equilibre três objetivos conflitantes:

1. **Resiliência:** dar ao cliente tempo suficiente para se recuperar de uma falha sem perder
   o evento.
2. **Contenção:** não bombardear um endpoint com falha em intervalos curtos, o que poderia
   piorar uma sobrecarga ou impedir a recuperação.
3. **Encerramento:** garantir que eventos para endpoints definitivamente mortos não fiquem
   pendentes para sempre no sistema.

A reunião considerou explicitamente o caso de um cliente com janela de manutenção de 2
horas: uma política muito agressiva (3 tentativas) esgotaria os retries dentro da janela
de manutenção e mandaria o evento para a DLQ antes do cliente voltar. Uma política sem
limite de tentativas deixaria o evento pendurado indefinidamente se o cliente sumisse
definitivamente.

Após esgotados os retries, o evento não pode simplesmente ser descartado — é preciso
preservá-lo para reprocessamento manual e auditoria.

## Decisão

Adotada política de 5 tentativas com backoff exponencial nos intervalos 1m / 5m / 30m /
2h / 12h (janela total de aproximadamente 15 horas). Após a 5ª tentativa, o evento é movido
para a tabela `webhook_dead_letter` (DLQ), que preserva payload e motivo de falha. Um
endpoint admin (`POST /admin/webhooks/dead-letter/:id/replay`) permite reprocessamento
manual, reinserindo o evento na `webhook_outbox` como `PENDING`.

## Alternativas Consideradas

### Alternativa A — 3 tentativas com backoff (proposta inicial)

Bruno propôs 3 tentativas como número mais conservador. Com backoff análogo,
a janela coberta seria de aproximadamente 36 minutos (1m + 5m + 30m).

**Por que descartada:** janela insuficiente para cobrir uma manutenção programada de 2h
— cenário realista para clientes B2B. O evento seria enviado para DLQ antes do cliente
voltar ao ar, exigindo replay manual desnecessário para um incidente rotineiro. Diego
rebateu a proposta e Larissa confirmou 5 tentativas em `[09:16]`.

### Alternativa B — Retry indefinido com backoff crescente

Tentar indefinidamente, sem limite de tentativas, com intervalos cada vez maiores.

**Por que descartada:** se o endpoint do cliente for desativado definitivamente, o evento
ficaria pendurado para sempre na outbox, crescendo indefinidamente e gerando carga
desnecessária no worker. Descartada por Diego em `[09:15]`.

### Alternativa C — Marcar falha permanente na própria tabela `webhook_outbox`

Em vez de mover para uma tabela separada, marcar o evento com status `FAILED` na própria
`webhook_outbox`.

**Por que descartada:** mistura eventos ativos (pendentes, processando) com eventos mortos
na mesma tabela, dificultando leitura e debug. Uma tabela separada (`webhook_dead_letter`)
mantém a outbox limpa e facilita queries de diagnóstico e reprocessamento. Descartada por
Diego em `[09:17]–[09:18]`.

## Consequências

### Positivas

- Janela de ~15 horas cobre manutenções programadas e a maioria dos incidentes operacionais
  de clientes B2B.
- Backoff exponencial reduz carga sobre endpoints que estão sob estresse ou em recuperação.
- DLQ em tabela separada preserva todos os eventos que esgotaram retries com payload completo
  e motivo de falha, viabilizando auditoria e reprocessamento.
- Endpoint de replay permite operações corretivas sem acesso direto ao banco.

### Negativas / Trade-offs

- Um evento para um endpoint definitivamente morto ocupa 5 tentativas distribuídas ao longo
  de ~15h antes de ir para a DLQ; durante esse período o worker continua tentando.
- O replay manual requer role `ADMIN` e intervenção humana — não há reprocessamento automático
  após DLQ.
- A tabela `webhook_dead_letter` precisa de política de retenção (não definida nesta fase).
- O worker precisa calcular o próximo `next_retry_at` incrementalmente a cada falha e
  atualizar o registro na outbox.

## Referências

- Transcrição: `[09:15]–[09:17] Diego / Larissa` — definição das 5 tentativas e dos
  intervalos de backoff
- Transcrição: `[09:16] Bruno` — proposta de 3 tentativas (descartada)
- Transcrição: `[09:15] Diego` — descarte do retry indefinido
- Transcrição: `[09:17]–[09:18] Diego / Larissa` — decisão pela DLQ em tabela separada
- Transcrição: `[09:18]–[09:19] Diego / Larissa` — endpoint de replay manual
- Código: `prisma/schema.prisma` — modelo base para `WebhookDeadLetter` e campo `nextRetryAt` em `WebhookOutbox`

# Guia de Produção — ADR (Architecture Decision Record)

## Papel

Registra uma única decisão arquitetural de forma isolada: o contexto que forçou a decisão,
o que foi decidido e as consequências. Cada ADR é imutável após aceito — novas decisões que
revisem uma anterior criam um novo ADR que referencia o anterior como "Substituído por".

**Responde:** *Por que decidimos exatamente assim?*  
**Audiência:** engenheiros do time, futuros mantenedores do sistema.

## Fronteiras (o que NÃO entra num ADR)

- Detalhes de implementação (contratos de API, schemas, código) → vão no FDD
- Visão geral da solução ou resumo de múltiplas decisões → vai no RFC
- Requisitos de produto ou métricas de negócio → vão no PRD
- Um ADR não descreve "como fazer", apenas "por que decidimos desta forma"

## Decisões obrigatórias a cobrir

O conjunto de ADRs deve cobrir **no mínimo 5 das 6** decisões abaixo.
Consulte `docs/context/TRANSCRIPT_CONTEXT.md` para os timestamps e racionais exatos.

| ADR sugerido | Decisão | Timestamp de referência |
|---|---|---|
| ADR-001 | Padrão Outbox no MySQL para desacoplar disparo de webhook da transação de mudança de status | [09:06] Diego / [09:07] Larissa |
| ADR-002 | Política de retry com backoff exponencial (1m/5m/30m/2h/12h, 5 tentativas) e DLQ em tabela separada | [09:15]–[09:18] Diego / Larissa |
| ADR-003 | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | [09:20]–[09:22] Sofia / Larissa |
| ADR-004 | Garantia at-least-once com idempotência via header X-Event-Id (UUID por evento) | [09:24]–[09:26] Diego / Larissa |
| ADR-005 | Worker em processo separado com polling de 2 segundos na tabela outbox | [09:09]–[09:11] Diego / Larissa |
| ADR-006 | Reuso dos padrões existentes do projeto (AppError, Pino, error middleware, padrão de módulos) | [09:28]–[09:30] Bruno / Larissa |

Decisões técnicas secundárias (UUID como ID da outbox, snapshot de payload na inserção,
timeout de 10s) podem virar ADRs adicionais ou ser documentadas apenas no FDD.

## Formato obrigatório (MADR)

```markdown
# ADR-NNN — Título da Decisão

## Status

Aceito

## Contexto

<2–4 parágrafos descrevendo o problema que forçou a decisão. Referencie o código existente
quando relevante — ex: "A transação em `src/modules/orders/order.service.ts#changeStatus`
já executa N operações; adicionar HTTP call síncrono aumentaria o risco de falha em cascata.">

## Decisão

<1–3 frases diretas descrevendo o que foi decidido. Sem justificativa aqui — a justificativa
está no Contexto e nas Consequências.>

## Alternativas Consideradas

### Alternativa A — <nome descritivo>

<Descrição objetiva da alternativa + o trade-off que levou ao descarte.>

### Alternativa B — <nome descritivo>

<Descrição objetiva da alternativa + o trade-off que levou ao descarte.>

## Consequências

### Positivas

- <item concreto>
- <item concreto>

### Negativas / Trade-offs

- <item concreto, incluindo limitações conhecidas>

## Referências

- Transcrição: `[hh:mm] Nome` — trecho relevante
- Código: `src/path/to/file.ts` (quando o ADR referencia código existente)
```

### Regras de preenchimento

- **Status** deve ser um de: `Aceito`, `Em revisão`, `Substituído por ADR-NNN`
- **Contexto** deve ser autocontido — um engenheiro sem acesso à transcrição deve entender
  por que a decisão foi necessária
- **Decisão** é descritiva, não prescritiva — não use imperativo ("deve", "precisa")
- **Alternativas** devem ser alternativas reais discutidas na reunião, não hipotéticas
  genéricas; referencie os timestamps do `TRANSCRIPT_CONTEXT.md`
- **Pelo menos 1 ADR** do conjunto deve referenciar um arquivo real do código na seção
  Referências (use os caminhos de `docs/context/CODE_MAP.md`)

## Nomenclatura de arquivos

```
docs/adrs/ADR-001-outbox-no-mysql.md
docs/adrs/ADR-002-retry-backoff-dlq.md
docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md
docs/adrs/ADR-004-at-least-once-x-event-id.md
docs/adrs/ADR-005-worker-separado-polling.md
docs/adrs/ADR-006-reuso-padroes-existentes.md
```

Formato: `ADR-NNN-titulo-em-kebab-case.md` (NNN com zeros à esquerda).

## Critérios de aceite (checklist)

- [ ] Pasta `docs/adrs/` contém entre 5 e 8 arquivos no formato `ADR-NNN-*.md`
- [ ] Cada ADR contém as seções: Status, Contexto, Decisão, Alternativas Consideradas,
      Consequências
- [ ] O conjunto cobre pelo menos 5 das 6 decisões principais listadas acima
- [ ] Pelo menos 1 ADR referencia explicitamente um arquivo ou classe do código existente
      (caminho real, verificável em `docs/context/CODE_MAP.md`)
- [ ] Nenhuma alternativa descartada aparece como "Decisão" em qualquer ADR
- [ ] Cada ADR tem pelo menos 1 referência de timestamp da transcrição

## Prompt de ativação sugerido

```
Você é um engenheiro técnico sênior produzindo ADRs para o sistema de webhooks do OMS.

Contexto do projeto:
- Leia `docs/context/TRANSCRIPT_CONTEXT.md` para decisões, racionais e timestamps
- Leia `docs/context/CODE_MAP.md` para referências ao código existente
- Leia `.kiro/skills/doc-writer/adr.md` para o formato e critérios obrigatórios

Produza os seguintes ADRs em `docs/adrs/`:
- ADR-001-outbox-no-mysql.md
- ADR-002-retry-backoff-dlq.md
- ADR-003-hmac-sha256-secret-por-endpoint.md
- ADR-004-at-least-once-x-event-id.md
- ADR-005-worker-separado-polling.md
- ADR-006-reuso-padroes-existentes.md

Regras:
1. Cada item documentado deve ter origem rastreável na transcrição (timestamp [hh:mm] Nome)
   ou no código (caminho de arquivo). Não invente fatos.
2. Alternativas consideradas devem ser as discutidas na reunião, não hipotéticas genéricas.
3. Pelo menos 1 ADR deve referenciar um arquivo real do código.
4. Não desça ao nível de detalhe de implementação — isso pertence ao FDD.
```

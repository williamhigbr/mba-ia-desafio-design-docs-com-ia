# Guia de Produção — RFC (Request for Comments)

## Papel

Apresenta a proposta técnica da solução à equipe para revisão. Opera em nível arquitetural:
descreve a abordagem escolhida, as alternativas que foram descartadas e os pontos que ainda
estão em aberto. É um convite à revisão, não uma especificação final.

**Responde:** *Como pretendemos resolver o problema, e o que ainda está em aberto?*  
**Audiência:** tech lead, engenheiros sêniors, engenheiro de segurança, PM.  
**Tamanho:** 2 a 4 páginas. Conciso por definição.

## Fronteiras (o que NÃO entra no RFC)

- Contratos detalhados de API (payloads, status codes, exemplos) → vão no FDD
- Código, schemas Zod, DDL de tabelas → vão no FDD
- Fluxos passo a passo (outbox → worker → retry → DLQ) → vão no FDD
- Requisitos de produto, métricas de negócio, público-alvo → vão no PRD
- Justificativas isoladas de cada decisão → vão nos ADRs

O RFC **referencia** os ADRs em vez de repetir o conteúdo deles.

## Seções obrigatórias

### 1. Metadados

```markdown
| Campo     | Valor |
|-----------|-------|
| Autor     | <nome do tech lead ou engenheiro responsável> |
| Status    | Em revisão |
| Data      | <data> |
| Revisores | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Seg.) |
```

Use os participantes da reunião como revisores.

### 2. TL;DR (Resumo Executivo)

3–5 linhas descrevendo a proposta em linguagem direta. Um engenheiro que não participou da
reunião deve entender a essência da solução só com esse parágrafo.

### 3. Contexto e Problema

Por que a feature é necessária agora? Quem pediu, qual dor resolve, qual é o impacto de não
fazer. Sem entrar em detalhes de implementação — foque no "porquê".

Inclua:
- A motivação de negócio (3 clientes B2B, polling caro, risco de churn)
- O estado atual do sistema (OMS sem mecanismo de notificação externa)
- A lacuna que a feature preenche

### 4. Proposta Técnica

Visão geral da solução em linguagem arquitetural. Use 3–5 parágrafos ou uma lista de
componentes. **Não descer ao detalhe de implementação do FDD.**

Deve cobrir:
- O padrão outbox e sua relação com a transação de `changeStatus`
- O worker separado e sua política de polling
- O mecanismo de autenticação (HMAC-SHA256)
- A garantia de entrega (at-least-once + X-Event-Id)
- Como o módulo se encaixa na arquitetura existente (módulo novo, padrões reaproveitados)

### 5. Alternativas Consideradas

**Mínimo 2 alternativas reais da reunião**, cada uma com:
- Nome descritivo
- Descrição em 2–3 linhas
- O trade-off específico que levou ao descarte

Alternativas obrigatórias (ambas foram discutidas na reunião):

| Alternativa | Por que descartada | Timestamp |
|---|---|---|
| Disparo síncrono do webhook dentro da transação de `changeStatus` | Transação já é pesada; cliente lento travaria outras mudanças de status; se cliente offline exigiria rollback da mudança de status | [09:04]–[09:06] Bruno/Diego |
| Redis Streams (ou fila externa) como intermediário | Exigiria nova infra; time pequeno; overengineering para o volume; MySQL existente resolve | [09:07] Diego/Larissa |

### 6. Questões em Aberto

**Mínimo 2 pontos** levantados na reunião e não decididos (ou explicitamente adiados).
Para cada ponto: descrição do problema, quem levantou, timestamp, e qual seria o próximo
passo para decidir.

Questões obrigatórias:

| Questão | Levantado por | Timestamp | Próximo passo |
|---|---|---|---|
| Rate limiting de envio por cliente (risco de bombardeio em pico de mudanças) | Diego | [09:38]–[09:39] | Observar métricas em produção e decidir após primeira fase |
| Endurecimento de roles para CRUD de configuração de webhook em fases futuras | Sofia | [09:37] | Revisão de política de acesso na fase 2 |

### 7. Impacto e Riscos

Riscos arquiteturais da abordagem proposta. Não são os mesmos riscos do PRD (que são de
produto/negócio). Aqui o foco é técnico: o que pode dar errado na arquitetura escolhida.

Exemplos relevantes:
- Acúmulo de eventos na outbox se o worker ficar offline por tempo prolongado
- Ordering de eventos não garantida se múltiplos workers forem necessários no futuro
- Grace period de 24h na rotação de secret cria janela de vulnerabilidade temporária

### 8. Decisões Relacionadas

Lista de ADRs que formalizam as decisões mencionadas neste RFC. Use links relativos.

```markdown
- [ADR-001 — Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com Backoff e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003 — HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-Least-Once com X-Event-Id](./adrs/ADR-004-at-least-once-x-event-id.md)
- [ADR-005 — Worker Separado em Polling](./adrs/ADR-005-worker-separado-polling.md)
- [ADR-006 — Reuso dos Padrões Existentes](./adrs/ADR-006-reuso-padroes-existentes.md)
```

**Obrigatório:** referenciar com link pelo menos 2 ADRs.

## Critérios de aceite (checklist)

- [ ] Arquivo `docs/RFC.md` existe e está em Markdown
- [ ] Contém as 8 seções obrigatórias (Metadados, TL;DR, Contexto, Proposta, Alternativas,
      Questões em Aberto, Impacto e Riscos, Decisões Relacionadas)
- [ ] Metadados incluem os 5 participantes da reunião como revisores
- [ ] Seção "Alternativas" lista pelo menos 2 alternativas reais descartadas na reunião,
      cada uma com o trade-off que motivou o descarte
- [ ] Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos
- [ ] Referencia com link pelo menos 2 ADRs do pacote
- [ ] Não contém contratos de API detalhados, payloads ou código (esses pertencem ao FDD)
- [ ] Documento tem entre 2 e 4 páginas (estimativa: 600–1200 palavras)

## Prompt de ativação sugerido

```
Você é um engenheiro sênior submetendo uma proposta técnica à equipe para revisão.

Contexto do projeto:
- Leia `docs/context/TRANSCRIPT_CONTEXT.md` para decisões, alternativas e questões em aberto
- Os ADRs em `docs/adrs/` já estão escritos — referencie-os, não repita o conteúdo
- Leia `.kiro/skills/doc-writer/rfc.md` para o formato e critérios obrigatórios

Produza `docs/RFC.md` seguindo rigorosamente o formato definido.

Regras críticas:
1. RFC é conciso (2–4 páginas). Se estiver crescendo além disso, você está descendo
   ao nível do FDD. Mova o excesso para o FDD.
2. Alternativas consideradas devem ser as da reunião (com timestamps), não hipotéticas.
3. Questões em aberto devem ser pontos reais não decididos — não invente incertezas.
4. Não repita o conteúdo dos ADRs — apenas referencie com link.
5. Todo fato deve ter origem rastreável em `TRANSCRIPT_CONTEXT.md`.
```

# Plano de Produção — Pacote de Design Docs

## Visão Geral

Este plano organiza a produção do pacote completo de documentação do Sistema de Webhooks de
Notificação de Pedidos. Os entregáveis finais são: **PRD**, **RFC**, **FDD**, **5–8 ADRs**,
**TRACKER** e **README** atualizado.

O trabalho é organizado em **5 etapas sequenciais**. As etapas 1 e 2 são de contextualização
e produzem artefatos intermediários (não entregáveis finais). As etapas 3–5 produzem os
documentos do pacote.

---

## ~~Etapa 1 — Análise do Código-Fonte~~ ✅ CONCLUÍDA

**Objetivo:** mapear os componentes do código existente que serão referenciados na
documentação, evitando que agentes de documentação precisem navegar o código durante a
escrita.

**Agente:** Agente de análise de código  
**Insumos:** toda a árvore `src/`, `prisma/schema.prisma`, `tests/`  
**Saída:** `docs/context/CODE_MAP.md` ✅

### O que o agente deve mapear

| Componente | Arquivo(s) | O que documentar |
|---|---|---|
| Máquina de estados de pedidos | `src/modules/orders/order.status.ts` | Estados possíveis e transições válidas |
| Método `changeStatus` | `src/modules/orders/order.service.ts` | Assinatura, lógica transacional, o que já acontece dentro da transação |
| Repositório de pedidos | `src/modules/orders/order.repository.ts` | Métodos existentes, padrão de acesso ao Prisma |
| Classes de erro | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts` | Hierarquia, campos, padrão de códigos de erro usado hoje |
| Middleware de autenticação | `src/middlewares/auth.middleware.ts` | Como `requireRole` funciona, o que injeta no request |
| Middleware de erro | `src/middlewares/error.middleware.ts` | Como erros são capturados e formatados na resposta |
| Logger | `src/shared/logger/index.ts` | Instância Pino, campos padrão, como usar |
| Schema do banco | `prisma/schema.prisma` | Modelos existentes, campos relevantes (orders, order_status_history, stock_quantity) |
| Estrutura de rota/controller/service | Qualquer módulo existente | Padrão modular adotado (routes → controller → service → repository) |
| Resposta HTTP padronizada | `src/shared/http/response.ts` | Helper de resposta, formato padrão de envelope |

O `CODE_MAP.md` deve ser objetivo: para cada componente, uma seção curta com o trecho de
código relevante (não o arquivo inteiro) e uma descrição de 2–4 linhas sobre como o módulo
de webhooks vai precisar interagir com ele.

---

## ~~Etapa 2 — Análise da Transcrição~~ ✅ CONCLUÍDA

**Objetivo:** extrair da transcrição os fatos documentáveis — decisões fechadas, requisitos,
restrições, alternativas descartadas, pontos adiados — e produzir um documento de contexto
estruturado que sirva de fonte única para todos os agentes de documentação.

> **Por que isso importa:** a transcrição tem ~55 minutos de conversa não estruturada. Cada
> agente que precisasse lê-la inteira levaria tempo e correria risco de interpretar pontos
> ambíguos de forma inconsistente. O documento de contexto resolve isso uma vez e garante
> que todos os agentes partem da mesma leitura.

**Agente:** Agente de análise de transcrição  
**Insumos:** `TRANSCRICAO.md`  
**Saída:** `docs/context/TRANSCRIPT_CONTEXT.md` ✅

### Estrutura obrigatória do `TRANSCRIPT_CONTEXT.md`

```
# Contexto da Transcrição — Webhooks de Notificação de Pedidos

## Participantes
<tabela: nome, papel>

## Decisões Fechadas
<tabela: timestamp | falante | decisão | racional resumido>

## Requisitos Funcionais Identificados
<lista numerada: cada RF com timestamp de origem e falante>

## Requisitos Não Funcionais Identificados
<lista numerada: cada RNF com timestamp de origem e falante>

## Alternativas Descartadas
<tabela: alternativa | por que descartada | quem argumentou | timestamp>

## Pontos Explicitamente Fora de Escopo
<lista: cada item com timestamp e justificativa>

## Itens Adiados para Fase Futura
<lista: cada item com timestamp e quem sugeriu o adiamento>

## Questões em Aberto (não decididas ao final)
<lista: cada ponto com timestamp>

## Restrições e Condicionantes
<lista: restrições técnicas, de prazo ou de negócio com timestamp>
```

### Critérios de qualidade para o agente

- Cada item deve ter um timestamp `[hh:mm]` e o nome do falante
- **Não inferir** o que não foi dito. Se algo foi mencionado mas não decidido, vai em
  "Questões em Aberto", não em "Decisões Fechadas"
- Alternativas descartadas não devem aparecer como requisitos ou decisões adotadas
- Itens adiados ("a gente faz isso na fase 2", "deixa pra depois") devem ir em "Adiados",
  não em escopo da feature

---

## ~~Etapa 3 — Produção dos ADRs~~ ✅ CONCLUÍDA

**Objetivo:** registrar cada decisão arquitetural isolada antes de qualquer outro documento.
As decisões formam o esqueleto dos demais docs.

**Agente:** Agente de ADR  
**Insumos:** `docs/context/TRANSCRIPT_CONTEXT.md`, `docs/context/CODE_MAP.md`  
**Skill:** `.kiro/skills/doc-writer/adr.md`  
**Saída:** `docs/adrs/ADR-001-*.md` … `ADR-00N-*.md` (5 a 8 arquivos) ✅

### Decisões a cobrir (mínimo 5 das 6)

| # | Decisão | ADR sugerido |
|---|---|---|
| 1 | Padrão Outbox no MySQL para desacoplar disparo de webhook da transação | ADR-001 ✅ |
| 2 | Política de retry com backoff exponencial e DLQ | ADR-002 ✅ |
| 3 | Autenticação HMAC-SHA256 com secret por endpoint | ADR-003 ✅ |
| 4 | Garantia at-least-once com idempotência via `X-Event-Id` | ADR-004 ✅ |
| 5 | Worker em processo separado com polling na outbox | ADR-005 ✅ |
| 6 | Reuso dos padrões existentes do projeto (erros, logger, middleware) | ADR-006 ✅ |

### Formato de cada ADR (MADR)

```markdown
# ADR-NNN — Título da Decisão

## Status
Aceito | Em revisão | Substituído por ADR-XXX

## Contexto
<por que essa decisão precisou ser tomada, qual era o problema>

## Decisão
<o que foi decidido, em 1–3 frases diretas>

## Alternativas Consideradas
### Alternativa A — <nome>
<descrição + trade-off que levou ao descarte>

### Alternativa B — <nome>
<descrição + trade-off que levou ao descarte>

## Consequências
### Positivas
- <item>

### Negativas / Trade-offs
- <item>

## Referências
- Transcrição: [hh:mm] Falante
- Código: src/path/to/file.ts (quando aplicável)
```

**Regra:** pelo menos 1 ADR deve referenciar explicitamente um arquivo do código-fonte.

---

## ~~Etapa 4 — Produção do RFC e do FDD~~ ✅ CONCLUÍDA

Estes dois documentos são produzidos em sequência (RFC primeiro, depois FDD) porque o RFC
define as fronteiras arquiteturais que o FDD detalha.

### ~~4a — RFC~~ ✅ CONCLUÍDA

**Agente:** Agente de RFC  
**Insumos:** `docs/context/TRANSCRIPT_CONTEXT.md`, ADRs gerados  
**Skill:** `.kiro/skills/doc-writer/rfc.md`  
**Saída:** `docs/RFC.md` ✅

Escopo do RFC: 2–4 páginas. Visão arquitetural, alternativas descartadas, questões em
aberto. **Não descer ao nível de implementação** — isso é função do FDD.

Seções obrigatórias:
- Metadados (autor, status, data, revisores = participantes da reunião)
- TL;DR
- Contexto e problema
- Proposta técnica (visão geral)
- Alternativas consideradas (mínimo 2 alternativas reais da reunião com trade-off)
- Questões em aberto (mínimo 2 pontos não decididos)
- Impacto e riscos
- Decisões relacionadas (links `./adrs/ADR-NNN-*.md`)

### ~~4b — FDD~~ ✅ CONCLUÍDA

**Agente:** Agente de FDD  
**Insumos:** `docs/context/TRANSCRIPT_CONTEXT.md`, `docs/context/CODE_MAP.md`, `docs/RFC.md`, ADRs gerados  
**Skill:** `.kiro/skills/doc-writer/fdd.md`  
**Saída:** `docs/FDD.md` ✅

Seções obrigatórias:
- Contexto e motivação técnica
- Objetivos técnicos
- Escopo e exclusões (marcar arquivos novos como "(novo)": `src/worker.ts`, `src/modules/webhooks/*`)
- Fluxos detalhados (outbox → worker → retry → DLQ)
- Contratos públicos (≥ 4 endpoints HTTP com payload **literal** request/response e status codes)
- Matriz de erros com prefixo `WEBHOOK_*`
- Estratégias de resiliência (timeouts, retries, backoff exponencial, fallback)
- Observabilidade (métricas, logs Pino, tracing)
- **Integração com o sistema existente** (≥ 4 caminhos de arquivo reais **já existentes** do `CODE_MAP.md`)
- Dependências e compatibilidade
- Critérios de aceite técnicos
- Riscos e mitigação

---

## Etapa 5 — PRD, Tracker e README

### 5a — PRD

**Agente:** Agente de PRD  
**Insumos:** `docs/context/TRANSCRIPT_CONTEXT.md`, `docs/RFC.md`, `docs/FDD.md`  
**Skill:** `.kiro/skills/doc-writer/prd.md`  
**Saída:** `docs/PRD.md`

O PRD é o mais alto nível. Com RFC, FDD e ADRs prontos, é principalmente uma consolidação
em linguagem de produto/negócio. Atenção especial para:
- Pelo menos 8 requisitos funcionais rastreáveis à transcrição
- Pelo menos 1 objetivo com métrica e meta quantitativa **com timestamp de origem**; metas
  derivadas (sem origem na reunião) devem ser marcadas como "(derivada — validar com PM)"
- "Fora de escopo" com ≥ 2 itens explicitamente descartados/adiados na reunião
- "Riscos" com ≥ 2 riscos com probabilidade, impacto e mitigação

### 5b — Tracker de Rastreabilidade

**Agente:** Agente de Tracker  
**Insumos:** todos os documentos produzidos + `docs/context/TRANSCRIPT_CONTEXT.md` + `docs/context/CODE_MAP.md`  
**Skill:** `.kiro/skills/doc-writer/tracker.md`  
**Saída:** `docs/TRACKER.md`

Formato da tabela:

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | ... | TRANSCRICAO | [09:00] Marcos |

Critérios mínimos:
- ≥ 80% dos itens dos documentos têm linha no tracker
- ≥ 70% das linhas têm `Fonte = TRANSCRICAO` com timestamp `[hh:mm] Nome`
- ≥ 5 linhas têm `Fonte = CODIGO` com caminho de arquivo real
- Itens sem origem rastreável (ex: métricas derivadas do PRD) usam `Fonte = DERIVADO` e não
  contam para as metas de cobertura acima — não forjar timestamp para eles

### 5c — README do processo

**Responsável:** humano (você), com apoio de agente de rascunho  
**Skill:** `.kiro/skills/doc-writer/readme-processo.md`  
**Saída:** `README.md` (substituindo o enunciado atual)

Seções obrigatórias:
- Sobre o desafio (1–2 parágrafos em suas palavras)
- Ferramentas de IA utilizadas
- Workflow adotado
- Prompts customizados (≥ 2 em bloco de código)
- Iterações e ajustes (≥ 2 momentos concretos de correção)
- Como navegar a entrega

---

## ~~Etapa 6 — Criação da Skill de Documentação~~ ✅ CONCLUÍDA

**Objetivo:** criar uma skill reutilizável para que agentes sejam direcionados à produção de
cada tipo de documento sem precisar ler todos os detalhes do README/enunciado a cada invocação.

**Responsável:** você (humano) ou agente de skill  
**Saída:** `.kiro/skills/doc-writer/` — `SKILL.md` (índice) + guias `adr.md`, `rfc.md`,
`fdd.md`, `prd.md`, `tracker.md`, `readme-processo.md` ✅

### Estrutura da skill

A skill adota o modelo **hub-and-spoke**: um `SKILL.md` como índice/entrada e um arquivo de
detalhe por tipo de documento. Cada arquivo de detalhe contém:

1. **Papel do documento** — em 2–3 linhas, o que ele responde e a quem se destina
2. **Fronteiras** — o que NÃO entra (para evitar duplicação entre docs)
3. **Seções obrigatórias** — lista com descrição de 1 linha cada
4. **Critérios de aceite** — checklist copiado do README, formatado como lista de tarefas
5. **Prompt de ativação sugerido** — template de prompt que o humano pode usar para invocar
   um agente dessa skill

#### Arquivos da skill

```
.kiro/skills/doc-writer/
  SKILL.md               # índice: contexto do projeto, ordem de produção, mapa de documentos
  adr.md                 # guia do ADR
  rfc.md                 # guia do RFC
  fdd.md                 # guia do FDD
  prd.md                 # guia do PRD
  tracker.md             # guia do Tracker
  readme-processo.md     # guia do README do processo
```

O `SKILL.md` não repete o conteúdo dos guias: ele referencia cada arquivo de detalhe
(`[adr.md](./adr.md)`, `[rfc.md](./rfc.md)`, etc.) e o agente lê o guia específico antes de
escrever cada documento.

---

## Artefatos Intermediários

Os documentos abaixo são de apoio ao processo. **Não são entregáveis finais**, mas são
obrigatórios para garantir qualidade e rastreabilidade.

| Artefato | Caminho | Produzido em | Consumido por |
|---|---|---|---|
| Mapa do código | `docs/context/CODE_MAP.md` ✅ | Etapa 1 ✅ | FDD (seção Integração), ADR-006, Tracker |
| Contexto da transcrição | `docs/context/TRANSCRIPT_CONTEXT.md` ✅ | Etapa 2 ✅ | Todos os agentes de doc |
| Skill de documentação | `.kiro/skills/doc-writer/` (`SKILL.md` + 6 guias) ✅ | Etapa 6 ✅ | Todos os agentes de doc |

---

## Sequência de Execução

```
Etapa 1: Análise de Código ✅ ────────────────────────────────┐
                                                               ↓
Etapa 2: Análise da Transcrição ✅ ──────────────────────> Etapa 3: ADRs ✅
                                                               ↓
                                                          Etapa 4a: RFC ✅
                                                               ↓
                                                          Etapa 4b: FDD ✅
                                                               ↓
                                           ┌──────────── Etapa 5a: PRD
                                           ↓
                                      Etapa 5b: Tracker
                                           ↓
                                      Etapa 5c: README
                                           ↓
                              Etapa 6: Skill ✅ (concluída antes das etapas de doc)
```

Etapas 1 e 2 podem ser executadas em paralelo entre si.

---

## Checklist de Entrega Final

Antes do push, verificar item por item:

### PRD
- [ ] `docs/PRD.md` existe
- [ ] Contém todas as seções obrigatórias
- [ ] ≥ 8 requisitos funcionais rastreáveis
- [ ] ≥ 1 objetivo com métrica quantitativa e timestamp de origem; metas sem origem na
      transcrição marcadas como "(derivada)" e nunca apresentadas como decisão da reunião
- [ ] Linguagem de produto — sem jargão técnico ("outbox", "HMAC", "DLQ", "worker") nas
      seções de produto
- [ ] "Fora de escopo" com ≥ 2 itens descartados/adiados
- [ ] "Riscos" com ≥ 2 itens (probabilidade + impacto + mitigação)
- [ ] Nenhum item inventado — todos os RFs têm timestamp de origem no TRANSCRIPT_CONTEXT

### RFC
- [ ] `docs/RFC.md` existe
- [ ] Contém todas as seções obrigatórias
- [ ] ≥ 2 alternativas descartadas com trade-off
- [ ] ≥ 2 questões em aberto
- [ ] Links para ≥ 2 ADRs

### FDD
- [ ] `docs/FDD.md` existe
- [ ] Contém todas as seções obrigatórias
- [ ] ≥ 4 endpoints com payload de exemplo **literal** (JSON de request e response, não
      apenas referência de tipo) e status codes
- [ ] Fluxos detalhados cobrem outbox → worker → retry → DLQ
- [ ] Matriz de erros com prefixo `WEBHOOK_*`
- [ ] "Integração com o sistema existente" referencia ≥ 4 arquivos reais **já existentes**
      (arquivos novos como `src/worker.ts` ficam na seção de Escopo, marcados como "(novo)")
- [ ] "Observabilidade" cobre métricas, logs e tracing

### ADRs
- [ ] `docs/adrs/` contém 5–8 arquivos `ADR-NNN-*.md`
- [ ] Cada ADR tem Status, Contexto, Decisão, Alternativas, Consequências
- [ ] Conjunto cobre ≥ 5 das 6 decisões principais
- [ ] ≥ 1 ADR referencia arquivo real do código

### Tracker
- [ ] `docs/TRACKER.md` existe com tabela no formato correto
- [ ] ≥ 80% de cobertura dos itens dos documentos
- [ ] ≥ 70% das linhas com `Fonte = TRANSCRICAO` e timestamp válido
- [ ] ≥ 5 linhas com `Fonte = CODIGO` e caminho real
- [ ] Nenhum ID de linha duplicado
- [ ] Nenhuma `Localização` vazia ou genérica; itens sem origem rastreável usam
      `Fonte = DERIVADO` (não contam para as metas de cobertura/percentuais)

### README
- [ ] `README.md` substituído (não é mais o enunciado)
- [ ] Contém todas as seções obrigatórias
- [ ] ≥ 1 ferramenta de IA listada
- [ ] ≥ 2 prompts customizados em bloco de código
- [ ] ≥ 2 iterações/ajustes concretos descritos

### Consistência
- [ ] Nenhum item dos docs contradiz a transcrição ou o código
- [ ] Nenhum arquivo de código citado é inexistente no repositório
- [ ] Itens descartados na reunião **não aparecem** como requisitos ou decisões adotadas

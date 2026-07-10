# Guia de Produção — Tracker de Rastreabilidade

## Papel

Tabela de referência cruzada que mapeia cada item dos documentos do pacote à sua origem na
transcrição ou no código. Funciona como verificação de integridade: se um item não tem
linha no tracker, ou se a coluna "Localização" não pode ser preenchida, o item provavelmente
foi inventado e deve ser removido dos documentos.

**Responde:** *De onde veio cada coisa?*  
**Audiência:** revisores do pacote, avaliadores do desafio, qualquer leitor que queira
verificar a autenticidade de um item.

## Formato obrigatório da tabela

```markdown
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
```

### Descrição das colunas

| Coluna | Descrição | Exemplos de valor |
|---|---|---|
| **ID** | Identificador único do item | `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002` |
| **Documento** | Arquivo onde o item aparece | `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-retry-backoff-dlq.md` |
| **Tipo** | Categoria do item | `Requisito Funcional`, `Requisito Não Funcional`, `Decisão`, `Restrição`, `Trade-off`, `Alternativa Descartada`, `Item Fora de Escopo`, `Contrato de API`, `Código de Erro` |
| **Conteúdo (resumo)** | Descrição de uma linha do item | `Worker roda como processo separado da API` |
| **Fonte** | Origem do item | `TRANSCRICAO` ou `CODIGO` |
| **Localização** | Onde encontrar na fonte | `[09:11] Diego` (TRANSCRICAO) ou `src/modules/orders/order.service.ts` (CODIGO) |

> **Itens derivados:** métricas ou metas de produto sem origem direta na transcrição (ex:
> algumas metas do §4 do PRD marcadas como "(derivada)") não têm fonte rastreável. Não invente
> um timestamp para elas. Use `Fonte = DERIVADO` com `Localização = "meta de produto (validar com PM)"`,
> ou deixe-as fora do tracker. Elas não contam para a meta de cobertura de 80% nem para os
> percentuais de `TRANSCRICAO`/`CODIGO`.

### Padrão de IDs por documento

| Documento | Prefixo | Tipos comuns |
|---|---|---|
| PRD | `PRD-FR-NN`, `PRD-RNF-NN`, `PRD-RISCO-NN`, `PRD-SCOPE-NN` | Requisito Funcional, Requisito Não Funcional, Risco, Escopo |
| RFC | `RFC-ALT-NN`, `RFC-OPEN-NN`, `RFC-RISCO-NN` | Alternativa Descartada, Questão em Aberto, Risco |
| FDD | `FDD-CONTRATO-NN`, `FDD-ERRO-NN`, `FDD-FLUXO-NN`, `FDD-OBS-NN`, `FDD-INTEG-NN` | Contrato de API, Código de Erro, Fluxo, Observabilidade, Integração |
| ADR | `ADR-001`, `ADR-002`, ... | Decisão |

## Critérios de cobertura mínima

| Critério | Meta |
|---|---|
| Itens dos documentos com linha no tracker | ≥ 80% |
| Linhas com `Fonte = TRANSCRICAO` e timestamp válido (`[hh:mm] Nome`) | ≥ 70% do total |
| Linhas com `Fonte = CODIGO` com caminho de arquivo real | ≥ 5 linhas |

### O que conta como "item identificável"

Um item é identificável (e portanto deve ter linha no tracker) quando é:
- Requisito funcional (qualquer RF listado no PRD ou FDD)
- Requisito não funcional (qualquer RNF)
- Decisão arquitetural (cada ADR = 1 linha)
- Alternativa descartada (cada alternativa na seção do RFC ou ADRs)
- Questão em aberto (cada questão no RFC)
- Contrato de API (cada endpoint no FDD)
- Código de erro (cada entrada na matriz de erros do FDD)
- Item fora de escopo (cada item na seção do PRD)
- Integração com sistema existente (cada arquivo referenciado no FDD)

## Processo de produção

O tracker é melhor construído após todos os outros documentos estarem prontos. O processo:

1. **Varra o PRD** linha por linha. Para cada RF, RNF, risco, item de escopo: crie uma
   linha no tracker. Preencha Localização a partir do `TRANSCRIPT_CONTEXT.md`.

2. **Varra o RFC** linha por linha. Para cada alternativa descartada e questão em aberto:
   crie uma linha. Verifique que as alternativas têm timestamp válido.

3. **Varra cada ADR**. Cada ADR é 1 linha com `Tipo = Decisão`. Se o ADR referencia
   arquivo de código, crie linha adicional com `Fonte = CODIGO`.

4. **Varra o FDD**. Para cada endpoint (Contratos), cada código de erro (Matriz), cada
   arquivo referenciado (Integração): crie uma linha.

5. **Verifique a cobertura:** conte total de itens vs linhas no tracker. Se < 80%,
   identifique o que está faltando.

6. **Valide as localizações de CODIGO:** cada caminho de arquivo em `Localização` deve
   existir realmente no repositório. Use `docs/context/CODE_MAP.md` para verificar.

## Linhas de CODIGO obrigatórias (mínimo 5)

Ao menos 5 linhas do tracker devem ter `Fonte = CODIGO`. Exemplos esperados:

| Item | Localização esperada |
|---|---|
| Ponto de integração: `changeStatus` recebe insert na outbox | `src/modules/orders/order.service.ts` |
| Padrão de erros reaproveitado no módulo de webhooks | `src/shared/errors/app-error.ts` |
| Middleware `requireRole` usado no endpoint admin | `src/middlewares/auth.middleware.ts` |
| Logger Pino reaproveitado no worker | `src/shared/logger/index.ts` |
| Padrão de módulo (routes/controller/service/repository) | `src/modules/orders/` |
| Modelos Prisma existentes (Order, OrderStatusHistory) | `prisma/schema.prisma` |
| Resposta paginada reaproveitada | `src/shared/http/response.ts` |

## Critérios de aceite do documento (checklist)

- [ ] Arquivo `docs/TRACKER.md` existe
- [ ] Tabela segue o formato com as 6 colunas obrigatórias
- [ ] Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- [ ] Pelo menos 70% das linhas têm `Fonte = TRANSCRICAO` com timestamp no formato
      `[hh:mm] Nome` (verificável em `docs/context/TRANSCRIPT_CONTEXT.md`)
- [ ] Pelo menos 5 linhas têm `Fonte = CODIGO` com caminho de arquivo real (verificável
      em `docs/context/CODE_MAP.md`)
- [ ] Nenhum ID de linha está duplicado
- [ ] Nenhuma linha tem `Localização` vazia ou genérica (ex: "transcrição" sem timestamp)

## Prompt de ativação sugerido

```
Você é um auditor técnico construindo o Tracker de Rastreabilidade do pacote de design docs
do sistema de webhooks.

Contexto do projeto:
- Leia todos os documentos produzidos: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`,
  `docs/adrs/ADR-001-*.md` a `ADR-NNN-*.md`
- Use `docs/context/TRANSCRIPT_CONTEXT.md` para localizar timestamps das origens
- Use `docs/context/CODE_MAP.md` para localizar caminhos de arquivo das origens de código
- Leia `.kiro/skills/doc-writer/tracker.md` para o formato e critérios obrigatórios

Produza `docs/TRACKER.md`.

Processo:
1. Varra cada documento do pacote e liste todos os itens identificáveis
2. Para cada item, crie uma linha na tabela com ID único, tipo, conteúdo resumido,
   fonte (TRANSCRICAO ou CODIGO) e localização exata
3. Para TRANSCRICAO: use o timestamp `[hh:mm] Nome` do TRANSCRIPT_CONTEXT
4. Para CODIGO: use o caminho de arquivo do CODE_MAP
5. Ao final, verifique: ≥ 80% de cobertura, ≥ 70% com TRANSCRICAO+timestamp,
   ≥ 5 linhas com CODIGO+caminho real

Regra absoluta: se você não consegue preencher a coluna Localização de um item,
esse item provavelmente foi inventado. Remova-o do documento de origem antes de
incluir no tracker, ou marque explicitamente como "ORIGEM NÃO RASTREÁVEL" para
investigação manual.
```

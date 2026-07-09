# Guia de Produção — PRD (Product Requirements Document)

## Papel

Define o "porquê" e o "o quê" da feature em linguagem de produto e negócio. É o documento
de mais alto nível do pacote: estabelece o problema, o público, o escopo, as métricas de
sucesso e os critérios de aceitação.

**Responde:** *Por que fazer e o que construir?*  
**Audiência:** PM, stakeholders de negócio, tech lead, e qualquer novo membro do time que
queira entender o contexto da feature.

## Fronteiras (o que NÃO entra no PRD)

- Como a solução funciona tecnicamente → vai no RFC e FDD
- Por que decisões arquiteturais foram tomadas → vai nos ADRs
- Contratos de API, schemas, código → vão no FDD
- O PRD não menciona outbox, HMAC, worker, DLQ — menciona *comportamento observável*

## Quando produzir

**Depois** de RFC, FDD e ADRs estarem prontos. Com esses documentos em mãos, o PRD é
principalmente uma consolidação em linguagem de produto — os fatos técnicos já estão
mapeados e basta traduzi-los para a perspectiva do negócio.

## Seções obrigatórias

### 1. Resumo e Contexto da Feature

2–3 parágrafos: o que é a feature, o cenário de negócio que a motivou e onde ela se encaixa
no produto. Mencione os três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) e
o problema de polling excessivo. Não use jargão técnico.

### 2. Problema e Motivação

Descreva o problema atual e seu impacto:
- Clientes B2B precisam realizar polling periódico em `GET /orders` para detectar mudanças
  de status — integrações lentas e caras para eles
- Risco concreto de churn (Atlas Comercial sinalizou migração para concorrente)
- Impossibilidade de reação em tempo real às mudanças de status dos pedidos

### 3. Público-Alvo e Cenários de Uso

**Público primário:** clientes B2B com integração programática via API (Atlas Comercial,
MaxDistribuição, Nova Cargo e futuros clientes com perfil similar).

**Cenários de uso obrigatórios:**
- Cliente recebe notificação quando pedido é pago → dispara processo logístico interno
- Cliente recebe notificação de envio → envia comunicação para o cliente final
- Cliente recebe notificação de cancelamento → libera estoque no sistema dele
- Cliente recebe notificação de entrega → finaliza ciclo de atendimento

### 4. Objetivos e Métricas de Sucesso

**Obrigatório: pelo menos 1 objetivo com métrica quantitativa.**

> **Rastreabilidade das metas:** apenas a latência `< 10s` tem origem direta na transcrição
> (`[09:02] Marcos`). As demais metas abaixo são **derivadas** (propostas de produto, não
> ditas na reunião) e estão marcadas com _(derivada — validar com PM)_. Toda meta sem
> timestamp deve ser sinalizada assim; no Tracker, use `Fonte = DERIVADO` ou omita a linha em
> vez de forjar um timestamp. Não apresente número derivado como se viesse da reunião.

| Objetivo | Métrica | Meta | Origem |
|---|---|---|---|
| Notificação em tempo real | Latência entre mudança de status e recebimento pelo cliente | < 10 segundos em P95 | `[09:02] Marcos` |
| Reduzir polling dos clientes B2B | Redução no volume de chamadas GET /orders por cliente | ≥ 80% de redução em 30 dias após ativação | _(derivada — validar com PM)_ |
| Confiabilidade de entrega | Taxa de entrega bem-sucedida (não contar DLQ) | ≥ 99% das entregas nos primeiros 5 attempts | _(derivada — validar com PM)_ |
| Adoção | Clientes B2B com webhook ativo | 3 clientes piloto ativos no primeiro sprint após lançamento | _(derivada — clientes citados em `[09:00]/[09:02] Marcos`)_ |

### 5. Escopo

#### Incluso nesta fase

- Sistema de webhooks outbound: plataforma notifica clientes (não o contrário)
- CRUD de endpoints webhook por customer (cadastrar, editar, ativar/desativar, remover)
- Filtro de eventos por status: cliente escolhe quais transições quer receber
- Autenticação de payload via HMAC-SHA256 para o cliente verificar autenticidade
- Rotação de secret com período de graça de 24h
- Histórico de entregas por webhook (sucesso/falha, tempo de resposta, payload)
- Reprocessamento manual de eventos falhos (Dead Letter Queue)

#### Fora de escopo desta fase

Itens explicitamente descartados ou adiados na reunião:

- **Webhooks inbound** (clientes enviando eventos para a plataforma) — fora por decisão de
  escopo `[09:02]–[09:03] Marcos/Sofia`
- **Notificação por email** quando webhook falha repetidamente — adiado para próxima fase
  após medição de impacto `[09:37] Larissa`
- **Dashboard visual** para o cliente acompanhar webhooks — projeto separado do time de
  frontend `[09:39]–[09:40] Larissa`
- **Rate limiting de saída** por cliente — adiado para observar em produção `[09:38]–[09:39]
  Diego/Larissa`
- **Garantia exactly-once** de entrega — fora por complexidade; adota-se at-least-once com
  idempotência via X-Event-Id `[09:25] Diego`

### 6. Requisitos Funcionais

**Mínimo 8 requisitos rastreáveis à transcrição.**

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | O sistema permite cadastrar um endpoint webhook informando URL (HTTPS), customer, e lista de status a monitorar | `[09:31] Marcos` |
| RF-02 | A secret de autenticação é gerada pela plataforma e devolvida apenas na criação do webhook | `[09:31] Marcos` |
| RF-03 | Cada mudança de status que constar no filtro do webhook gera uma notificação HTTP para o endpoint do cliente | `[09:33]–[09:34] Marcos/Bruno/Diego` |
| RF-04 | Em caso de falha de entrega, o sistema retenta automaticamente com backoff exponencial (até 5 tentativas) | `[09:15]–[09:17] Diego/Larissa` |
| RF-05 | Eventos que esgotam todas as tentativas são movidos para uma fila de mortos (DLQ) para reprocessamento manual | `[09:17]–[09:18] Diego/Larissa` |
| RF-06 | Operadores com role ADMIN podem reprocessar manualmente eventos da DLQ | `[09:18]–[09:19] Diego/Larissa / [09:35]–[09:36] Larissa/Sofia` |
| RF-07 | O cliente pode consultar o histórico completo de tentativas de entrega de cada webhook | `[09:34] Marcos` |
| RF-08 | O cliente pode rotacionar a secret do webhook; a secret anterior permanece válida por 24h após a rotação | `[09:21] Sofia` |
| RF-09 | O payload de cada notificação inclui um identificador único de evento (X-Event-Id) para deduplicação | `[09:25] Diego` |
| RF-10 | Cada notificação é assinada com HMAC-SHA256 usando a secret do endpoint; assinatura enviada no header X-Signature | `[09:20] Sofia` |
| RF-11 | Webhooks com URL HTTP (não-HTTPS) são recusados na criação com erro de validação | `[09:23] Sofia` |
| RF-12 | A notificação carrega o snapshot do estado do pedido no momento da mudança de status (não o estado atual) | `[09:52] Larissa/Diego` |

### 7. Requisitos Não Funcionais

| ID | Requisito | Meta | Fonte |
|---|---|---|---|
| RNF-01 | Latência de notificação | < 10 segundos entre mudança de status e envio | `[09:02] Marcos` |
| RNF-02 | Impacto na API | Zero degradação de latência na transação de mudança de status | `[09:06] Diego/Larissa` (decorrência da decisão de outbox; discutido em `[09:04]–[09:06]`) |
| RNF-03 | Disponibilidade do worker | Worker em processo separado; falha do worker não afeta a API | `[09:11] Diego` |
| RNF-04 | Confiabilidade | Garantia at-least-once; no máximo 1 duplicata esperada em cenários de retry | `[09:24]–[09:26] Diego/Larissa` |
| RNF-05 | Prazo | Entrega em 3 sprints incluindo revisão de segurança | `[09:46]–[09:47] Larissa/Sofia` |
| RNF-06 | Revisão de segurança | Código de HMAC e geração de secret revisados por engenheira de segurança antes do deploy | `[09:46] Sofia` |
| RNF-07 | Tamanho de payload | Notificações acima de 64KB são recusadas com erro | `[09:23]–[09:24] Diego/Larissa` |

### 8. Decisões e Trade-offs Principais

Não liste detalhes técnicos — apenas os trade-offs de produto visíveis ao cliente.

- **At-least-once vs exactly-once:** o sistema pode entregar o mesmo evento mais de uma vez
  em cenários de retry. Clientes devem deduplicar pelo X-Event-Id. Escolha pragmática para
  manter a complexidade gerenciável nesta fase.
- **Ordering implícita, não garantida globalmente:** eventos do mesmo pedido chegam em ordem
  enquanto houver um único worker. Se múltiplos workers forem necessários no futuro, a
  ordering pode ser afetada. Clientes não devem depender de ordering cross-pedido.
- **Responsabilidade da secret:** a plataforma gera e rotaciona secrets; clientes são
  responsáveis por não vazar a secret nos próprios logs ou sistemas.

### 9. Dependências

| Dependência | Tipo | Status |
|---|---|---|
| Banco MySQL existente (sem nova infra) | Técnica | Disponível |
| Revisão de segurança (Sofia) antes do deploy | Processo | A agendar |
| Documentação no portal de desenvolvedor (Marcos) | Produto | A produzir pós-feature |
| Confirmação de prazo com clientes B2B (Atlas, MaxDistribuição, Nova Cargo) | Negócio | A confirmar (`[09:47] Marcos`) |

### 10. Riscos e Mitigação

**Mínimo 2 riscos com probabilidade, impacto e mitigação.**

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Churn de cliente B2B (Atlas) se prazo não for cumprido | Alta | Alto | Entregar MVP das 3 sprints; comunicar progresso semanalmente via Marcos |
| Evento entregue duplicado gera inconsistência no sistema do cliente | Média | Médio | Documentar X-Event-Id no portal; orientar clientes na integração; deduplicação é responsabilidade do cliente |
| Indisponibilidade prolongada do endpoint do cliente (> 15h) leva evento para DLQ sem notificação automática | Média | Médio | Histórico de entregas via API; considerar email de alerta na fase 2 |
| Vazamento de secret HMAC pelo cliente | Baixa | Alto | Endpoint de rotação de secret com grace period de 24h; documentar como proceder |

### 11. Critérios de Aceitação

Comportamentos observáveis que indicam que a feature está funcionando corretamente:

- [ ] Um cliente B2B consegue cadastrar um webhook e receber notificação em < 10s após
      mudança de status de pedido
- [ ] A notificação contém assinatura HMAC verificável com a secret do endpoint
- [ ] Um webhook com URL HTTP é recusado na criação
- [ ] Após falha no endpoint do cliente, o sistema retenta automaticamente até 5 vezes
- [ ] Após 5 falhas, o evento aparece no histórico da DLQ e pode ser reprocessado por admin
- [ ] O cliente consegue rotacionar a secret sem downtime (secret antiga válida por 24h)
- [ ] O histórico de entregas está disponível via API com status, código HTTP e tempo de
      resposta de cada tentativa

### 12. Estratégia de Testes e Validação

#### Testes automatizados (obrigatórios antes do deploy)
- Testes de integração dos endpoints CRUD de webhook
- Testes unitários da lógica de HMAC (assinatura e verificação)
- Testes de integração do fluxo completo: mudança de status → inserção na outbox → entrega
- Testes de retry: simular falha N vezes e verificar progressão de backoff
- Testes de DLQ: simular 5 falhas e verificar movimentação para dead_letter

#### Validação de segurança
- Revisão de código por Sofia (mínimo 2 dias úteis) focada em HMAC e geração de secret
- Verificar que secret não é logada em nenhum ponto
- Verificar que URL HTTP é recusada antes de chegar ao banco

#### Validação de negócio (UAT)
- Demonstração com um dos três clientes piloto em ambiente de staging antes do go-live
- Verificar latência < 10s em condições normais
- Marcos confirma com clientes que o formato do payload atende às necessidades de integração

## Critérios de aceite do documento (checklist)

- [ ] Arquivo `docs/PRD.md` existe e está em Markdown
- [ ] Contém todas as 12 seções obrigatórias
- [ ] Identifica no mínimo 8 requisitos funcionais com origem na transcrição (timestamp)
- [ ] Inclui pelo menos 1 objetivo com métrica quantitativa e meta numérica com timestamp
      de origem; metas sem origem na transcrição estão marcadas como "(derivada)"
- [ ] Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados/adiados
      na reunião, com timestamp
- [ ] Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação
- [ ] Linguagem de produto — sem termos como "outbox", "HMAC", "DLQ", "worker" nas seções
      de produto (podem aparecer em Decisões e Trade-offs se necessário)
- [ ] Nenhum item inventado — todos os RFs têm timestamp de origem no TRANSCRIPT_CONTEXT e
      nenhuma métrica derivada é apresentada como se tivesse sido decidida na reunião

## Prompt de ativação sugerido

```
Você é um Product Manager sênior escrevendo o PRD (Product Requirements Document) do
sistema de webhooks de notificação de pedidos.

Contexto do projeto:
- Leia `docs/context/TRANSCRIPT_CONTEXT.md` para decisões, requisitos e timestamps
- O RFC em `docs/RFC.md`, o FDD em `docs/FDD.md` e os ADRs em `docs/adrs/` já estão
  prontos — use-os como referência mas não repita o conteúdo técnico neles
- Leia `.kiro/skills/doc-writer/prd.md` para o formato e critérios obrigatórios

Produza `docs/PRD.md`.

Regras críticas:
1. PRD usa linguagem de produto, não técnica. Não mencione "outbox", "HMAC", "polling",
   "worker" ou "DLQ" nas seções de produto — descreva o comportamento observável.
2. Mínimo 8 requisitos funcionais, cada um com timestamp de origem no TRANSCRIPT_CONTEXT.
3. Seção "Fora de escopo" deve listar pelo menos 2 itens descartados/adiados na reunião
   com timestamp — não itens hipotéticos.
4. Seção "Riscos" deve ter pelo menos 2 riscos com probabilidade, impacto e mitigação.
5. Pelo menos 1 objetivo com métrica quantitativa (número, percentual ou tempo) com
   timestamp de origem. Metas sem origem na transcrição devem ser marcadas como
   "(derivada — validar com PM)"; nunca apresente um número derivado como decisão da reunião.
```

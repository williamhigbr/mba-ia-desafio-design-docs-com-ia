# Da Reunião ao Documento — Processo de Produção

> O enunciado original do desafio está disponível em:
> https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia

---

## Sobre o desafio

O desafio propôs uma tarefa concreta: pegar a gravação de uma reunião técnica de 55 minutos
e transformá-la em um pacote completo de documentação de engenharia — PRD, RFC, FDD, seis
ADRs e um Tracker de rastreabilidade — sem inventar nenhum requisito, decisão ou restrição
que não pudesse ser rastreado à transcrição ou ao código existente.

O cenário era um Order Management System (OMS) em Node.js + TypeScript que precisava de um
sistema de webhooks outbound para notificar clientes B2B sobre mudanças de status de pedidos.
A reunião já tinha acontecido, as decisões já tinham sido tomadas — meu papel foi destilá-las
em documentação acionável. A IA funcionou como parceira de escrita e análise, mas a
responsabilidade de garantir que nada foi inventado, que os documentos não se contradizem e
que cada afirmação tem uma origem verificável foi inteiramente humana.

---

## Ferramentas de IA utilizadas

- **Kiro (Claude Sonnet 4.5)** — ferramenta principal ao longo de todo o processo. Usado
  para análise do código-fonte (geração do `CODE_MAP.md`), extração estruturada da
  transcrição (`TRANSCRIPT_CONTEXT.md`), criação da skill de documentação e produção de
  todos os documentos do pacote (ADRs, RFC, FDD, PRD, Tracker). A interação foi conduzida
  via agentes especializados por etapa, com revisão crítica entre cada entrega.

- **Claude.ai (interface web)** — usado pontualmente para exploração inicial da estrutura
  do problema e validação de consistência entre documentos antes de passar para a produção
  final no Kiro.

---

## Workflow adotado

O trabalho foi organizado em seis etapas sequenciais, documentadas em `docs/docs_plan.md`.
A ordem não foi arbitrária — cada documento depende dos anteriores para não repetir conteúdo
no lugar errado.

**Etapa 1 — Análise do código-fonte:** antes de escrever qualquer linha de documentação,
pedi ao agente que mapeasse os componentes do código que o módulo de webhooks precisaria
referenciar ou estender: o método `changeStatus`, a hierarquia de erros, o logger Pino, os
middlewares de autenticação e erro, o padrão de módulos, o schema Prisma. O resultado foi
o `docs/context/CODE_MAP.md` — um documento de referência que todos os agentes subsequentes
consultaram sem precisar navegar o código diretamente.

**Etapa 2 — Análise da transcrição:** a transcrição bruta tem 55 minutos de conversa não
estruturada. Pedi ao agente que a lesse integralmente e extraísse, em formato estruturado,
as decisões fechadas (com timestamp e falante), os requisitos funcionais e não funcionais, as
alternativas descartadas, os itens fora de escopo e as questões em aberto. O resultado foi
o `docs/context/TRANSCRIPT_CONTEXT.md` — a fonte única de verdade que eliminou ambiguidades
de interpretação entre os agentes subsequentes.

**Etapa 6 — Skill de documentação** (executada em paralelo com as etapas 1–2): criei uma
skill reutilizável em `.kiro/skills/doc-writer/` com um guia por tipo de documento. Cada
guia define papel, fronteiras, seções obrigatórias, critérios de aceite e um prompt de
ativação sugerido. Isso permitiu que os agentes de cada etapa recebessem instruções precisas
sem precisar ler o enunciado completo a cada invocação.

**Etapa 3 — ADRs:** produzi os seis ADRs antes de qualquer outro documento de entrega. As
decisões formam o esqueleto dos documentos subsequentes — RFC e FDD referenciam os ADRs em
vez de reexplicar as justificativas. Cada ADR foi escrito com pelo menos uma alternativa real
da reunião (com timestamp) e, quando aplicável, referência a arquivo de código existente.

**Etapa 4a — RFC:** com os ADRs prontos, o RFC se reduziu a consolidar a visão arquitetural,
listar as alternativas descartadas e registrar as questões em aberto — tudo já mapeado no
`TRANSCRIPT_CONTEXT.md`. O cuidado principal foi manter o documento conciso (2–4 páginas)
e não repetir o detalhamento que pertence ao FDD.

**Etapa 4b — FDD:** o documento mais extenso e o mais trabalhoso. Com RFC e ADRs como
fundação, o FDD detalhou os fluxos passo a passo, os contratos de API com payloads literais,
a matriz de erros, as estratégias de resiliência e a seção obrigatória de integração com o
sistema existente — que exigiu referenciar pelo menos quatro arquivos reais do código.

**Etapa 5a — PRD:** produzido por último entre os grandes documentos, porque com RFC, FDD
e ADRs prontos é principalmente uma consolidação em linguagem de produto. O desafio foi
manter a linguagem de produto — sem "outbox", "HMAC", "DLQ" ou "worker" nas seções voltadas
ao negócio — e distinguir claramente as metas com origem na transcrição das derivadas.

**Etapa 5b — Tracker:** varredura sistemática de todos os documentos para mapear cada item
à sua origem. Funcionou também como auditoria final: qualquer linha sem `Localização`
preenchível indicaria que o item correspondente havia sido inventado.

---

## Prompts customizados

### Prompt 1: Análise estruturada da transcrição
**Usado para:** gerar o `docs/context/TRANSCRIPT_CONTEXT.md` a partir da `TRANSCRICAO.md`

```
Você é um analista técnico experiente. Sua tarefa é ler integralmente a transcrição
da reunião em TRANSCRICAO.md e extrair os fatos documentáveis em formato estruturado.

Produza docs/context/TRANSCRIPT_CONTEXT.md com as seguintes seções obrigatórias:

1. Participantes (tabela: nome, papel)
2. Decisões Fechadas (tabela: timestamp | falante | decisão | racional resumido)
   — apenas decisões confirmadas por consenso ou pelo tech lead. Se foi mencionado
   mas não decidido, vai em "Questões em Aberto", não aqui.
3. Requisitos Funcionais Identificados (lista numerada com timestamp e falante)
4. Requisitos Não Funcionais Identificados (lista numerada com timestamp e falante)
5. Alternativas Descartadas (tabela: alternativa | por que descartada | quem | timestamp)
6. Pontos Explicitamente Fora de Escopo (lista com timestamp e justificativa)
7. Itens Adiados para Fase Futura (lista com quem sinalizou e timestamp)
8. Questões em Aberto — não decididas ao final da reunião (lista com timestamp)
9. Restrições e Condicionantes (lista com tipo e timestamp)

Regras absolutas:
- Cada item deve ter timestamp [hh:mm] e nome do falante
- NÃO inferir o que não foi dito. Dúvida → "Questões em Aberto"
- Alternativas descartadas NÃO aparecem como decisões adotadas
- Itens adiados ("na fase 2", "deixa pra depois") → "Adiados", não em escopo
- Se um item foi discutido mas não resolvido, vai em "Questões em Aberto"
```

### Prompt 2: Produção de ADR individual
**Usado para:** gerar cada ADR com rastreabilidade precisa e sem descida ao nível do FDD

```
Você é um engenheiro sênior documentando uma decisão arquitetural do sistema de
webhooks do OMS. Produza o arquivo docs/adrs/ADR-00N-titulo.md.

Insumos:
- docs/context/TRANSCRIPT_CONTEXT.md — decisões, racionais e timestamps
- docs/context/CODE_MAP.md — referências ao código existente
- .kiro/skills/doc-writer/adr.md — formato obrigatório

Decisão a documentar: [NOME DA DECISÃO]
Timestamp de referência: [hh:mm] [Falante]

Regras:
1. Contexto: autocontido — engenheiro sem acesso à transcrição deve entender
   por que a decisão foi necessária. Referencie o código existente quando relevante.
2. Decisão: 1–3 frases descritivas, sem imperativo ("deve", "precisa").
3. Alternativas: apenas as discutidas na reunião, com timestamp exato.
   NÃO use alternativas hipotéticas genéricas.
4. Consequências: pelo menos 1 positiva e 1 negativa/trade-off concretos.
5. Referências: timestamp da transcrição + caminho de arquivo de código (quando aplicável).
6. NÃO desça ao nível de implementação — contratos, schemas e código ficam no FDD.
```

### Prompt 3: Seção "Integração com o sistema existente" do FDD
**Usado para:** garantir que o FDD referenciasse arquivos reais e não inventasse pontos de integração

```
Você está escrevendo a seção §10 "Integração com o sistema existente" do FDD.

Esta seção deve referenciar EXCLUSIVAMENTE arquivos que já existem no repositório.
Use docs/context/CODE_MAP.md como fonte — não mencione nenhum caminho que não esteja
nesse documento.

Para cada arquivo referenciado, descreva:
1. O que o arquivo faz hoje (1 linha)
2. Como o módulo de webhooks vai interagir com ele (2–3 linhas + snippet de código
   quando a integração for não óbvia)

Arquivos mínimos a cobrir (todos verificáveis em CODE_MAP.md):
- src/modules/orders/order.service.ts
- src/shared/errors/app-error.ts e src/shared/errors/http-errors.ts
- src/middlewares/auth.middleware.ts
- src/shared/logger/index.ts
- src/config/database.ts
- src/shared/http/response.ts
- prisma/schema.prisma

Regra: arquivos NOVOS (src/worker.ts, src/modules/webhooks/*) NÃO pertencem a esta seção.
Eles ficam na seção §3 (Escopo), marcados como "(novo)".
```

---

## Iterações e ajustes

### Iteração 1 — RFC cresceu além do escopo

Na primeira versão do RFC gerada pelo agente, a seção "Proposta Técnica" desceu ao nível
de implementação: incluía a tabela completa de campos da `webhook_outbox`, a progressão
exata de backoff (1m/5m/30m/2h/12h) e exemplos de código. O RFC estava virando um FDD
comprimido — exatamente o que o guia diz que não deve acontecer.

Corrigi com um prompt focado: "O RFC deve operar em nível arquitetural. Remova qualquer
detalhe que um desenvolvedor precise para codar — esse nível de detalhe pertence ao FDD.
Mantenha apenas o suficiente para que um stakeholder técnico entenda a abordagem escolhida
e por que as alternativas foram descartadas." O documento reduziu de ~8 páginas para ~4,
ficou mais direto e as fronteiras com o FDD ficaram nítidas.

### Iteração 2 — ADRs com alternativas hipotéticas

Na primeira rodada, dois ADRs incluíam alternativas que não foram discutidas na reunião —
o agente inventou "alternativas plausíveis" para preencher a seção, mas sem timestamp.
O ADR-004 (at-least-once), por exemplo, tinha uma "Alternativa C — Sem garantia de
entrega" que nunca foi mencionada por ninguém.

Identifiquei o problema ao tentar preencher a coluna "Localização" do Tracker: não havia
timestamp porque a alternativa não existia na transcrição. Corrigi com a instrução: "Apenas
alternativas discutidas na reunião com timestamp em TRANSCRIPT_CONTEXT.md. Se não há
timestamp para uma alternativa, ela não foi discutida e não pertence ao ADR."

### Iteração 3 — PRD com linguagem técnica nas seções de produto

A primeira versão do PRD usava termos como "outbox", "HMAC", "DLQ" e "worker" nas seções
de requisitos funcionais e objetivos — linguagem de implementação num documento de produto.
Um PM lendo esse PRD estaria sendo exposto a detalhes que não são da camada dele.

Refinei com o prompt: "Traduza cada requisito técnico para comportamento observável pelo
cliente ou pelo negócio. 'O sistema usa outbox para garantir atomicidade' vira 'O envio
da notificação é atômico com a confirmação da mudança de status — se a mudança falhar, a
notificação não é enviada.' Termos como outbox, DLQ, HMAC e worker não devem aparecer nas
seções de produto."

### Iteração 4 — Tracker com timestamps forjados

Na primeira versão do Tracker, o agente colocou timestamps em métricas derivadas — metas
como "≥ 99% de taxa de entrega" e "≥ 80% de redução de polling" receberam localizações
inventadas do tipo `[09:05] Marcos`. Essas metas foram propostas como razoáveis, mas não
foram ditas na reunião.

A correção foi classificá-las explicitamente como `Fonte = DERIVADO` com
`Localização = "meta de produto (validar com PM)"`. O guia é claro: inventar timestamp para
item derivado é pior que não ter rastreabilidade — é rastreabilidade falsa.

---

## Como navegar a entrega

Ordem sugerida de leitura, do contexto à implementação:

1. [`docs/context/TRANSCRIPT_CONTEXT.md`](docs/context/TRANSCRIPT_CONTEXT.md) — contexto
   extraído da transcrição: decisões, requisitos, alternativas descartadas e questões em
   aberto. Ponto de partida para entender o "porquê" de tudo.

2. [`docs/context/CODE_MAP.md`](docs/context/CODE_MAP.md) — mapa do código existente:
   componentes relevantes com trechos e notas de integração. Referência para entender o
   "onde" do sistema atual.

3. [`docs/adrs/`](docs/adrs/) — seis ADRs, um por decisão arquitetural. Leia nesta ordem:
   - [ADR-001 — Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 — Retry com Backoff e DLQ](docs/adrs/ADR-002-retry-backoff-dlq.md)
   - [ADR-003 — HMAC-SHA256 por Endpoint](docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
   - [ADR-004 — At-Least-Once com X-Event-Id](docs/adrs/ADR-004-at-least-once-x-event-id.md)
   - [ADR-005 — Worker Separado em Polling](docs/adrs/ADR-005-worker-separado-polling.md)
   - [ADR-006 — Reuso dos Padrões Existentes](docs/adrs/ADR-006-reuso-padroes-existentes.md)

4. [`docs/RFC.md`](docs/RFC.md) — proposta arquitetural consolidada: abordagem escolhida,
   alternativas descartadas e questões em aberto. 2–4 páginas, nível de arquitetura.

5. [`docs/FDD.md`](docs/FDD.md) — especificação de implementação: fluxos passo a passo,
   contratos de API com payloads literais, matriz de erros, resiliência, observabilidade e
   integração com o código existente. Documento acionável para o desenvolvedor.

6. [`docs/PRD.md`](docs/PRD.md) — visão de produto: problema, público, métricas de sucesso,
   escopo, requisitos e critérios de aceitação em linguagem de negócio.

7. [`docs/TRACKER.md`](docs/TRACKER.md) — rastreabilidade: tabela que mapeia cada item dos
   documentos à sua origem na transcrição (com timestamp) ou no código (com caminho de
   arquivo). Use para verificar a integridade de qualquer afirmação do pacote.

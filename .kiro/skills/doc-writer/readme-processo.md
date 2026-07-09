# Guia de Produção — README do Processo

## Papel

Substitui o enunciado original do `README.md` com a documentação do processo de produção
do pacote de design docs. É o único documento escrito em primeira pessoa — narra a jornada
de quem produziu a documentação.

**Responde:** *Como este pacote de documentação foi produzido?*  
**Audiência:** avaliadores do desafio, colegas de curso, futuros contribuidores.

## Fronteiras (o que NÃO entra no README)

- Conteúdo técnico da feature (webhook, outbox, etc.) — isso está nos outros docs
- Reprodução do enunciado original — pode manter um link, mas não copiar o conteúdo
- Instruções de como usar a aplicação (setup, npm, docker) — não é o foco

## Quando produzir

**Por último**, após todos os outros documentos estarem prontos e revisados. Só então o
processo pode ser documentado com clareza.

## Seções obrigatórias

### 1. Sobre o desafio

1 a 2 parágrafos descrevendo a tarefa em suas palavras. Não copie o enunciado. Explique:
- O que foi pedido (transformar transcrição de reunião em pacote de design docs)
- O cenário (OMS + feature de webhooks)
- O diferencial (IA como ferramenta principal, humano como maestro)

### 2. Ferramentas de IA utilizadas

Lista das ferramentas que você usou, com uma nota curta sobre o papel de cada uma.
Formato sugerido:

```markdown
- **[Nome da ferramenta]** — [para que foi usada neste projeto]
```

Exemplos de papéis: análise da transcrição, geração de rascunho de ADRs, revisão de
consistência, geração de payloads de exemplo, etc.

### 3. Workflow adotado

Descreva como o trabalho foi organizado:
- Em que ordem os documentos foram produzidos (deve refletir ADRs → RFC → FDD → PRD → Tracker)
- Como a interação com a IA foi estruturada (prompts, revisão, iteração)
- Como os artefatos intermediários (CODE_MAP, TRANSCRIPT_CONTEXT) foram usados

### 4. Prompts customizados

**Mínimo 2 prompts em bloco de código.** Devem ser prompts reais que você usou ou adaptou,
não templates genéricos. Inclua o contexto de para que serviu cada prompt.

Estrutura sugerida:

```markdown
### Prompt: [nome descritivo]
**Usado para:** [análise da transcrição / geração de ADRs / etc.]

\```
[conteúdo do prompt]
\```
```

### 5. Iterações e ajustes

Descreva os principais momentos em que a IA gerou algo errado ou superficial e você
precisou corrigir. **Mínimo 2 momentos concretos**, descrevendo:
- O que a IA gerou
- O que estava errado ou superficial
- Como você corrigiu (novo prompt, edição manual, etc.)
- Quantas iterações no total até o resultado final

### 6. Como navegar a entrega

Caminho dos arquivos entregues e a ordem sugerida de leitura. Formato sugerido:

```markdown
1. `docs/context/TRANSCRIPT_CONTEXT.md` — contexto extraído da transcrição (ponto de partida)
2. `docs/adrs/ADR-001-*.md` a `ADR-NNN-*.md` — decisões isoladas
3. `docs/RFC.md` — proposta arquitetural
4. `docs/FDD.md` — especificação de implementação
5. `docs/PRD.md` — visão de produto
6. `docs/TRACKER.md` — rastreabilidade de todos os itens
```

## Critérios de aceite (checklist)

- [ ] `README.md` foi substituído (não é mais o enunciado do desafio)
- [ ] Contém as 6 seções obrigatórias
- [ ] Lista pelo menos 1 ferramenta de IA com nota sobre seu papel
- [ ] Mostra pelo menos 2 prompts customizados em bloco de código Markdown
- [ ] Descreve pelo menos 2 iterações/ajustes concretos (o que estava errado + como corrigiu)
- [ ] Inclui seção de navegação com os caminhos dos arquivos entregues

## Nota sobre autenticidade

O README é o único documento do pacote que deve refletir genuinamente a sua experiência.
Evite gerar o README inteiro com IA — use-a para rascunhar seções específicas e reescreva
com suas palavras. Avaliadores percebem quando o relato é genérico demais.

## Prompt de ativação sugerido (para rascunho)

```
Você é um assistente ajudando a estruturar o README do processo de um desafio de design docs.

O desafio foi: transformar a transcrição de uma reunião técnica em um pacote completo de
documentação (PRD, RFC, FDD, ADRs, Tracker) usando IA como ferramenta principal.

Com base nas seguintes informações que vou fornecer, produza um rascunho do README:
- Ferramentas usadas: [listar aqui]
- Ordem de produção dos documentos: ADRs → RFC → FDD → PRD → Tracker
- Principais dificuldades encontradas: [listar aqui]
- Prompts que funcionaram bem: [colar aqui]

Leia `.kiro/skills/doc-writer/readme-processo.md` para o formato obrigatório.

Produza apenas o rascunho — o humano vai revisar e reescrever as partes pessoais em
primeira pessoa antes da entrega final.
```

# Skill: Redator de Design Docs — OMS Webhooks

Skill para produção do pacote de documentação técnica do Sistema de Webhooks de Notificação
de Pedidos. Use esta skill para orientar agentes na escrita de cada tipo de documento.

## Contexto do projeto

- **Aplicação:** Order Management System (OMS) em Node.js + TypeScript, banco MySQL via Prisma
- **Feature:** Sistema de Webhooks outbound para notificar clientes B2B sobre mudanças de
  status de pedidos
- **Fonte de verdade:**
  - Decisões e requisitos → `docs/context/TRANSCRIPT_CONTEXT.md`
  - Código existente → `docs/context/CODE_MAP.md`
- **Regra de ouro:** todo item documentado deve ter origem rastreável na transcrição
  (com timestamp `[hh:mm] Nome`) ou no código (com caminho de arquivo). Não inventar.

## Documentos do pacote

| Documento | Arquivo de detalhes | Saída |
|---|---|---|
| ADR | [adr.md](./adr.md) | `docs/adrs/ADR-NNN-*.md` (5–8 arquivos) |
| RFC | [rfc.md](./rfc.md) | `docs/RFC.md` |
| FDD | [fdd.md](./fdd.md) | `docs/FDD.md` |
| PRD | [prd.md](./prd.md) | `docs/PRD.md` |
| Tracker | [tracker.md](./tracker.md) | `docs/TRACKER.md` |
| README do processo | [readme-processo.md](./readme-processo.md) | `README.md` |

## Ordem de produção

```
ADRs → RFC → FDD → PRD → Tracker → README
```

ADRs primeiro porque as decisões formam o esqueleto dos demais documentos. RFC antes do FDD
porque define as fronteiras arquiteturais que o FDD detalha.

## Princípio de separação entre documentos

| Documento | Pergunta que responde | Altura |
|---|---|---|
| PRD | Por que e o quê? | Produto / negócio |
| RFC | Como pretendemos resolver, e o que ainda está em aberto? | Arquitetura |
| ADR | Por que decidimos exatamente assim? | Decisão pontual |
| FDD | Como construir, em detalhe? | Implementação |
| Tracker | De onde veio cada coisa? | Transversal |

Conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.
Leia o arquivo de detalhes do documento antes de escrever.

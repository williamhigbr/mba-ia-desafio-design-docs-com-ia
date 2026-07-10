# PRD — Sistema de Webhooks de Notificação de Pedidos

> **Documentos relacionados:** [RFC](./RFC.md) | [FDD](./FDD.md) | [ADRs](./adrs/)  
> Toda informação rastreável a `docs/context/TRANSCRIPT_CONTEXT.md` ou ao código existente.

---

## 1. Resumo e Contexto da Feature

O Sistema de Webhooks de Notificação de Pedidos permite que clientes B2B recebam
notificações automáticas toda vez que um pedido muda de status na plataforma OMS. Em vez
de consultar a API periodicamente para detectar mudanças, o cliente cadastra um endereço
de entrega (endpoint) e a plataforma envia a notificação no momento em que a mudança
acontece.

A feature foi motivada por uma demanda concreta de três clientes B2B — Atlas Comercial,
MaxDistribuição e Nova Cargo — que dependem da visibilidade em tempo real do ciclo de vida
dos pedidos para acionar fluxos internos: separação de mercadoria, atualização de ERP,
comunicação com transportadoras e notificação ao cliente final. Sem essa visibilidade, o
custo de integração é alto e a reação a eventos críticos (pagamento confirmado, envio
realizado, cancelamento) é lenta e propensa a falhas. `[09:00]–[09:02] Marcos`

A feature faz parte da evolução da camada de integração do OMS e é o primeiro mecanismo
de notificação ativa da plataforma. Ela não altera nenhum comportamento existente — apenas
adiciona uma forma de comunicação de saída que não existia.

---

## 2. Problema e Motivação

**Situação atual:** clientes B2B que precisam reagir a mudanças de status de pedidos não
têm outra opção senão consultar a API repetidamente (`GET /orders`) para verificar se houve
alguma alteração. Essa abordagem é cara (volume de chamadas desnecessárias), lenta (a
reação depende do intervalo de polling) e frágil (falhas de rede resultam em eventos
perdidos silenciosamente).

**Impacto no negócio:** o Atlas Comercial sinalizou risco de migração para um concorrente
caso a plataforma não ofereça notificações em tempo real. A MaxDistribuição e a Nova Cargo
têm dificuldade em automatizar seus processos logísticos por conta da latência inerente ao
polling. `[09:00] Marcos`

**O que a feature resolve:** com webhooks, a plataforma passa a ser a parte ativa da
comunicação. Clientes recebem a notificação em menos de 10 segundos após a mudança de
status, sem precisar manter integração de consulta periódica. `[09:02] Marcos`

---

## 3. Público-Alvo e Cenários de Uso

**Público primário:** clientes B2B com integração programática via API — empresas que têm
sistemas internos (ERP, WMS, plataformas de e-commerce) que precisam reagir automaticamente
a eventos do ciclo de vida de pedidos. `[09:00]–[09:02] Marcos`

**Cenários de uso:**

- **Pagamento confirmado → disparo logístico:** ao receber notificação de status `PAID`, o
  sistema do cliente libera a separação de mercadoria no armazém sem esperar a próxima
  janela de consulta.

- **Pedido enviado → comunicação ao comprador:** ao receber notificação de status `SHIPPED`,
  o cliente dispara automaticamente o e-mail ou SMS de rastreamento para o comprador final.

- **Pedido cancelado → reposição de estoque:** ao receber notificação de status `CANCELLED`,
  o sistema do cliente repõe o item no seu próprio inventário e estorna reservas de entrega.

- **Pedido entregue → fechamento de ciclo:** ao receber notificação de status `DELIVERED`,
  o cliente finaliza o fluxo de atendimento, libera a fatura e arquiva o pedido.

`[09:02] Marcos`

---

## 4. Objetivos e Métricas de Sucesso

> **Nota de rastreabilidade:** a meta de latência tem origem direta na transcrição. As
> demais metas são derivadas e estão marcadas como tal — não foram mencionadas na reunião
> e devem ser validadas com o PM antes de usar como OKRs formais.

| Objetivo | Métrica | Meta | Origem |
|---|---|---|---|
| Notificação em tempo real | Tempo entre mudança de status e recebimento pelo cliente (P95) | < 10 segundos | `[09:02] Marcos` |
| Reduzir carga de polling | Redução no volume de chamadas `GET /orders` por cliente B2B | ≥ 80% em 30 dias após ativação por cliente | _(derivada — validar com PM)_ |
| Confiabilidade de entrega | Taxa de entregas bem-sucedidas nas primeiras 5 tentativas | ≥ 99% | _(derivada — validar com PM)_ |
| Adoção inicial | Clientes piloto com pelo menos 1 webhook ativo | 3 clientes (Atlas, MaxDistribuição, Nova Cargo) no primeiro sprint após lançamento | _(derivada — clientes citados em `[09:00] Marcos`)_ |

---

## 5. Escopo

### Incluso nesta fase

- Cadastro, edição, ativação/desativação e remoção de endpoints de notificação por cliente
- Filtro por tipo de evento: o cliente escolhe quais mudanças de status quer receber
- Entrega automática de notificações com reenvio em caso de falha
- Autenticação das notificações: o cliente pode verificar que a notificação veio da plataforma
- Rotação de credencial de autenticação com período de transição de 24 horas
- Histórico de tentativas de entrega por endpoint (sucesso, falha, tempo de resposta)
- Reprocessamento manual de notificações que falharam definitivamente (role ADMIN)

### Fora de escopo desta fase

Itens explicitamente descartados ou adiados na reunião:

- **Notificações recebidas de clientes (inbound):** o escopo é exclusivamente saída da
  plataforma para o cliente; o caminho inverso não está previsto para esta fase.
  `[09:02]–[09:03] Marcos / Sofia`

- **Alerta por email em caso de falha repetida:** Marcos sugeriu notificação por email
  quando um webhook falha consecutivamente; Larissa adiou para próxima fase após medir
  o impacto real em produção. `[09:37]–[09:38] Larissa`

- **Dashboard visual de webhooks:** painel de acompanhamento para o cliente foi
  identificado como projeto separado do time de frontend, fora do escopo desta entrega.
  `[09:39]–[09:40] Larissa`

- **Controle automático de volume de saída por cliente:** mecanismo para evitar que um
  cliente receba muitas notificações simultâneas foi levantado por Diego e adiado para
  observação em produção. `[09:38]–[09:39] Diego / Larissa`

- **Garantia de entrega única (sem duplicatas):** a plataforma adota entrega ao menos uma
  vez; clientes devem usar o identificador de evento para descartar duplicatas caso recebam
  a mesma notificação mais de uma vez. `[09:25] Diego`

---

## 6. Requisitos Funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | O sistema permite cadastrar um endpoint de notificação informando URL (obrigatoriamente HTTPS), identificador do cliente e lista de mudanças de status a monitorar | `[09:31] Marcos`; HTTPS obrigatório em `[09:23] Sofia` |
| RF-02 | A credencial de autenticação do endpoint é gerada pela plataforma e entregue ao cliente apenas no momento do cadastro; não é possível consultá-la depois | `[09:31] Marcos` |
| RF-03 | O cliente pode editar a URL, o filtro de eventos e o estado ativo/inativo de um endpoint cadastrado | `[09:33] Bruno` |
| RF-04 | O cliente pode remover um endpoint de notificação cadastrado | `[09:33] Bruno` |
| RF-05 | O cliente pode listar todos os endpoints de notificação cadastrados para um cliente, com filtro por cliente | `[09:33] Bruno` |
| RF-06 | Apenas mudanças de status que constem no filtro configurado pelo cliente geram notificação para aquele endpoint; a filtragem é feita no momento do registro do evento | `[09:33]–[09:34] Marcos / Bruno / Diego` |
| RF-07 | Em caso de falha de entrega, a plataforma retenta automaticamente até 5 vezes com intervalos crescentes entre as tentativas | `[09:15]–[09:17] Diego / Larissa` |
| RF-08 | Notificações que esgotam todas as tentativas são movidas para uma fila de falhas definitivas, preservando o conteúdo e o motivo da falha para investigação | `[09:17]–[09:18] Diego / Larissa` |
| RF-09 | Usuários com permissão de administrador podem reprocessar manualmente uma notificação da fila de falhas definitivas, reiniciando as tentativas de entrega | `[09:18]–[09:19] Diego / Larissa`; role ADMIN exigida em `[09:36] Sofia / Larissa` |
| RF-10 | O cliente pode consultar o histórico completo de tentativas de entrega de cada endpoint, incluindo resultado, código de resposta e tempo de resposta de cada tentativa | `[09:34] Marcos` |
| RF-11 | O cliente pode solicitar a rotação da credencial de autenticação; a credencial anterior permanece válida por 24 horas após a rotação para evitar interrupção de serviço | `[09:21] Sofia` |
| RF-12 | Cada notificação carrega um identificador único imutável, enviado em todas as tentativas de entrega do mesmo evento, para que o cliente identifique e descarte duplicatas | `[09:25] Diego` |
| RF-13 | Cada notificação contém uma assinatura calculada sobre o conteúdo da mensagem usando a credencial do endpoint, permitindo que o cliente verifique autenticidade e integridade | `[09:20] Sofia` |
| RF-14 | O conteúdo da notificação reflete o estado do pedido no momento em que a mudança de status ocorreu, não o estado atual no momento da entrega | `[09:52] Larissa / Diego` |

---

## 7. Requisitos Não Funcionais

| ID | Requisito | Meta | Fonte |
|---|---|---|---|
| RNF-01 | Latência de notificação | < 10 segundos entre a confirmação da mudança de status e o envio ao cliente (P95) | `[09:02] Marcos` |
| RNF-02 | Impacto na operação principal | Zero degradação de latência na confirmação de mudança de status do pedido | `[09:04]–[09:06] Bruno / Diego` |
| RNF-03 | Isolamento de falhas | Uma falha no serviço de entregas não afeta o processamento de pedidos da API | `[09:11] Diego` |
| RNF-04 | Garantia de entrega | Ao menos uma entrega por evento; possibilidade de duplicata em cenários de reenvio | `[09:24]–[09:26] Diego / Larissa` |
| RNF-05 | Segurança do canal | URL do endpoint deve usar HTTPS obrigatoriamente; HTTP é recusado na validação | `[09:23] Sofia` |
| RNF-06 | Tamanho da notificação | Notificações acima de 64 KB são recusadas; nenhum conteúdo é truncado silenciosamente | `[09:23]–[09:24] Diego / Larissa` |
| RNF-07 | Prazo de entrega | Feature completa em 3 sprints, incluindo revisão de segurança | `[09:46]–[09:47] Larissa / Sofia` |
| RNF-08 | Revisão de segurança | Código de autenticação e geração de credenciais revisado por engenheira de segurança antes do deploy (mínimo 2 dias úteis) | `[09:46] Sofia` |

---

## 8. Decisões e Trade-offs Principais

**Entrega ao menos uma vez:** a plataforma pode entregar a mesma notificação mais de uma
vez em cenários de reenvio (ex: timeout sem resposta clara do cliente). O identificador
único de evento permite que o cliente descarte duplicatas. Essa escolha mantém a solução
simples e robusta, seguindo o padrão adotado por plataformas como Stripe e GitHub.
`[09:24]–[09:26] Diego / Larissa`

**Credencial por endpoint:** cada endpoint tem sua própria credencial de autenticação em
vez de uma única compartilhada entre todos. Isso limita o impacto de um eventual vazamento
a um único endpoint. `[09:21]–[09:22] Sofia / Larissa`

**Ordering por pedido, não global:** eventos do mesmo pedido chegam em ordem. Para cenários
futuros com múltiplos processadores paralelos, a ordering entre pedidos diferentes não seria
garantida — essa limitação é conhecida e está documentada. `[09:12]–[09:13] Diego / Larissa`

---

## 9. Dependências

| Dependência | Tipo | Status | Fonte |
|---|---|---|---|
| Banco de dados existente (sem nova infraestrutura) | Técnica | Disponível | `[09:07] Diego` |
| Revisão de segurança pelo time de segurança antes do deploy | Processo | A agendar (mínimo 2 dias úteis) | `[09:46] Sofia` |
| Documentação no portal do desenvolvedor (formato de payload, X-Event-Id, verificação de assinatura) | Produto | A produzir após feature pronta | `[09:26] Marcos` |
| Confirmação de prazo com clientes B2B piloto | Negócio | A confirmar com Atlas, MaxDistribuição e Nova Cargo | `[09:47] Marcos` |

---

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Churn do Atlas Comercial se prazo de 3 sprints não for cumprido | Alta | Alto | Comunicar progresso semanalmente via PM; priorizar MVP funcional nas primeiras sprints | `[09:00] Marcos` |
| Notificação duplicada causa processamento duplo no sistema do cliente | Média | Médio | Documentar identificador único de evento no portal; orientar clientes B2B no processo de integração; responsabilidade de deduplicação é do cliente | `[09:25]–[09:26] Diego / Marcos` |
| Indisponibilidade prolongada do endpoint do cliente (> 15h) leva evento para fila de falhas sem notificação automática | Média | Médio | Histórico de tentativas disponível via API; alerta por email previsto para fase 2 após medição de impacto | `[09:37]–[09:38] Marcos / Larissa` |
| Vazamento da credencial de autenticação pelo cliente | Baixa | Alto | Endpoint de rotação com período de transição de 24h; documentação de procedimento de resposta a incidente | `[09:21]–[09:22] Sofia` |

---

## 11. Critérios de Aceitação

- [ ] Um cliente B2B consegue cadastrar um endpoint e receber a primeira notificação em
      menos de 10 segundos após uma mudança de status de pedido
- [ ] A notificação recebida contém assinatura verificável usando a credencial do endpoint
- [ ] Um endpoint com URL HTTP (não HTTPS) é recusado no cadastro com mensagem de erro clara
- [ ] Ao simular falha no endpoint do cliente, o sistema reenvia automaticamente sem
      intervenção manual
- [ ] Após o número máximo de tentativas, o evento aparece na fila de falhas e pode ser
      reprocessado por um administrador via API
- [ ] O cliente consegue rotacionar a credencial e continuar recebendo notificações sem
      interrupção durante o período de transição
- [ ] O histórico de tentativas de entrega está disponível via API com resultado, código de
      resposta HTTP e tempo de resposta de cada tentativa
- [ ] Uma mudança de status de pedido não tem latência perceptível comparada ao comportamento
      antes da feature (zero impacto na operação principal)

---

## 12. Estratégia de Testes e Validação

### Testes automatizados

- Testes de integração dos endpoints de gerenciamento de webhooks (cadastro, edição,
  remoção, listagem)
- Testes unitários da lógica de geração e verificação de assinatura de notificação
- Testes de integração do fluxo completo: mudança de status → evento registrado → entrega
- Testes de reenvio: simular falha e verificar que os reenvios ocorrem nos intervalos
  corretos
- Testes da fila de falhas: simular número máximo de falhas e verificar movimentação para
  a fila de falhas definitivas

### Validação de segurança

- Revisão de código pelo time de segurança (Sofia) com foco em autenticação e geração de
  credenciais (mínimo 2 dias úteis antes do deploy) `[09:46] Sofia`
- Verificar que a credencial não aparece em logs em nenhum ponto do sistema
- Verificar que URL HTTP é bloqueada antes de persistir no banco de dados

### Validação com clientes

- Demonstração com pelo menos um dos três clientes piloto em ambiente de staging antes do
  go-live
- Validar que o formato da notificação atende aos requisitos de integração dos clientes
  (Marcos coordena) `[09:47] Marcos`
- Verificar latência de ponta a ponta em condições normais (< 10s)

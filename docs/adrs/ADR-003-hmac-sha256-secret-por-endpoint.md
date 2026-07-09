# ADR-003 — Autenticação HMAC-SHA256 com Secret por Endpoint

## Status

Aceito

## Contexto

Cada chamada HTTP que o worker faz para o endpoint do cliente transporta dados sensíveis
de negócio (mudança de status de pedido, IDs de clientes, valores). O cliente precisa ter
um mecanismo para verificar que a requisição realmente veio da plataforma OMS e que o
payload não foi alterado em trânsito.

Existem dois problemas distintos a resolver:

1. **Autenticidade:** como o cliente prova que a requisição partiu da plataforma?
2. **Integridade:** como o cliente verifica que o payload não foi adulterado?

HMAC (Hash-based Message Authentication Code) com SHA-256 resolve ambos: a assinatura é
calculada sobre o corpo da requisição usando um segredo compartilhado. Qualquer alteração
no payload invalida a assinatura.

A questão do escopo do segredo é crítica. Um segredo global (único para toda a plataforma)
simplificaria a gestão, mas criaria um risco sistêmico: o vazamento do segredo de um
único cliente exporia todos os outros clientes da plataforma. A equipe de segurança (Sofia)
relatou que esse cenário já ocorreu com um cliente anterior da empresa, o que tornou a
decisão por segredo individual não-negociável.

Adicionalmente, toda operação de segurança que exige troca de credenciais precisa de uma
estratégia de rotação que não cause downtime. Um segredo que não pode ser rotacionado sem
interromper o fluxo de notificações não seria operacionalmente viável para clientes B2B.

## Decisão

Cada webhook endpoint tem seu próprio segredo HMAC-SHA256, gerado pela plataforma no
momento da criação e retornado ao cliente em texto plano apenas nessa ocasião. O worker
assina o corpo de cada requisição com esse segredo e envia a assinatura no header
`X-Signature`. A rotação de segredo é suportada via endpoint dedicado, com grace period
de 24 horas durante o qual o segredo antigo e o novo são aceitos em paralelo.

## Alternativas Consideradas

### Alternativa A — Segredo HMAC global (único para toda a plataforma)

Um único segredo compartilhado entre todos os endpoints webhook de todos os clientes.
Simplificaria a gestão e a rotação.

**Por que descartada:** vazamento do segredo de um cliente (via log, captura de request,
engenharia reversa) exporia a assinatura de todos os outros clientes da plataforma.
Sofia relatou que esse cenário já ocorreu na empresa e foi o motivador direto da decisão
por segredo individual. Descartada por Sofia em `[09:21]–[09:22]`.

### Alternativa B — Autenticação via token estático no header (Bearer token)

O cliente configuraria um token que a plataforma enviaria em cada request no header
`Authorization: Bearer <token>`. Sem cálculo de assinatura.

**Por que descartada:** não garante integridade do payload — um intermediário poderia
alterar o corpo da requisição sem invalidar o token. HMAC assina o corpo, tornando
adulteração detectável. Não foi discutida explicitamente como alternativa formal, mas
o debate de Sofia em `[09:20]` deixa implícito que a assinatura sobre o corpo é requisito.

### Alternativa C — Rotação de segredo com substituição imediata (sem grace period)

Trocar o segredo imediatamente, invalidando o anterior no momento da rotação.

**Por que descartada:** causaria downtime para o cliente entre a rotação e o deploy
do novo segredo na aplicação consumidora. O grace period de 24h permite que o cliente
atualize sua configuração sem interromper o recebimento de webhooks. Descartada implicitamente
por Sofia ao propor o grace period em `[09:21]`.

## Consequências

### Positivas

- Blast radius de vazamento é limitado a um único endpoint — outros clientes não são afetados.
- Assinatura sobre o corpo garante integridade: payload adulterado em trânsito é detectado.
- Grace period de 24h elimina o risco de downtime durante rotação de segredo.
- Padrão amplamente documentado e com bibliotecas maduras disponíveis em todas as linguagens
  (facilita implementação no lado do cliente).
- TLS obrigatório (URL HTTPS) adiciona camada de proteção sobre o canal de transporte.

### Negativas / Trade-offs

- Gestão de segredo por endpoint aumenta a superfície de armazenamento de material
  criptográfico — a plataforma precisa armazenar N segredos (um por endpoint configurado).
- Durante o grace period de rotação (24h), dois segredos coexistem; o worker precisa
  tentar ambos, ou o endpoint de verificação precisa verificar com ambos.
- O cliente precisa implementar a verificação HMAC no lado receptor; adiciona complexidade
  de integração que precisa ser bem documentada no portal do cliente (mencionado por Marcos
  em `[09:26]`).
- A plataforma é responsável pela geração e armazenamento seguro dos segredos — vazamento
  do banco exporia os segredos de todos os endpoints.

## Referências

- Transcrição: `[09:20]–[09:22] Sofia / Larissa` — decisão por HMAC-SHA256 e secret por endpoint
- Transcrição: `[09:21] Sofia` — relato de incidente com secret global e motivação para secret individual
- Transcrição: `[09:21] Sofia` — grace period de 24h para rotação
- Transcrição: `[09:23] Sofia` — TLS obrigatório (HTTPS) como camada complementar
- Transcrição: `[09:26] Marcos` — documentação no portal do cliente para implementação do receptor
- Código: `src/middlewares/auth.middleware.ts` — padrão de autenticação existente reutilizado nos endpoints do módulo de webhooks

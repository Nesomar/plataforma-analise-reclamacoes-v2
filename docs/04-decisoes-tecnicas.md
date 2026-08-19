# 04 — Decisões Técnicas e Requisitos Não-Funcionais

## Stack

### Definida pelo diagrama **[DIAGRAMA]**

| Camada | Tecnologia |
|---|---|
| Frontend | React |
| Entrada HTTP | Amazon API Gateway |
| Mensageria | Amazon SQS — 3 filas principais + 3 DLQs |
| Compute (microserviços) | Amazon ECS |

### Definida pelo time **[DECIDIDO]**

| Camada | Tecnologia | Observação |
|---|---|---|
| Backend | **Python 3.13** | Ambos os microserviços e as Lambdas |
| Banco de dados | **Amazon DynamoDB** | Tabela única `solicitacoes` |
| LLM | **Google Gemini `gemini-3.6-flash`** | Externo à AWS |
| Autenticação | **Lambda Authorizer** no API Gateway | Python |
| Emulação AWS local | **MiniStack** | Ver `06-ambiente-local.md` |

### Complementos propostos **[SUPOSIÇÃO — confirmar]**

| Item | Proposta | Justificativa |
|---|---|---|
| Framework HTTP (consulta de status) | FastAPI + Uvicorn | Async nativo, validação por Pydantic, OpenAPI automático |
| Validação de schemas | Pydantic v2 | Contratos de API, mensagens SQS e saída do LLM em um só lugar |
| SDK AWS | boto3 (Lambdas) + aioboto3 (workers ECS) | Consumidores de fila se beneficiam de I/O assíncrono |
| SDK Gemini | `google-genai` | SDK oficial; `from google import genai` |
| Gerenciamento de dependências | uv + pyproject.toml | Rápido e reprodutível; `uv.lock` versionado |
| Lint / format / tipos | ruff + mypy (modo estrito) | — |
| Testes | pytest + pytest-asyncio | — |
| IaC | AWS CDK (Python) | Mesma linguagem do backend; MiniStack declara compatibilidade com CDK, Terraform e Pulumi |
| CI/CD | GitHub Actions | — |

> Sobre IaC: o MiniStack anuncia compatibilidade com CDK, mas a documentação e os exemplos dele
> enfatizam **Terraform**. Se o `cdk deploy` contra o MiniStack se mostrar instável na primeira
> semana, troque para Terraform sem hesitar — é a opção com mais tração no ecossistema do
> emulador. Registre a troca como emenda ao ADR-010.

## ADRs

### ADR-001 — Comunicação entre microserviços exclusivamente por SQS
**Status:** aceito (imposto pelo diagrama).
**Decisão:** MS Solicitação e MS Antifraude não expõem nem consomem HTTP um do outro.
**Consequências:** desacoplamento temporal; em troca, correlação manual via `correlationId` e
entrega *at-least-once* a ser tratada.

### ADR-002 — API assíncrona com 202 Accepted
**Status:** aceito. `POST /v1/solicitacoes` responde `202` com protocolo. Nenhum endpoint
bloqueia esperando o LLM.

### ADR-003 — MS Solicitação como único dono do estado
**Status:** aceito. Apenas ele escreve na tabela `solicitacoes`. O MS Antifraude não tem acesso
IAM a essa tabela.

### ADR-004 — Mecanismo de notificação do frontend (passo 9)
**Status:** **pendente**. O diagrama continua registrando "Websocket / Polling" sem escolher.
**Recomendação:** começar com polling em `GET /v1/solicitacoes/{id}` (3 s com backoff) e tratar
push como feature separada. O MiniStack emula API Gateway v2 com data plane, mas WebSocket API
não aparece na lista de serviços suportados — o que torna o polling também a opção mais
testável localmente.

### ADR-005 — DLQ dedicada por fila
**Status:** aceito. **[DIAGRAMA V2]**
**Decisão:** as três filas principais têm redrive policy com `maxReceiveCount = 5` e DLQ
própria. Nenhuma DLQ tem consumidor automático. Alarme no CloudWatch quando
`ApproximateNumberOfMessagesVisible > 0` em qualquer DLQ.
**Consequências:** mensagem em DLQ é incidente. É preciso um procedimento documentado de
inspeção e redrive manual, e o payload precisa carregar contexto suficiente para diagnóstico
(`correlationId`, `tentativa`, motivo da última falha em message attribute).

### ADR-006 — Autenticação via Lambda Authorizer
**Status:** aceito. **[DECIDIDO]**
**Decisão:** um Lambda Authorizer em Python protege as rotas do API Gateway. Ele valida o token
do cliente e devolve uma IAM policy (`Allow`/`Deny`) mais um `context` com `sub` do usuário e
`tenantId`.
**Detalhes:**
- Tipo **REQUEST** (não TOKEN), para poder inspecionar header e caminho — necessário porque a
  autorização de `GET /v1/solicitacoes/{id}` depende de o recurso pertencer ao chamador.
- Cache de autorização habilitado (TTL 300 s) com chave de cache pelo header `Authorization`.
- O `context` retornado é injetado na integração e propagado como identidade confiável — o
  backend **nunca** confia em um `userId` vindo do corpo da requisição.
- Um token inválido resulta em `401`; token válido sem permissão sobre o recurso, em `403`.
**Consequências:** cold start do authorizer entra no p95 do passo 1 — daí o cache. Ver o risco
de emulação em `06-ambiente-local.md`.

### ADR-007 — Observabilidade
**Status:** aceito. Logs estruturados em JSON com `correlationId` em toda linha
(`structlog`); tracing distribuído com OpenTelemetry propagado por message attributes do SQS;
métricas de negócio: taxa de aprovação, taxa de `INDETERMINADO`, latência e **custo por análise
em tokens do Gemini**.

### ADR-008 — Minimização de dados enviados ao LLM
**Status:** aceito, e **reforçado** pela escolha do Gemini.
**Contexto:** o Gemini é um serviço do Google, fora da AWS. Os dados cruzam a fronteira do
provedor de nuvem, o que muda a avaliação de privacidade e possivelmente a de conformidade
(LGPD — transferência internacional de dados pessoais).
**Decisão:** documento em hash SHA-256, nome removido, data de nascimento convertida em faixa
etária, e-mail reduzido ao domínio. O prompt enviado é registrado para auditoria com retenção
definida, nunca em log de aplicação.
**Pendência jurídica:** confirmar com o time de privacidade se a base legal e o aviso de
transferência internacional estão cobertos antes do primeiro ambiente com dado real.

### ADR-009 — Lambda Publisher entre API Gateway e SQS
**Status:** aceito. **[DECIDIDO — desvio consciente do diagrama]**
**Contexto:** o diagrama sugere integração direta API Gateway → SQS (integração de serviço AWS).
Isso funciona na AWS real, mas o MiniStack documenta para API Gateway v1/v2 apenas integrações
**MOCK** e **Lambda proxy** — integração direta com serviço AWS não consta na lista. Manter a
integração direta criaria um trecho do fluxo impossível de testar localmente, logo no ponto de
entrada.
**Decisão:** uma Lambda Python fina entre o API Gateway e a fila. Ela valida o corpo com
Pydantic, resolve a `Idempotency-Key`, gera o `idSolicitacao` (ULID), publica na
`SQS Solicitações` e devolve o `202`.
**Consequências:** paridade local completa; validação de contrato e resposta de erro de campo
ficam mais fáceis do que com VTL; em troca, mais um cold start no caminho e uma unidade a mais
para operar. A Lambda continua sem lógica de domínio — ela não decide nada sobre a proposta.
**Alternativa rejeitada:** integração direta com stub local caseiro para o trecho — mais código
de teste do que a própria Lambda.

### ADR-010 — MiniStack como único emulador AWS, decisão fechada
**Status:** aceito, **decisão final**. **[DECIDIDO]**
**Contexto:** este é um projeto de estudo. Ele nunca é implantado em AWS real — roda inteiramente
local, via Docker. Avaliamos duas alternativas em 19/08/2026 — `floci` e `fakecloud` — sob o
critério que mais importa aqui: Lambda Authorizer REQUEST + `Authorization: Bearer` em REST API
v1. `floci` tem bug **confirmado e aberto** que quebra esse cenário (issue #2050 do
`floci-io/floci` rejeita qualquer header `Authorization` na REST API v1) — descartado por falha
técnica. `fakecloud` lista suporte explícito a API Gateway v1 e tem SDK Python oficial, mas é
licenciado **AGPL-3.0-or-later** — descartado por escopo: um projeto sem distribuição nem
produto não tem motivo para trocar MIT por copyleft de rede só para ganhar um SDK de asserção.
**Decisão:** MiniStack é o único emulador AWS deste projeto, sem plano de reavaliação salvo
mudança de escopo (ex.: se o projeto deixar de ser só de estudo). Se o risco do Lambda
Authorizer (ver `docs/06-ambiente-local.md`) se confirmar, a resposta é o plano B do middleware
compartilhado — não a troca de emulador. Detalhe completo e tabela comparativa em
`docs/06-ambiente-local.md`, apêndice "Alternativas avaliadas".

### ADR-012 — Skills obrigatórias por tipo de tarefa
**Status:** aceito. **[DECIDIDO]**
**Decisão:** todo trabalho de frontend (componente, tela ou fluxo React) é feito com as skills
`frontend-design`, `react-expert` e `ui-ux-prox-max` carregadas antes de escrever código. Todo
trabalho de backend Python usa `fullstack-dev-skills` da mesma forma. Ao final de qualquer
tarefa de desenvolvimento — frontend ou backend — as skills `security-guidance` e `code-review`
rodam nessa ordem, antes de a tarefa ser considerada concluída.
**Consequências:** a ordem importa — segurança revisa antes do code review geral, porque uma
correção de segurança pode mudar trechos que o code review ainda vai olhar. Detalhe operacional
em `docs/07-fluxo-qualidade.md`.

### ADR-013 — Pipeline de CI/CD com abertura automática de PR
**Status:** aceito. **[DECIDIDO]**
**Decisão:** todo `push` para um branch que não seja `main` dispara lint, checagem de tipos e
testes (unitários e de integração contra o MiniStack em container de serviço). Se tudo passar e
não existir PR aberto para aquele branch, um PR para `main` é criado automaticamente. O mesmo
conjunto de testes roda de novo como *status check* obrigatório do próprio PR — a criação
automática do PR não substitui o gate de qualidade, só remove o passo manual de abrir o PR.
**Consequências:** commits diretos em `main` não passam por esse fluxo — a convenção deste
projeto é sempre trabalhar em branch e nunca commitar direto em `main`. Detalhe e YAML completo
em `docs/07-fluxo-qualidade.md`.

### ADR-011 — Escolha do modelo `gemini-3.6-flash`
**Status:** aceito. **[DECIDIDO]**
**Contexto:** `gemini-3.6-flash` foi lançado em 21/07/2026 como modelo estável, com janela de
1.048.576 tokens de entrada e 65.536 de saída, suportando structured output e function calling.
O Google lançou o `gemini-3.7-flash` em 13/08/2026, então o 3.6 já não é o Flash mais recente —
mas segue listado como estável, o que é o que importa para produção.
**Decisão:** fixar o ID estável `gemini-3.6-flash` em configuração, nunca um alias `-latest`.
Trocar de modelo é mudança de configuração + reexecução da suíte de casos de referência, nunca
um hot-swap silencioso.
**Notas de API específicas do Gemini 3.x** (mudaram em relação ao 2.x — não copie código antigo):
- `temperature`, `top_p` e `top_k` não são usados.
- `thinking_budget` foi substituído por `thinking_level`, com valores `"medium"` ou `"high"`.
- `candidate_count` não é suportado.
- Instruções de sistema vão em `system_instruction`.
**Custo:** a documentação oficial do tier pago padrão indica US$ 1,50 por milhão de tokens de
entrada e US$ 7,50 por milhão de saída; agregadores terceiros publicam valores diferentes.
Trate qualquer número como estimativa e **meça o custo real por análise em produção** — é uma
das métricas de negócio do ADR-007.

## Requisitos não-funcionais

### Resiliência
- Entrega SQS é *at-least-once*: todo consumidor é idempotente, deduplicando por
  `correlationId` + tipo de mensagem, com `ConditionExpression` no DynamoDB.
- Chamada ao Gemini com timeout duro de **20 s** **[SUPOSIÇÃO — calibrar com o p99 medido]** e
  no máximo 2 retentativas com backoff exponencial e jitter.
- Circuit breaker no cliente do Gemini: circuito aberto, a mensagem volta para a fila (não é
  deletada) em vez de virar `INDETERMINADO` em massa.
- Falha técnica **nunca** vira `REPROVADA` — vira `EM_ANALISE_MANUAL`.
- Visibility timeout de cada fila ≥ 6× o p99 de processamento do consumidor. Para a
  `SQS Análise Antifraude`, isso significa levar em conta a latência do Gemini mais os retries:
  com timeout de 20 s e 2 retries, o p99 de processamento passa de 60 s, exigindo visibility
  timeout na casa dos 360 s. **Dimensionar errado aqui causa processamento duplicado.**
- `maxReceiveCount = 5` em todas as filas, com DLQ dedicada (ADR-005).

### Segurança
- Chave da API do Gemini em AWS Secrets Manager, carregada em memória no start da task e
  renovada por rotação. Nunca em variável de ambiente em texto claro, nunca em imagem Docker.
- Filas e tabela criptografadas em repouso (KMS); TLS em trânsito.
- IAM com menor privilégio por task role: o MS Antifraude não acessa a tabela `solicitacoes`;
  o MS Solicitação não acessa o segredo do Gemini.
- Saída para a internet (Gemini) apenas via NAT Gateway em subnet privada, com egress restrito
  ao domínio da API do Google.
- **Prompt injection:** dados do usuário entram no prompt como conteúdo delimitado, jamais como
  instrução. A resposta do modelo é dado não confiável, validada contra schema Pydantic antes de
  qualquer uso.

### Performance e escala
- ECS Fargate com auto-scaling por `ApproximateNumberOfMessagesVisible`, não por CPU — o worker
  do MS Antifraude fica ocioso esperando I/O do Gemini, então CPU não é sinal de carga.
- Processamento assíncrono em lote no MS Antifraude, respeitando o rate limit da conta Google.
- Cache do Lambda Authorizer com TTL de 300 s para não pagar cold start a cada requisição.

### Qualidade
- Cobertura mínima de 80% nas regras de domínio (máquina de estados, idempotência, parsing do
  veredito). **[SUPOSIÇÃO]**
- Testes de integração contra MiniStack para SQS, DynamoDB, Lambda e API Gateway.
- Cliente do Gemini substituído por dublê de contrato nos testes; um conjunto de **casos de
  referência** (entrada conhecida → veredito esperado) roda a cada mudança de prompt ou de
  modelo, impedindo regressão silenciosa de qualidade.
- Ao menos um teste ponta a ponta cobrindo o caminho 1→9 e um cobrindo o caminho de falha até a
  DLQ.

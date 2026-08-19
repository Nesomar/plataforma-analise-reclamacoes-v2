# Prompt para `/speckit.plan`

Rode depois do `/speckit.specify` e do `/speckit.clarify`, dentro da mesma feature.

---

/speckit.plan

A arquitetura está em `docs/02-arquitetura.md`, os contratos em `docs/03-contratos-eventos.md`,
as decisões em `docs/04-decisoes-tecnicas.md` e o ambiente local em `docs/06-ambiente-local.md`.
Siga-os. Se algo neles conflitar com a spec desta feature, pare e aponte o conflito em vez de
escolher por conta própria.

**Lembrete de escopo:** este é um projeto de estudo. Nada aqui é implantado em AWS real — toda
a infraestrutura abaixo roda contra o MiniStack, em Docker, local. `AWS_ENDPOINT_URL` aponta
sempre para `http://localhost:4566`.

**Infraestrutura (fixa, emulada no MiniStack):**

- AWS, com Amazon API Gateway REST como única entrada HTTP pública, protegido por um
  **Lambda Authorizer do tipo REQUEST** (Python), com cache de 300 s.
- Uma **Lambda Publisher** (Python) entre o API Gateway e a fila de entrada — ela valida o corpo,
  gera o `idSolicitacao` e publica. Não há integração direta API Gateway → SQS (ADR-009).
- Amazon SQS: três filas principais (`solicitacao-entrada`, `antifraude-analise`,
  `antifraude-resultado`), **cada uma com DLQ dedicada**, `maxReceiveCount = 5`, sem consumidor
  automático nas DLQs.
- Amazon ECS Fargate para os dois microserviços, com auto-scaling por
  `ApproximateNumberOfMessagesVisible` — nunca por CPU.
- Amazon DynamoDB, tabela única `solicitacoes`, PK `pk` / SK `sk`, transição de estado por
  `UpdateItem` com `ConditionExpression`.
- **Google Gemini `gemini-3.6-flash`** como provedor de LLM. É externo à AWS: a chamada sai da
  VPC por NAT Gateway, a chave fica no Secrets Manager, e só o MS Antifraude tem acesso a ela.

**Stack de aplicação:**

- **Python 3.13**, tipagem estrita (mypy strict), gerenciado com uv (`pyproject.toml` + `uv.lock`).
- FastAPI + Uvicorn para o serviço HTTP de consulta de status no MS Solicitação.
- Consumidores SQS com long polling e processamento em lote, usando aioboto3.
- Pydantic v2 para toda validação: corpo da API, envelope e payloads das mensagens, e saída do
  Gemini.
- SDK `google-genai` para o Gemini, encapsulado atrás de um protocolo `AnalisadorDeRisco`.
- ruff (lint + format), pytest, pytest-asyncio.
- **MiniStack** (`ministackorg/ministack`, porta 4566) para SQS, DynamoDB, Lambda, API Gateway,
  Secrets Manager e CloudWatch Logs em dev e CI. Endpoint sempre via `AWS_ENDPOINT_URL`.
- AWS CDK em Python para IaC. <!-- AJUSTE para Terraform se o CDK contra MiniStack se mostrar instável -->
- GitHub Actions: ruff, mypy, testes unitários, testes de integração contra MiniStack, `cdk synth`.

**Organização do código (monorepo Python):**

```
apps/
  ms_solicitacao/      # orquestrador, dono do estado (worker SQS + API FastAPI de status)
  ms_antifraude/       # worker de análise via Gemini
  lambda_authorizer/   # validação de token → policy + context
  lambda_publisher/    # validação de payload → publicação na fila de entrada
  frontend/            # React
packages/
  contracts/           # modelos Pydantic da API, das mensagens e do veredito — fonte única
  infra_aws/           # fábrica de clientes boto3 com AWS_ENDPOINT_URL, helpers de SQS/DDB
  observability/       # structlog, propagação de correlationId, OpenTelemetry
infra/                 # CDK
```

Cada microserviço em camadas: `dominio` (entidades e regras, sem boto3 nem genai),
`aplicacao` (casos de uso), `adaptadores` (SQS, DynamoDB, Gemini, HTTP). **O pacote `dominio`
não pode importar boto3, google-genai nem FastAPI** — se importar, a camada está errada.

**Exigências obrigatórias no plano:**

1. Idempotência de todo consumidor, com mecanismo explícito: dedupe por `correlationId` +
   `UpdateItem` com `ConditionExpression` sobre `status`.
2. `correlationId` propagado por message attributes do SQS e presente em toda linha de log.
3. Timeout de 20 s, 2 retentativas com backoff exponencial e jitter, e circuit breaker no
   adaptador do Gemini. Circuito aberto devolve a mensagem à fila em vez de marcar
   `INDETERMINADO` em massa.
4. **Dimensionamento do visibility timeout de cada fila, com a conta escrita no plano.** Atenção
   à `antifraude-analise`: com timeout de 20 s e 2 retries, o p99 de processamento passa de 60 s.
   Errar aqui causa processamento duplicado.
5. Validação da saída do Gemini com Pydantic, uma tentativa corretiva, e fallback para
   `INDETERMINADO`.
6. Uso correto da API do Gemini 3.x: `response_schema`, `system_instruction`, `thinking_level`;
   **sem** `temperature`, `top_p`, `top_k` ou `candidate_count`.
7. Dublê do `AnalisadorDeRisco` com modos `fake` / `record` / `replay` / `live`, incluindo
   cenários de falha no modo `fake`.
8. IAM com menor privilégio por role: MS Antifraude sem acesso à tabela; MS Solicitação sem
   acesso ao segredo do Gemini.
9. Estratégia de teste por camada, incluindo um teste ponta a ponta do fluxo 1→9 e ao menos um
   teste que force uma mensagem até a DLQ.
10. Como o ambiente local reproduz esta feature no MiniStack — e, se algum recurso não for
    emulável, qual é o plano B explícito (ver os riscos em `docs/06-ambiente-local.md`).

**Restrições:**

- Nada de chamada HTTP síncrona entre MS Solicitação e MS Antifraude.
- Nada de lógica de negócio no API Gateway (sem VTL de transformação de domínio) nem nas Lambdas.
- Nada de `if ambiente == "local"` no código — só `AWS_ENDPOINT_URL`.
- O pacote `contracts` é a única definição de schema; nenhum serviço redefine tipos localmente.
- Nada de biblioteca nova sem justificativa registrada no plano.

**Skills e qualidade (não negociável, ver `docs/07-fluxo-qualidade.md`):**

- Tarefas de frontend: carregar `frontend-design`, `react-expert` e `ui-ux-prox-max` antes de
  escrever qualquer componente.
- Tarefas de backend Python: carregar `fullstack-dev-skills` antes de escrever qualquer módulo.
- Ao final de cada tarefa de implementação: rodar `security-guidance`, depois `code-review`,
  nessa ordem, antes de marcar a tarefa como concluída.
- O plano deve indicar, para cada grupo de tarefas gerado depois pelo `/speckit.tasks`, se é
  tarefa de frontend, backend ou mista — para que a skill certa seja carregada no momento certo.

**CI/CD:** o plano não precisa desenhar o pipeline — ele já existe em
`.github/workflows/ci.yml` (copiado de `workflows/ci.yml`) e roda testes contra o MiniStack a
cada push, abrindo PR automaticamente quando verde. Se a feature introduzir um novo pacote,
serviço ou comando de teste, o plano deve dizer explicitamente o que precisa ser adicionado ao
`ci.yml` (novo working-directory, novo comando de lint/teste, nova variável de ambiente) para
que a feature não fique sem cobertura no pipeline.

Antes de escrever o plano, liste as decisões que ainda dependem de mim e pare para confirmar.

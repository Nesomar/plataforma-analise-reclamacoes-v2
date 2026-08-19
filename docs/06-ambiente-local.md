# 06 — Ambiente Local (MiniStack)

**MiniStack** é um emulador AWS open source, licenciado sob MIT, criado como substituto do
LocalStack depois que este moveu serviços centrais para plano pago. Expõe 60+ serviços em uma
única porta (**4566**, a mesma do LocalStack) e é compatível com boto3, AWS CLI, Terraform,
CDK e Pulumi.

> **Decisão fechada:** este é um projeto de estudo. Ele **nunca** roda contra uma conta AWS
> real — só contra o MiniStack, em Docker, na máquina local. Duas alternativas foram avaliadas
> (`floci` e `fakecloud`, ver apêndice no fim deste documento) e descartadas: `floci` tem bug
> confirmado para o cenário de autenticação deste projeto; `fakecloud` é AGPL — licença que não
> faz sentido adotar aqui, ainda mais sem necessidade de implantação real. MiniStack é a única
> opção. Ver ADR-010.

- Repositório: `https://github.com/ministackorg/ministack`
- Imagem Docker: `ministackorg/ministack`
- Site e documentação: `https://ministack.org/`

O Gemini **não** é emulado — ver a seção "O Gemini não é emulado" no fim.

## Serviços que este projeto precisa

| Serviço | Cobertura declarada pelo MiniStack | Risco |
|---|---|---|
| **SQS** | Filas, FIFO, **DLQ**, batch, visibility timeout | 🟢 baixo |
| **DynamoDB** | Tabelas, CRUD, query, scan, transações, TTL, GSI | 🟢 baixo |
| **Lambda** | Execução real de Python, workers quentes, **event source mapping de SQS**, Layers | 🟢 baixo |
| **Secrets Manager** | CRUD, versionamento, rotação, resource policies | 🟢 baixo |
| **IAM / STS** | Usuários, roles, policies, AssumeRole | 🟡 emuladores são permissivos — não serve de prova de menor privilégio |
| **API Gateway v1 (REST)** | Recursos, métodos, integrações, stages, MOCK, **Lambda proxy formato 1.0**, data plane | 🟡 ver riscos abaixo |
| **CloudWatch Logs** | Grupos, streams, retenção, filtros | 🟢 baixo |
| **ECS** | `RunTask` inicia containers Docker reais | 🟡 útil, mas rodar os workers com `python -m` é mais rápido em dev |
| **CloudFormation** | Ciclo de vida de stacks, change sets, 123 tipos de recurso | 🟡 base do `cdk deploy` local |

## Riscos a validar na primeira semana

Pontos onde a emulação pode não cobrir o que o projeto precisa. **Valide cada um com uma prova
de conceito de meia hora antes de construir em cima.**

### 1. Lambda Authorizer no API Gateway emulado 🔴 crítico

A documentação do MiniStack lista, para API Gateway v1, integrações **MOCK** e **Lambda proxy**,
e menciona busca de JWKS para JWT — mas **não afirma explicitamente suportar authorizers do tipo
REQUEST/TOKEN**. Como o ADR-006 depende disso, é a incerteza mais cara do ambiente local.

**Plano B:** mover a verificação para um middleware da aplicação, ativado por variável de
ambiente. Em `local`, o middleware valida o token e popula a identidade; em AWS, ele apenas lê a
identidade já injetada pelo `context` do authorizer. A lógica de validação vive em um módulo
compartilhado, testado por unidade — assim authorizer real e middleware local nunca divergem em
regra de negócio.

**Sobre trocar de emulador — decisão já fechada, ver Apêndice.** `floci` foi descartado por bug
confirmado (issue aberta) que quebra exatamente este cenário. `fakecloud` foi descartado por
licença (AGPL) e por ser desnecessário — este projeto não implanta em lugar nenhum além do
MiniStack, então o SDK de asserção do `fakecloud` não paga o custo da licença diferente.

### 2. Integração API Gateway → SQS 🔴 já endereçado

Não consta suporte a integração direta com serviço AWS. É exatamente por isso que o **ADR-009**
introduz o Lambda Publisher: com Lambda proxy, o passo 2 fica testável localmente.

### 3. WebSocket API 🟡

Não aparece na lista de serviços suportados. Reforça a recomendação do ADR-004 de começar com
polling.

### 4. Testcontainers 🟡

O módulo oficial de Testcontainers do MiniStack é **para Java**
(`org.ministack:testcontainers-ministack`, no Maven Central). Não há módulo Python oficial.
Para pytest, use uma destas rotas:

- `docker compose up -d ministack` em fixture de sessão do pytest, com healthcheck em
  `http://localhost:4566/_localstack/health`;
- ou `testcontainers-python` com `DockerContainer("ministackorg/ministack").with_exposed_ports(4566)`
  — funciona por ser container genérico, só não há açúcar sintático pronto.

## Setup

### `docker-compose.yml`

```yaml
services:
  ministack:
    image: ministackorg/ministack   # fixe uma tag específica em CI, nunca latest
    ports:
      - "4566:4566"
    volumes:
      - ./scripts/init-aws.sh:/etc/ministack/init/ready.d/init-aws.sh
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
      interval: 5s
      timeout: 3s
      retries: 10
```

O MiniStack suporta init scripts com convenção compatível com a do LocalStack — o script em
`ready.d` roda quando o emulador fica pronto. Confirme o caminho exato na documentação da versão
que você fixar.

### Configuração dos clientes

Regra única: **o endpoint vem do ambiente, nunca de um `if`**.

```python
# packages/infra_aws/clients.py
import os
import boto3

def client(service: str):
    return boto3.client(
        service,
        endpoint_url=os.getenv("AWS_ENDPOINT_URL"),  # None em produção → endpoint real da AWS
        region_name=os.getenv("AWS_REGION", "us-east-1"),
    )
```

```bash
# .env.local
AWS_ENDPOINT_URL=http://localhost:4566
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
AWS_REGION=us-east-1
GEMINI_MODE=fake
```

### `scripts/init-aws.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
AWS="aws --endpoint-url=http://localhost:4566 --region us-east-1"

criar_fila_com_dlq() {
  local nome="$1"
  $AWS sqs create-queue --queue-name "${nome}-dlq"
  local dlq_url dlq_arn
  dlq_url=$($AWS sqs get-queue-url --queue-name "${nome}-dlq" --query QueueUrl --output text)
  dlq_arn=$($AWS sqs get-queue-attributes --queue-url "$dlq_url" \
    --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)
  $AWS sqs create-queue --queue-name "${nome}" --attributes "{
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"${dlq_arn}\\\",\\\"maxReceiveCount\\\":\\\"5\\\"}\",
    \"VisibilityTimeout\": \"360\"
  }"
}

criar_fila_com_dlq local-solicitacao-entrada
criar_fila_com_dlq local-antifraude-analise
criar_fila_com_dlq local-antifraude-resultado

$AWS dynamodb create-table \
  --table-name local-solicitacoes \
  --attribute-definitions AttributeName=pk,AttributeType=S AttributeName=sk,AttributeType=S \
  --key-schema AttributeName=pk,KeyType=HASH AttributeName=sk,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

$AWS secretsmanager create-secret \
  --name local/gemini/api-key \
  --secret-string '{"apiKey":"chave-falsa-para-dev"}'
```

## O Gemini não é emulado

O MiniStack emula AWS. O **Google Gemini está fora desse escopo** — e o proxy Bedrock→OpenAI que
o MiniStack oferece não ajuda, porque o projeto não usa Bedrock.

Estratégia de dublê, com quatro modos selecionados por `GEMINI_MODE`:

| Modo | Comportamento | Uso |
|---|---|---|
| `fake` | Veredito determinístico por regras simples sobre a entrada | Dev local e CI — rápido, gratuito, previsível |
| `record` | Chama o Gemini de verdade e grava requisição/resposta em fixtures | Atualizar casos de referência |
| `replay` | Reproduz as fixtures gravadas | Testes de contrato em CI, sem chave nem custo |
| `live` | Chama o Gemini de verdade | Homologação e produção |

O dublê implementa **a mesma interface** do cliente real (`AnalisadorDeRisco`) e o código de
domínio não sabe qual está em uso. O modo `fake` precisa incluir cenários de falha — timeout,
JSON malformado, confiança baixa, HTTP 429 — senão o caminho de `INDETERMINADO` nunca é
exercitado em desenvolvimento.

## Camadas de teste

| Camada | Depende de | Cobre |
|---|---|---|
| Unitário | nada | máquina de estados, idempotência, parsing do veredito |
| Integração | MiniStack | publicação/consumo SQS, escrita condicional no DynamoDB, redrive para DLQ |
| Contrato do LLM | dublê (`replay`) | schema do veredito e todos os modos de falha |
| Casos de referência | Gemini real (fora do CI padrão) | qualidade das decisões após mudança de prompt ou de modelo |
| Ponta a ponta | MiniStack + dublê | fluxo completo 1→9 |

**Teste obrigatório de DLQ:** forçar o consumidor a falhar 5 vezes e verificar que a mensagem
chega à DLQ com `correlationId` e message attributes preservados. Um dos três é o suficiente
para provar o padrão; idealmente os três.

## Comandos do dia a dia

```bash
docker compose up -d                              # sobe o MiniStack e roda o init
uv run pytest tests/unit                          # domínio puro
uv run pytest tests/integration                   # contra o MiniStack
uv run python -m ms_solicitacao.worker            # consumidor da fila de entrada
uv run python -m ms_antifraude.worker             # consumidor da fila de análise
uv run uvicorn ms_solicitacao.api:app --reload    # API de consulta de status

aws --endpoint-url=http://localhost:4566 sqs list-queues
aws --endpoint-url=http://localhost:4566 dynamodb scan --table-name local-solicitacoes
```

## Limite honesto do emulador

MiniStack (como qualquer emulador) valida **integração**, não **produção**. Não reproduz
fielmente: enforcement de IAM, throttling, latência de rede, semântica exata de visibility
timeout sob concorrência alta, nem cold start. Verde local significa "a lógica está certa" —
que é exatamente o objetivo de um projeto de estudo. Não há homologação em AWS real prevista
neste projeto; se um dia isso mudar, essas lacunas são a lista do que precisaria ser revalidado.


## Apêndice — Alternativas avaliadas

Avaliadas em 19/08/2026, sob o critério que mais importa para este projeto: **Lambda Authorizer
tipo REQUEST + header `Authorization: Bearer` numa REST API (API Gateway v1)**.

### `floci` (`floci-io/floci`) — descartado para esta feature

MIT, ~24ms de startup, ~13 MiB em idle, 75 serviços, containers Docker reais para Lambda/RDS/ECS.
Em tese o mais rápido dos três. Na prática, três issues abertas no GitHub atingem exatamente o
nosso cenário:

- **[#2050](https://github.com/floci-io/floci/issues/2050)** (aberta há ~3 semanas): a emulação
  de REST API rejeita **qualquer** requisição com header `Authorization`, tratando-o sempre como
  tentativa de assinatura SigV4 — quebra qualquer esquema Bearer/JWT próprio, mesmo em rotas sem
  authorizer.
- **[#812](https://github.com/floci-io/floci/issues/812)**: authorizer Lambda tipo REQUEST em
  API Gateway v2 é **silenciosamente ignorado** — a requisição passa sem autorização.
- **[#874](https://github.com/floci-io/floci/issues/874)**: invocação por path em REST API
  retorna 404.

Diferente da incerteza "não documentado" do MiniStack, aqui é **bug confirmado e aberto** para o
desenho exato que este projeto precisa. Reavaliar quando essas issues fecharem — não antes.

### `fakecloud` (`faiscadev/fakecloud`) — descartado por licença e por escopo

105 serviços, 7.391 operações, conformidade Smithy declarada em 100% (nota: isso mede fidelidade
de *shape* de request/response contra o modelo da AWS — a própria documentação separa cobertura
de *control-plane* de *data-plane* por serviço na "parity matrix", e não confirmamos nela o
comportamento específico de runtime do REQUEST authorizer). Lista **API Gateway v1 (REST)**
explicitamente, com integração cruzada "API Gateway → Lambda" testada ponta a ponta. Tem **SDK
Python oficial** (`pip install fakecloud`), com introspecção útil para teste.

**Licença: AGPL-3.0-or-later**, diferente do MIT do MiniStack e do floci. Seria a alegação mais
explícita de suporte ao nosso cenário de autenticação entre os três — mas **este é um projeto de
estudo, rodando só localmente, sem distribuição nem produto**: não há motivo para trocar a
licença permissiva do MiniStack por uma copyleft de rede só para ganhar um SDK de asserção que
o pytest comum já resolve. Descartado por decisão de escopo, não por falha técnica.

| Critério | MiniStack | floci | fakecloud |
|---|---|---|---|
| Licença | MIT | MIT | **AGPL-3.0-or-later** |
| REST API v1 listada | ✅ | ✅ | ✅ (explícita) |
| REQUEST authorizer + Bearer | não confirmado | 🔴 **bug aberto** | não confirmado, sem bug conhecido |
| SDK Python de teste | não | não | ✅ oficial |
| Startup / footprint | ~2s / ~30 MB | ~24 ms / ~13 MiB | ~300 ms / ~10 MiB |
| Maturidade percebida | estabelecida | ativa, muitas issues recentes | mais nova, menos issues encontradas (pode ser menos uso, não necessariamente menos bugs) |

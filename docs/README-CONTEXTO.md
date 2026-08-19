# Contexto Inicial — Sistema de Solicitação com Antifraude via LLM

Arquivos de contexto extraídos do diagrama `DiagramaSistemaSolicitacaoV2.xml`
(draw.io / AWS Architecture) mais as decisões de stack registradas nos ADRs, para alimentar o
fluxo **Spec-Driven Development** do [GitHub Spec Kit](https://github.com/github/spec-kit)
usando o **Claude Code**.

**Stack decidida:** React · Python 3.13 · DynamoDB · SQS com DLQ · ECS Fargate ·
Lambda Authorizer · Google `gemini-3.6-flash` · MiniStack para AWS local.

> **[DIAGRAMA]** = explícito no arquivo · **[DECIDIDO]** = decisão posterior ao desenho ·
> **[SUPOSIÇÃO]** = inferido, precisa de validação sua.
>
> ⚠️ O diagrama V2 não mostra o Lambda Authorizer, o DynamoDB, e ainda rotula o LLM como
> "OpenAI / Bedrock" dentro da AWS. Ver "Divergências" em `docs/02-arquitetura.md`.

## Estrutura

```
.
├── README-CONTEXTO.md          ← este arquivo
├── CLAUDE.md                   ← contexto permanente do agente (copiar para a raiz do repo)
├── docs/
│   ├── 01-visao-geral.md       ← problema, escopo, atores, fora de escopo
│   ├── 02-arquitetura.md       ← componentes e fluxo 1→9 do diagrama (+ Mermaid)
│   ├── 03-contratos-eventos.md ← API REST, mensagens SQS, máquina de estados
│   ├── 04-decisoes-tecnicas.md ← stack, ADRs, requisitos não-funcionais
│   ├── 05-glossario.md         ← vocabulário ubíquo do domínio
│   └── 06-ambiente-local.md    ← MiniStack, bootstrap e estratégia de testes
└── speckit/
    ├── constitution.prompt.md  ← entrada para /speckit.constitution
    ├── specify.prompt.md       ← entrada para /speckit.specify
    ├── plan.prompt.md          ← entrada para /speckit.plan
    └── checklist.prompt.md     ← entrada para /speckit.checklist
```

## Passo a passo

### 1. Instalar o Specify CLI

Requer [uv](https://docs.astral.sh/uv/), Python 3.11+ e Git.

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
# ou, do PyPI:
uv tool install specify-cli
```

Troque `vX.Y.Z` pela tag mais recente em [Releases](https://github.com/github/spec-kit/releases)
(mantenha o `v` inicial).

### 2. Inicializar o projeto com integração Claude Code

```bash
specify init sistema-solicitacao --integration claude
cd sistema-solicitacao
```

Rode `specify integration list` para confirmar o identificador exato da integração
na versão que você instalou.

### 3. Copiar os arquivos de contexto para o repositório

```bash
cp CLAUDE.md ./CLAUDE.md                          # se já existir, faça merge em vez de sobrescrever
cp -r docs ./docs
cp -r speckit ./docs/speckit                      # opcional: manter os prompts versionados
mkdir -p .github/workflows
cp workflows/ci.yml .github/workflows/ci.yml
```

### Skills necessárias no Claude Code

Confirme que estas skills estão disponíveis na sua sessão/organização antes de começar a
implementar — elas são obrigatórias por tarefa, não opcionais (ver `docs/07-fluxo-qualidade.md`):

- `frontend-design`, `react-expert`, `ui-ux-prox-max` — carregadas antes de qualquer código de
  frontend.
- `fullstack-dev-skills` — carregada antes de qualquer código de backend Python.
- `security-guidance`, `code-review` — rodam nessa ordem ao final de toda tarefa de
  desenvolvimento, frontend ou backend.

### Branch protection (uma vez, manual)

Depois do primeiro push do `ci.yml`, configure em **Settings → Branches → main** a exigência dos
status checks `test-backend` e `test-frontend` antes de permitir merge. Isso não é automatizável
pelo YAML — é uma configuração do repositório.

O `CLAUDE.md` é carregado automaticamente pelo Claude Code em toda sessão; os arquivos
em `docs/` são referenciados por ele e lidos sob demanda.

### 4. Executar o fluxo Spec Kit

Abra o Claude Code na raiz do projeto e rode, nesta ordem:

| Ordem | Comando | Entrada |
|---|---|---|
| 1 | `/speckit.constitution` | conteúdo de `speckit/constitution.prompt.md` |
| 2 | `/speckit.specify` | conteúdo de `speckit/specify.prompt.md` |
| 3 | `/speckit.clarify` | (sem argumento — responde as perguntas do agente) |
| 4 | `/speckit.plan` | conteúdo de `speckit/plan.prompt.md` |
| 5 | `/speckit.tasks` | (sem argumento) |
| 6 | `/speckit.checklist` | conteúdo de `speckit/checklist.prompt.md` |
| 7 | `/speckit.analyze` | (sem argumento) |
| 8 | `/speckit.implement` | (sem argumento) |

Cada prompt em `speckit/` foi escrito para ser colado inteiro depois do comando.
Eles referenciam os arquivos de `docs/` por caminho, então o agente lê o detalhe
sem estourar o contexto.

### 5. Ordem sugerida de features

O sistema é grande demais para um único `/speckit.specify`. Sugestão de fatiamento —
uma feature (e portanto um branch/pasta em `specs/`) por vez:

0. `autorizacao` — Lambda Authorizer, identidade e isolamento entre clientes
1. `ingestao-solicitacao` — API Gateway → SQS Solicitações → MS Solicitação (fluxos 1–3)
2. `orquestracao-antifraude` — MS Solicitação ↔ MS Antifraude via filas (fluxos 4–5, 7–8)
3. `analise-llm` — MS Antifraude → Gemini, prompt e validação do veredito (fluxo 6)
4. `notificacao-frontend` — WebSocket/polling de status (fluxo 9)
5. `resiliencia-observabilidade` — DLQs, redrive, retries, idempotência, tracing

A autorização vem primeiro porque o `clienteId` que ela produz é pré-requisito de todas as
demais: sem ele não há como isolar solicitações por cliente.

O `specify.prompt.md` traz o texto pronto das seis features.

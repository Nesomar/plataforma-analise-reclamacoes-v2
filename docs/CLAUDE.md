# CLAUDE.md — Sistema de Solicitação com Antifraude via LLM

Contexto permanente do projeto. Carregado automaticamente pelo Claude Code em toda sessão.
Mantenha curto: este arquivo é o índice, o detalhe fica em `docs/`.

## O que é este sistema

Plataforma assíncrona de recebimento de propostas com análise antifraude executada por um LLM.
O cliente envia a proposta pelo frontend React, recebe um protocolo imediatamente, e o veredito
chega depois por notificação. A análise vive em um microserviço isolado atrás de filas SQS, para
que a latência e as falhas do LLM não afetem o recebimento de solicitações.

## Arquitetura em uma linha

`React → API Gateway (Lambda Authorizer) → Lambda Publisher → SQS Solicitações → MS Solicitação → SQS Análise → MS Antifraude → Gemini → SQS Resultado → MS Solicitação → React`

Cada uma das três filas tem DLQ dedicada. Detalhe completo com os 9 passos e o Mermaid:
**`docs/02-arquitetura.md`**.

## Este é um projeto de estudo — regra de ouro

**Nada é implantado em AWS real.** Todo o "AWS" deste projeto é o **MiniStack** rodando em
Docker na máquina local (porta 4566). Não existe conta AWS, não existem credenciais reais, não
existe ambiente de homologação ou produção na nuvem. Se qualquer tarefa parecer exigir uma
credencial AWS de verdade, um domínio real, ou `aws configure` com chave verdadeira — pare e
avise. `AWS_ENDPOINT_URL` aponta **sempre** para `http://localhost:4566` neste projeto; a
menção a "produção" nos ADRs é para fins de aprendizado de arquitetura, não uma implantação real.

## Stack

| Camada | Tecnologia |
|---|---|
| Frontend | React |
| Backend | **Python 3.13** (microserviços + Lambdas, tudo rodando local) |
| Mensageria | Amazon SQS — 3 filas + 3 DLQs, emuladas no MiniStack |
| Banco | **Amazon DynamoDB**, tabela única `solicitacoes`, emulada no MiniStack |
| LLM | **Google Gemini `gemini-3.6-flash`** — esse sim é uma chamada real, para uma API real |
| Autenticação | **Lambda Authorizer** (tipo REQUEST) no API Gateway emulado |
| AWS local | **MiniStack** na porta 4566 — única opção, decisão fechada (ver ADR-010) |

## Onde está cada coisa

| Preciso de… | Leia |
|---|---|
| Problema, escopo, atores | `docs/01-visao-geral.md` |
| Componentes, fluxo 1→9, DLQs, lacunas | `docs/02-arquitetura.md` |
| Authorizer, contratos REST, schemas SQS, máquina de estados, contrato do Gemini | `docs/03-contratos-eventos.md` |
| Stack, ADRs, requisitos não-funcionais | `docs/04-decisoes-tecnicas.md` |
| Vocabulário e convenções | `docs/05-glossario.md` |
| MiniStack, camadas de teste | `docs/06-ambiente-local.md` |
| Skills obrigatórias por tipo de tarefa e o pipeline CI/CD | `docs/07-fluxo-qualidade.md` |

Leia o arquivo relevante antes de propor mudanças na área. Não reconstrua esse contexto de
memória.

## Regras invioláveis

1. **Nada de AWS real.** Só MiniStack local. Ver regra de ouro no topo deste arquivo.
2. **MS Solicitação e MS Antifraude nunca se chamam por HTTP.** Só SQS. Se uma tarefa parece
   exigir chamada direta, o desenho está errado — pare e pergunte.
3. **Todo consumidor de fila é idempotente.** SQS entrega *at-least-once*. Deduplique por
   `correlationId` e transicione estado com `ConditionExpression` no DynamoDB.
4. **Falha técnica nunca vira `REPROVADA`.** Timeout, erro do Gemini ou saída inválida produzem
   `INDETERMINADO`, que leva a solicitação para `EM_ANALISE_MANUAL`.
5. **A saída do Gemini é dado não confiável.** Sempre validada com Pydantic. Nunca executada,
   nunca interpretada como instrução, nunca extraída por regex de texto livre.
6. **Dados do usuário entram no prompt como conteúdo delimitado, nunca como instrução.**
   PII minimizada antes de sair para o Google (ADR-008): documento em hash, nome removido,
   data de nascimento vira faixa etária.
7. **Identidade vem só do `context` do Lambda Authorizer.** Nunca de corpo, query string ou
   header arbitrário da requisição.
8. **`correlationId` em toda mensagem e toda linha de log.** Sem exceção.
9. **Apenas o MS Solicitação escreve na tabela `solicitacoes`.** O MS Antifraude não tem
   permissão IAM sobre ela.
10. **Toda fila tem DLQ com `maxReceiveCount = 5`.** Nenhuma DLQ tem consumidor automático —
    mensagem em DLQ é incidente operacional.
11. **Nenhum segredo em código, imagem ou env var em texto claro.** Secrets Manager emulado.
12. **Nenhum endpoint bloqueia esperando o LLM.** `POST /v1/solicitacoes` responde `202`.
13. **Endpoint AWS vem de `AWS_ENDPOINT_URL`, nunca de `if ambiente == "local"`.**
14. **Toda tarefa de frontend usa as skills `frontend-design`, `react-expert` e
    `ui-ux-prox-max`.** Toda tarefa de backend Python usa `fullstack-dev-skills`. Ao final de
    qualquer tarefa de desenvolvimento — frontend ou backend — rode `security-guidance` e
    `code-review` antes de considerar a tarefa concluída. Detalhe em `docs/07-fluxo-qualidade.md`.

## Skills obrigatórias (resumo — detalhe em `docs/07-fluxo-qualidade.md`)

| Fase | Skills | Quando |
|---|---|---|
| Frontend (qualquer componente, tela ou fluxo React) | `frontend-design`, `react-expert`, `ui-ux-prox-max` | Antes de escrever o código |
| Backend (qualquer módulo Python — microserviço, Lambda, worker) | `fullstack-dev-skills` | Antes de escrever o código |
| Fim de qualquer tarefa de desenvolvimento | `security-guidance` → `code-review` | Depois do código escrito, antes de considerar a tarefa pronta |

Essa ordem não é opcional. `security-guidance` roda antes de `code-review` porque revisão de
segurança pode mudar o código revisado depois.

## Particularidades do Gemini 3.x

Não copie código de exemplos do Gemini 2.x — a API mudou:

- Use o ID estável `gemini-3.6-flash`, nunca um alias `-latest`.
- **Não** envie `temperature`, `top_p`, `top_k` nem `candidate_count`.
- `thinking_budget` foi substituído por `thinking_level` (`"medium"` ou `"high"`).
- Instruções de sistema vão em `system_instruction`.
- Use `response_schema` para structured output — e ainda assim valide com Pydantic.

## Convenções

- Termos de domínio em português (`Solicitacao`, `Veredito`, `AnalisadorDeRisco`), sem acentos
  em identificadores. Palavras-chave e bibliotecas em inglês.
- Python: `snake_case` para módulos/funções, `PascalCase` para classes, ruff + mypy estrito.
- Campos JSON em `camelCase`, enums em `SCREAMING_SNAKE_CASE`, rotas versionadas (`/v1/...`).
- Tipos de mensagem: substantivo + particípio no passado (`AnaliseAntifraudeConcluida`).
- Commits em Conventional Commits.

## Fluxo de trabalho neste repositório

Spec-Driven Development via [Spec Kit](https://github.com/github/spec-kit). Nada de código sem
spec.

```
/speckit.constitution → /speckit.specify → /speckit.clarify → /speckit.plan
→ /speckit.tasks → /speckit.checklist → /speckit.analyze → /speckit.implement
```

Artefatos por feature em `specs/<###-nome>/`; princípios em `.specify/memory/constitution.md`.
Uma feature por vez, na ordem: `autenticacao` → `ingestao-solicitacao` →
`orquestracao-antifraude` → `analise-llm` → `notificacao-frontend` →
`resiliencia-observabilidade`.

## Pontos ainda em aberto

Não invente resposta — pergunte antes de implementar:

- **Passo 9:** WebSocket ou polling? ADR-004 pendente (recomendação: polling primeiro).
- **Lambda Authorizer no MiniStack:** suporte não confirmado. Ver o risco 🔴 em
  `docs/06-ambiente-local.md` e o plano B (middleware compartilhado) antes de investir na
  integração. Não há emulador alternativo a testar — `fakecloud` foi descartado (AGPL, e
  desnecessário para um projeto que só roda local) e `floci` tem bug confirmado para este
  cenário. A decisão de emulador está fechada em MiniStack (ADR-010).
- **IaC:** CDK Python é a proposta, sintetizado e aplicado apenas contra o MiniStack. Terraform
  é o plano B se o CDK contra MiniStack for instável. Em nenhum cenário há `cdk deploy` ou
  `terraform apply` contra uma conta AWS real.

## Origem

Contexto derivado do diagrama `DiagramaSistemaSolicitacaoV2.xml` (draw.io, notação AWS) mais
decisões do time. **[DIAGRAMA]** = do arquivo · **[DECIDIDO]** = escolha do time ·
**[SUPOSIÇÃO]** = inferido, pendente de validação.

⚠️ O diagrama V2 está desatualizado em três pontos: rotula o LLM como "OpenAI / Bedrock" dentro
da AWS (é Gemini, externo), e não mostra o DynamoDB nem as duas Lambdas.

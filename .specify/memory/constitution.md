<!--
Sync Impact Report
==================
Version change: N/A (template) → 1.0.0
Ratification: initial adoption

Modified principles: none (first version — template placeholders filled)

Added sections:
- Core Principles I–XII (arquitetura/domínio, não-negociáveis)
- Core Principles XIII–XIV (processo: skills obrigatórias, gate de CI)
- Governance: emendas, exceções justificadas, exigência de conformidade em plano

Removed sections: none

Templates requiring updates:
- ✅ .specify/templates/plan-template.md — Constitution Check gate expandido com checklist por princípio
- ✅ .specify/templates/spec-template.md — nota de contexto assíncrono/mensageria adicionada (não muda estrutura mandatória)
- ✅ .specify/templates/tasks-template.md — nota de categorias de tarefa dirigidas por princípio (observabilidade, idempotência, contratos, segurança)
- ✅ .specify/templates/checklist-template.md — genérico, sem alteração necessária
- ✅ docs/CLAUDE.md — já reflete os princípios (regras invioláveis 1–14); nenhuma contradição encontrada
- ⚠ .claude/skills/speckit-*/ — não inspecionados linha a linha nesta emenda; nenhuma referência quebrada identificada nas leituras feitas

Follow-up TODOs: nenhum. RATIFICATION_DATE fixada em 2026-08-18 (data desta criação, sem constituição anterior documentada).
-->

# Sistema de Solicitação com Antifraude via LLM Constitution

## Core Principles

### I. Desacoplamento por Mensageria

Microserviços NUNCA se chamam de forma síncrona. Toda comunicação entre domínios acontece
exclusivamente por fila SQS. O MS Solicitação e o MS Antifraude não expõem nem consomem HTTP um
do outro sob nenhuma circunstância. Qualquer proposta de chamada HTTP direta entre os dois
microserviços — inclusive "temporária", "só para debug" ou "só nesse caso especial" — é uma
violação arquitetural e deve ser rejeitada em design review, não implementada e depois corrigida.

**Racional**: o desacoplamento temporal via fila é o que permite que a latência e as falhas do
LLM externo não se propaguem para o caminho de recebimento de solicitações. Uma chamada síncrona
reintroduz exatamente o acoplamento que a arquitetura foi desenhada para eliminar.

### II. Idempotência Obrigatória

A entrega de mensagens SQS é *at-least-once*. Todo consumidor DEVE produzir o mesmo efeito ao
processar a mesma mensagem N vezes. Transições de estado usam escrita condicional
(`ConditionExpression` no DynamoDB) — nunca um `PutItem`/`UpdateItem` incondicional que sobrescreve
sem checar o estado atual. A API de escrita síncrona (`POST /v1/solicitacoes`) aceita e respeita
um cabeçalho `Idempotency-Key`. Deduplicação em consumidores é feita por `correlationId` + tipo de
mensagem.

**Racional**: at-least-once não é um detalhe de infraestrutura a ser tolerado — é a garantia
contratual da fila. Código que assume entrega exatamente-uma-vez está incorreto por construção,
mesmo que passe nos testes felizes.

### III. Falha Técnica Não é Decisão de Negócio

Indisponibilidade, timeout ou saída inválida do LLM JAMAIS resultam em reprovação da solicitação.
Esses cenários produzem um resultado `INDETERMINADO`, que encaminha o caso para
`EM_ANALISE_MANUAL` — revisão humana. O sistema falha em direção à revisão, nunca à negativa.
Nenhum código de tratamento de exceção do cliente do LLM pode, direta ou indiretamente, chegar a
um veredito `REPROVADA`.

**Racional**: confundir "o modelo não respondeu" com "o modelo reprovou" é um erro de
classificação com consequência real sobre uma pessoa. A ambiguidade técnica deve ser absorvida
pelo sistema, nunca repassada como decisão de negócio.

### IV. Saída de LLM é Entrada Não Confiável

Toda resposta do modelo é validada contra schema (Pydantic) antes de qualquer uso. A saída do LLM
NUNCA é executada, NUNCA vira instrução para outro componente, e NUNCA é extraída por regex de
texto livre — se o schema não valida, é falha técnica (ver Princípio III), não uma tentativa de
parsing alternativo. Dados do usuário entram no prompt como conteúdo delimitado, jamais como
instrução; o prompt deve ser estruturado de forma que texto fornecido pelo usuário não possa ser
interpretado pelo modelo como comando do sistema.

**Racional**: um LLM externo é, por definição, uma fronteira de confiança. Tratá-lo como código
confiável (via `eval`, regex heurístico ou concatenação direta de instrução) abre caminho para
prompt injection e para falhas silenciosas de parsing que produzem veredito errado com aparência
de sucesso.

### V. Rastreabilidade Ponta a Ponta

Um `correlationId` único é gerado na entrada do sistema e acompanha o fluxo do primeiro clique até
a notificação final. Ele está presente em toda mensagem SQS (como atributo, não só no corpo) e em
toda linha de log estruturado. Nenhum caminho de código — incluindo caminhos de erro, DLQ e
retries — pode perdê-lo ou substituí-lo por um novo identificador.

**Racional**: em um sistema assíncrono de múltiplos saltos, o `correlationId` é a única forma
prática de reconstruir o que aconteceu com uma solicitação específica. Perdê-lo em qualquer ponto
quebra a capacidade de diagnóstico exatamente onde ela é mais necessária — na fila de erro.

### VI. Privacidade por Minimização

Dados pessoais são reduzidos ao mínimo necessário antes de cruzar qualquer fronteira de serviço, e
especialmente antes de sair para o provedor de LLM externo à AWS. Documento é enviado como hash,
nome é removido, data de nascimento vira faixa etária, e-mail é reduzido ao domínio — ou
equivalente de minimização aprovado. Segredos (chave de API, credenciais) vivem apenas em cofre
gerenciado (Secrets Manager ou equivalente emulado); nunca em variável de ambiente em texto claro,
imagem de container ou repositório de código.

**Racional**: o Gemini é externo à AWS e ao país de origem dos dados — cruzar essa fronteira é uma
transferência internacional de dado pessoal, com implicações de conformidade que a minimização
reduz mas não elimina sozinha. Segredo em texto claro é a causa mais comum e mais evitável de
vazamento.

### VII. Contratos Explícitos e Versionados

Toda mensagem de fila e todo evento carregam um campo de tipo e um campo de versão (`tipo`,
`versao`, ou equivalente). Uma mudança incompatível em um contrato exige uma nova versão, com
período de convivência entre a versão antiga e a nova — nunca uma alteração in-place que quebra
consumidores existentes silenciosamente. Contratos são a fonte de verdade entre times: a
implementação segue o contrato publicado, o contrato não é inferido a partir do código de um único
consumidor.

**Racional**: MS Solicitação e MS Antifraude evoluem de forma independente, conectados apenas por
fila. Sem versionamento explícito, uma mudança em um lado quebra o outro de forma invisível até a
produção — ou, neste projeto, até o MiniStack.

### VIII. Qualidade Verificável

Regras de domínio — máquina de estados da solicitação, lógica de idempotência, parsing/validação
do veredito do LLM — têm teste automatizado escrito antes da implementação correspondente. O
caminho completo (passo 1 ao passo 9) tem ao menos um teste ponta a ponta, e o caminho de falha até
a DLQ tem ao menos um teste dedicado. Cobertura mínima de 80% no código de domínio.

**Racional**: a máquina de estados e a idempotência são exatamente os pontos onde um bug é mais
caro (efeito duplicado, veredito perdido) e mais fácil de não perceber em teste manual. Exigir
teste automatizado antes da implementação nessas áreas é a defesa contra reintroduzir o mesmo bug
de outra forma.

### IX. Observabilidade como Requisito, Não como Extra

Logs estruturados (JSON, com `correlationId` em toda linha), tracing distribuído propagado através
das filas (via message attributes) e métricas de negócio — taxa de aprovação, taxa de
`INDETERMINADO`, latência por etapa e custo por análise em tokens do LLM — fazem parte da
definição de pronto de qualquer feature. Uma feature sem essas três camadas não está concluída,
independentemente de passar nos testes funcionais.

**Racional**: em um sistema assíncrono com um custo variável por chamada externa (tokens do
Gemini), observabilidade não é diagnóstico pós-incidente — é o único jeito de saber, no dia a dia,
se o sistema está se comportando dentro do esperado e do orçamento.

### X. Resiliência Explícita

Toda fila tem DLQ dedicada com alarme (ex.: `ApproximateNumberOfMessagesVisible > 0`). Nenhuma DLQ
tem consumidor automático — reprocessamento a partir de uma DLQ é sempre um ato humano deliberado,
nunca um job agendado. Toda chamada externa (LLM, e qualquer outra API de terceiro) tem timeout
duro, retry com backoff exponencial e jitter, e circuit breaker — circuito aberto devolve a
mensagem à fila em vez de forçar um veredito. Escala do worker é dirigida por profundidade de fila
(`ApproximateNumberOfMessagesVisible`), não por CPU, quando o worker é predominantemente limitado
por I/O de chamada externa.

**Racional**: mensagem em DLQ representa um caso que já falhou pelo caminho normal — reprocessar
automaticamente sem entendimento humano do porquê arrisca repetir a mesma falha em massa. Escalar
por CPU um worker limitado por I/O externo não reage ao sinal real de carga.

### XI. Controle de Acesso que Falha Fechado

Nenhuma operação ocorre sem identidade verificada. A identidade do solicitante é determinada
exclusivamente pelo `context` devolvido pelo Lambda Authorizer a partir da credencial apresentada
— nada que o solicitante escreva no corpo da requisição, query string ou header arbitrário
influencia essa identidade. Indisponibilidade ou erro do mecanismo de controle de acesso resulta
em negação (`401`/`403`), nunca em liberação por padrão.

**Racional**: um sistema que confia em `userId` do corpo da requisição, ou que libera acesso
quando o authorizer falha, converte uma falha de infraestrutura em uma falha de segurança. Falhar
fechado é a única postura segura por padrão.

### XII. Paridade entre Ambiente Local e Produção sem Código Condicional

A configuração diferencia os ambientes (ex.: `AWS_ENDPOINT_URL` apontando para o MiniStack local
ou para AWS real); o código não. Nenhum adapter contém condicional de ambiente (`if ambiente ==
"local"`). Ao mesmo tempo, verde no emulador local (MiniStack) NUNCA é tratado como prova
suficiente de correção em nuvem real — permissões IAM, limites de serviço e comportamento sob
carga exigem validação em ambiente real antes de qualquer confiança de produção.

**Racional**: condicional de ambiente no código é a forma mais comum de código nunca testado em
produção rodar pela primeira vez em produção. E o inverso também é verdade: o emulador tem
limitações conhecidas (ex.: integrações de serviço AWS não suportadas, WebSocket API ausente) que
tornam "passou no MiniStack" uma condição necessária, não suficiente.

### XIII. Skills Obrigatórias por Tipo de Tarefa

Todo trabalho de frontend (componente, tela ou fluxo React) carrega as skills `frontend-design`,
`react-expert` e `ui-ux-prox-max`, nesta ordem, ANTES de qualquer código ser escrito. Todo
trabalho de backend Python carrega `fullstack-dev-skills` da mesma forma — antes da primeira
linha de código. Ao final de qualquer tarefa de desenvolvimento, frontend ou backend,
`security-guidance` roda antes de `code-review` — nessa ordem — antes de a tarefa ser considerada
concluída. Uma tarefa que mistura frontend e backend carrega as skills de ambos os lados antes de
escrever código, e roda o par `security-guidance`/`code-review` uma única vez ao final, sobre o
conjunto completo da mudança. Se uma skill exigida não estiver disponível na sessão, o trabalho
para e o fato é reportado — não prossegue sem ela.

**Racional**: as skills de início de tarefa informam decisões de estrutura caras de desfazer
depois de escritas; carregá-las depois do código já existir perde o ponto. `security-guidance`
antes de `code-review` evita que a revisão geral analise código que uma correção de segurança está
prestes a alterar.

### XIV. Todo Commit Passa pelo Gate de CI antes do Merge

Push em qualquer branch diferente de `main` dispara lint, checagem de tipos e testes (unitários e
de integração) contra o MiniStack. Se tudo passar e não existir PR aberto para aquele branch, um
PR para `main` é aberto automaticamente. O mesmo conjunto de testes roda novamente no PR como
status check obrigatório — a abertura automática do PR não substitui o gate, apenas remove o passo
manual de criá-lo. Nenhum código chega a `main` sem esse gate verde, e commit direto em `main` não
é uma prática deste projeto.

**Racional**: automatizar a abertura do PR remove fricção sem remover o gate — o teste que decide
se o código pode ser mesclado roda sempre no PR, nunca apenas no push. Isso mantém um único ponto
de verdade sobre "está verde", em vez de dois mecanismos que podem divergir.

## Contexto Arquitetural

Este projeto é uma plataforma assíncrona orientada a mensagens em AWS (API Gateway, SQS, ECS),
composta por dois microserviços em Python (MS Solicitação, MS Antifraude) que se comunicam
exclusivamente por filas SQS, cada uma com DLQ dedicada. O acesso é controlado por um Lambda
Authorizer no API Gateway; o estado da solicitação vive em uma tabela única do DynamoDB, de
propriedade exclusiva do MS Solicitação. A análise de risco é executada por um LLM externo à AWS
(Google Gemini). Este é, adicionalmente, um projeto de estudo: toda a infraestrutura "AWS" roda
localmente via MiniStack — não há implantação em conta AWS real. O contexto completo, incluindo
diagramas, contratos de mensagem, ADRs e requisitos não-funcionais, vive em `docs/`; este
documento estabelece os princípios que esse contexto deve respeitar, não repete o detalhe que já
está lá.

## Qualidade e Processo de Desenvolvimento

O fluxo de trabalho deste repositório é Spec-Driven Development via Spec Kit: nenhuma
implementação começa sem spec. Toda feature passa por
`/speckit.constitution → /speckit.specify → /speckit.clarify → /speckit.plan → /speckit.tasks →
/speckit.checklist → /speckit.analyze → /speckit.implement`, nessa ordem. As skills obrigatórias
por tipo de tarefa (Princípio XIII) e o gate de CI (Princípio XIV) aplicam-se a toda mudança de
código produzida sob esse fluxo, sem exceção por tamanho da mudança — a skill decide se há algo a
apontar, não o desenvolvedor antecipadamente.

## Governance

**Precedência**: esta constituição tem precedência sobre qualquer prática, convenção ou preferência
individual em conflito. Em caso de conflito entre este documento e `docs/`, o princípio aqui
prevalece sobre o detalhe lá — e o detalhe em `docs/` deve ser corrigido para refletir o princípio.

**Emendas**: qualquer alteração de um princípio existente, adição de novo princípio ou remoção de
princípio é uma emenda formal a este documento, feita via `/speckit.constitution`. Toda emenda:
1. Declara a mudança e o racional em um Sync Impact Report no topo do arquivo (versão anterior →
   nova, princípios afetados, templates que precisam de atualização).
2. Incrementa `CONSTITUTION_VERSION` segundo versionamento semântico: MAJOR para remoção ou
   redefinição incompatível de princípio; MINOR para novo princípio ou expansão material de
   orientação existente; PATCH para clarificação, correção de texto ou reformulação não-semântica.
3. Propaga a mudança para `plan-template.md`, `spec-template.md`, `tasks-template.md`,
   `checklist-template.md` e `docs/CLAUDE.md` quando aplicável, na mesma emenda — uma emenda que
   deixa esses artefatos desalinhados está incompleta.

**Exceções justificadas**: uma violação deliberada de um princípio (ex.: uma dependência adicional
que aumenta complexidade, um desvio momentâneo de uma regra de resiliência) só é aceitável quando
registrada explicitamente na seção "Complexity Tracking" do `plan.md` da feature correspondente,
com a necessidade concreta e a razão pela qual a alternativa mais simples e alinhada ao princípio
foi rejeitada. Uma exceção sem esse registro é tratada como violação, não como decisão. Exceções
recorrentes sobre o mesmo princípio são sinal de que o princípio precisa de emenda, não de mais
exceções.

**Conformidade obrigatória em planos**: todo `plan.md` gerado por `/speckit.plan` DEVE declarar
explicitamente, na seção "Constitution Check", sua conformidade com cada um dos catorze princípios
listados acima — incluindo os de processo (XIII, XIV) — antes do Phase 0 e novamente após o Phase
1. Um plano que não passa por essa checagem, ou que a passa com violações não registradas em
Complexity Tracking, não está pronto para `/speckit.tasks`.

**Revisão de conformidade**: `security-guidance` e `code-review` (Princípio XIII) são o mecanismo
de verificação de conformidade ao nível de código; a checagem de constituição em `plan.md` é o
mecanismo ao nível de desenho. Nenhum PR é mesclado com uma violação de princípio não-negociável
(I–XII) conhecida e não registrada como exceção justificada.

**Version**: 1.0.0 | **Ratified**: 2026-08-18 | **Last Amended**: 2026-08-18

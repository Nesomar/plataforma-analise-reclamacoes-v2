# Feature Specification: Avaliação de Risco de Fraude via Modelo de Linguagem

**Feature Branch**: `004-avaliacao-risco-llm`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Construa a capacidade de avaliar o risco de fraude de uma
solicitação usando um modelo de linguagem, produzindo uma recomendação estruturada e auditável.
Dado um conjunto de dados de solicitação, o sistema produz uma recomendação contendo: decisão
(aprovar, reprovar ou revisar), um score de risco entre 0 e 1, um nível de confiança entre 0 e 1,
e uma lista de motivos em linguagem natural que sustentam a decisão. A recomendação sempre
respeita um formato rígido e verificável. Uma resposta do modelo fora do formato é rejeitada, não
interpretada por aproximação. Se o modelo falhar, exceder o tempo limite ou insistir em resposta
inválida após uma tentativa corretiva, o sistema produz um resultado explicitamente indeterminado,
jamais uma reprovação. Dados fornecidos pelo cliente nunca alteram o comportamento da análise além
de servirem como informação avaliada — tentativas de manipular a avaliação por meio do conteúdo
dos campos não têm efeito sobre a decisão. Confiança abaixo de um limiar configurável resulta em
recomendação de revisão manual, mesmo que a decisão sugerida seja aprovar ou reprovar. Cada
avaliação registra qual versão do critério de avaliação foi usada, o tempo gasto e o custo,
permitindo comparar desempenho entre versões ao longo do tempo. Existe um conjunto de casos de
referência com resultado esperado, executável a cada mudança no critério de avaliação, que impede
regressões silenciosas de qualidade."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Produzir recomendação estruturada a partir dos dados da solicitação (Priority: P1)

Dado um conjunto de dados de uma solicitação, o sistema avalia o risco de fraude e produz uma
recomendação com decisão (aprovar, reprovar ou revisar), score de risco, nível de confiança e uma
lista de motivos em linguagem natural que sustentam a decisão.

**Why this priority**: é a razão de existir da feature — sem uma recomendação estruturada não há
nada para orquestrar nem para auditar depois.

**Independent Test**: fornecer um conjunto de dados de solicitação e confirmar que a recomendação
produzida contém os quatro elementos (decisão, score, confiança, motivos), com score e confiança
dentro do intervalo 0 a 1 e ao menos um motivo presente.

**Acceptance Scenarios**:

1. **Given** um conjunto de dados de solicitação, **When** o sistema o avalia, **Then** produz uma
   recomendação com decisão, score de risco entre 0 e 1, nível de confiança entre 0 e 1, e uma
   lista de motivos em linguagem natural.
2. **Given** uma recomendação de aprovar ou reprovar, **When** ela é produzida, **Then** vem
   acompanhada de ao menos um motivo que a sustenta.

---

### User Story 2 - Resposta fora do formato é rejeitada, não interpretada por aproximação (Priority: P1)

Quando a resposta do modelo não respeita o formato rígido esperado (campo ausente, tipo errado,
valor fora do intervalo permitido, motivo vazio numa decisão), o sistema rejeita a resposta
inteira, sem tentar extrair ou inferir dado por aproximação.

**Why this priority**: é a garantia de integridade que sustenta toda a confiança na recomendação —
sem ela, uma resposta malformada poderia virar uma decisão de negócio equivocada. Mesma prioridade
da User Story 1 porque as duas juntas definem o contrato de saída da avaliação.

**Independent Test**: simular respostas do modelo com campo ausente, tipo incorreto, valor fora do
intervalo 0-1, e motivo vazio; confirmar que cada uma é integralmente rejeitada, sem gerar
recomendação parcial ou aproximada.

**Acceptance Scenarios**:

1. **Given** uma resposta do modelo com um campo obrigatório ausente ou de tipo incorreto, **When**
   o sistema a recebe, **Then** rejeita a resposta inteira, sem produzir recomendação a partir
   dela.
2. **Given** uma resposta com score ou confiança fora do intervalo 0 a 1, **When** o sistema a
   recebe, **Then** rejeita a resposta inteira.
3. **Given** uma resposta de aprovar ou reprovar sem nenhum motivo, **When** o sistema a recebe,
   **Then** rejeita a resposta inteira.

---

### User Story 3 - Falha, timeout ou resposta inválida persistente resulta em indeterminado (Priority: P2)

Se o modelo falhar, exceder o tempo limite configurado, ou continuar respondendo fora do formato
mesmo após uma tentativa corretiva, o sistema produz um resultado explicitamente indeterminado —
nunca uma reprovação.

**Why this priority**: é a postura de segurança que impede que uma falha técnica do avaliador vire
uma penalidade indevida ao cliente — depende da User Story 2 já existir (é preciso primeiro
detectar formato inválido para então decidir o que fazer com a persistência do erro).

**Independent Test**: simular indisponibilidade do modelo, timeout, e resposta inválida repetida
após a tentativa corretiva; confirmar que os três cenários produzem resultado indeterminado, nunca
reprovação.

**Acceptance Scenarios**:

1. **Given** o modelo indisponível ou retornando erro, **When** o sistema tenta avaliar, **Then**
   produz um resultado explicitamente indeterminado.
2. **Given** o tempo limite configurado excedido sem resposta do modelo, **When** o prazo expira,
   **Then** o sistema produz um resultado explicitamente indeterminado.
3. **Given** uma resposta inválida na primeira tentativa, **When** a tentativa corretiva também
   resulta em resposta inválida, **Then** o sistema produz um resultado explicitamente
   indeterminado, sem realizar uma terceira tentativa.
4. **Given** qualquer um dos três cenários acima, **When** o sistema decide o resultado, **Then**
   em nenhuma circunstância o resultado é uma reprovação.

---

### User Story 4 - Conteúdo do cliente não influencia o comportamento da avaliação (Priority: P2)

Dados fornecidos pelo cliente nos campos da solicitação servem exclusivamente como informação a
ser avaliada. Tentativas de manipular a avaliação por meio do conteúdo desses campos — por
exemplo, texto que tenta instruir o avaliador a decidir de determinada forma — não têm efeito
sobre a decisão.

**Why this priority**: protege a integridade da avaliação contra manipulação deliberada —
essencial para a confiabilidade da recomendação, no mesmo nível das demais garantias de
integridade do resultado.

**Independent Test**: submeter um conjunto de dados de solicitação em que um campo de texto
contém uma tentativa explícita de instruir o avaliador (ex.: pedindo aprovação incondicional) e
confirmar que a decisão final não é determinada por essa instrução, apenas pelo conteúdo avaliado
como informação de risco.

**Acceptance Scenarios**:

1. **Given** um campo da solicitação contendo uma tentativa de instrução dirigida ao avaliador,
   **When** o sistema avalia, **Then** essa tentativa não altera o comportamento da avaliação além
   de ser considerada como conteúdo avaliado.
2. **Given** duas solicitações idênticas exceto por uma tentativa de manipulação inserida em um
   campo de uma delas, **When** ambas são avaliadas, **Then** a presença da tentativa de
   manipulação, por si só, não determina a decisão.

---

### User Story 5 - Confiança abaixo do limiar força revisão manual (Priority: P2)

Quando o nível de confiança da recomendação está abaixo de um limiar configurável, a decisão final
é sempre "revisar", mesmo que a decisão originalmente sugerida pela avaliação fosse aprovar ou
reprovar.

**Why this priority**: é uma garantia conservadora que evita decisões automáticas de baixa
confiança — depende da User Story 1 já existir (é preciso primeiro ter confiança calculada), mas
tem prioridade equivalente às demais garantias de integridade da decisão.

**Independent Test**: produzir avaliações com confiança abaixo e acima do limiar configurado, para
decisões originalmente sugeridas de aprovar e de reprovar; confirmar que toda avaliação abaixo do
limiar resulta em decisão final "revisar".

**Acceptance Scenarios**:

1. **Given** uma avaliação com decisão sugerida "aprovar" e confiança abaixo do limiar
   configurado, **When** o resultado final é determinado, **Then** a decisão final é "revisar".
2. **Given** uma avaliação com decisão sugerida "reprovar" e confiança abaixo do limiar
   configurado, **When** o resultado final é determinado, **Then** a decisão final é "revisar".
3. **Given** uma avaliação com confiança igual ou acima do limiar configurado, **When** o
   resultado final é determinado, **Then** a decisão sugerida é mantida sem alteração.

---

### User Story 6 - Cada avaliação registra versão, tempo e custo (Priority: P3)

Toda avaliação realizada registra qual versão do critério de avaliação foi usada, o tempo gasto e
o custo incorrido, permitindo comparar desempenho entre versões ao longo do tempo.

**Why this priority**: é uma capacidade de observabilidade e melhoria contínua — valiosa desde o
início, mas não bloqueia a produção da recomendação em si.

**Independent Test**: realizar avaliações usando diferentes versões do critério de avaliação e
confirmar que cada uma registra a versão usada, o tempo gasto e o custo, permitindo comparar
métricas entre as versões.

**Acceptance Scenarios**:

1. **Given** uma avaliação concluída (com recomendação ou indeterminada), **When** ela é
   registrada, **Then** o registro inclui a versão do critério de avaliação usada, o tempo gasto e
   o custo incorrido.
2. **Given** avaliações realizadas com versões diferentes do critério, **When** um operador compara
   seus registros, **Then** consegue comparar desempenho (tempo, custo, resultado) entre as
   versões.

---

### User Story 7 - Casos de referência impedem regressão silenciosa de qualidade (Priority: P3)

Existe um conjunto de casos de referência, cada um com resultado esperado, que pode ser executado
a cada mudança no critério de avaliação para detectar se a mudança degradou a qualidade das
recomendações produzidas.

**Why this priority**: é uma proteção de qualidade de longo prazo — importante para evolução segu
ra do critério de avaliação, mas independente do funcionamento do fluxo de avaliação em produção.

**Independent Test**: executar o conjunto de casos de referência contra a versão atual do critério
de avaliação e confirmar que todos os casos produzem o resultado esperado; alterar
deliberadamente o critério de forma a divergir de um caso conhecido e confirmar que a execução do
conjunto de referência aponta a divergência.

**Acceptance Scenarios**:

1. **Given** o conjunto de casos de referência e a versão atual do critério de avaliação, **When**
   o conjunto é executado, **Then** cada caso é comparado ao seu resultado esperado.
2. **Given** uma mudança no critério de avaliação que altera o resultado de um caso de referência
   existente, **When** o conjunto é executado após a mudança, **Then** a divergência é identificada
   antes da mudança ser considerada válida para uso.

---

### Edge Cases

- O que acontece quando a resposta do modelo é estruturalmente válida mas contém valores fora do
  intervalo permitido para score ou confiança? É tratada como resposta fora do formato — mesma
  regra da User Story 2.
- O que acontece quando a confiança está exatamente igual ao limiar configurado, não abaixo dele?
  A decisão original é mantida — o limiar se aplica estritamente a valores abaixo dele.
- O que acontece se a tentativa corretiva falhar por um motivo diferente do erro original (ex.:
  timeout na primeira tentativa, formato inválido na segunda)? Ainda assim resulta em
  indeterminado — o sistema não realiza uma terceira tentativa, independentemente do tipo de falha
  em cada uma das duas primeiras.
- Como o sistema trata um caso de referência cujo resultado esperado é, ele mesmo,
  "indeterminado"? É um resultado esperado válido como qualquer outro — a execução do conjunto de
  referência compara o resultado obtido ao esperado, incluindo esse caso.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Dado um conjunto de dados de solicitação, o sistema MUST produzir uma recomendação
  contendo decisão (aprovar, reprovar ou revisar), score de risco entre 0 e 1, nível de confiança
  entre 0 e 1, e uma lista de motivos em linguagem natural.
- **FR-002**: Toda recomendação de aprovar ou reprovar MUST vir acompanhada de ao menos um motivo
  que a sustente.
- **FR-003**: O sistema MUST validar toda resposta recebida contra um formato rígido antes de
  qualquer uso; uma resposta fora do formato (campo ausente, tipo incorreto, valor fora do
  intervalo permitido, motivo ausente quando exigido) MUST ser rejeitada integralmente, sem
  interpretação ou extração por aproximação.
- **FR-004**: Quando a resposta inicial for rejeitada por formato inválido, o sistema MUST realizar
  exatamente uma tentativa corretiva antes de desistir.
- **FR-005**: Se o modelo falhar, exceder o tempo limite configurado, ou a resposta continuar
  inválida após a tentativa corretiva, o sistema MUST produzir um resultado explicitamente
  indeterminado. O sistema MUST NOT, em nenhuma dessas circunstâncias, produzir uma reprovação.
- **FR-006**: O conteúdo fornecido pelo cliente nos dados da solicitação MUST ser tratado
  exclusivamente como informação avaliada; MUST NOT ter efeito sobre o comportamento da avaliação
  além desse papel — tentativas de manipulação por meio do conteúdo dos campos MUST NOT alterar a
  decisão por si só.
- **FR-007**: Quando o nível de confiança de uma recomendação estiver abaixo de um limiar
  configurável, o sistema MUST substituir a decisão final por "revisar", independentemente da
  decisão originalmente sugerida.
- **FR-008**: Toda avaliação realizada MUST registrar a versão do critério de avaliação usada, o
  tempo gasto e o custo incorrido.
- **FR-009**: O sistema MUST manter um conjunto de casos de referência, cada um com um resultado
  esperado, executável a cada mudança no critério de avaliação.
- **FR-010**: A execução do conjunto de casos de referência MUST identificar qualquer caso cujo
  resultado obtido divirja do resultado esperado, permitindo detectar regressão de qualidade antes
  que uma mudança no critério de avaliação seja considerada válida para uso.

### Key Entities

- **Dados da Solicitação a Avaliar**: conjunto de dados de entrada para a avaliação, já preparado
  e minimizado por processo anterior a esta feature.
- **Recomendação**: saída estruturada de uma avaliação bem-sucedida — decisão (aprovar, reprovar
  ou revisar), score de risco, nível de confiança, lista de motivos.
- **Avaliação**: registro de uma execução da capacidade de avaliação — contém a recomendação
  produzida (ou a marcação de indeterminado), a versão do critério de avaliação usada, o tempo
  gasto e o custo incorrido.
- **Critério de Avaliação**: versão da lógica usada para produzir recomendações a partir dos dados
  da solicitação; comparável ao longo do tempo por meio dos registros de avaliação.
- **Caso de Referência**: par de entrada e resultado esperado usado para detectar regressão de
  qualidade a cada mudança no critério de avaliação.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% das recomendações produzidas respeitam o formato definido (decisão, score,
  confiança, motivos), verificado em teste.
- **SC-002**: 100% das respostas fora do formato, em um conjunto de teste com respostas
  propositalmente malformadas, são rejeitadas integralmente, sem gerar recomendação aproximada.
- **SC-003**: 100% dos cenários de falha de modelo, timeout ou resposta inválida persistente
  resultam em indeterminado, e 0% resultam em reprovação, verificado em teste de falha induzida.
- **SC-004**: em um conjunto de casos de teste adversariais contendo tentativas de manipulação via
  conteúdo dos campos, 100% das decisões produzidas não são determinadas pela presença da
  tentativa de manipulação em si.
- **SC-005**: 100% das avaliações com confiança abaixo do limiar configurado resultam em decisão
  final "revisar", independentemente da decisão originalmente sugerida.
- **SC-006**: 100% das avaliações realizadas em um período de teste têm versão do critério, tempo
  gasto e custo registrados e reconstruíveis.
- **SC-007**: a suíte de casos de referência, executada a cada mudança no critério de avaliação em
  teste, detecta 100% das divergências introduzidas deliberadamente nos casos existentes.

## Assumptions

- Esta feature recebe como entrada o pacote de dados já minimizado/anonimizado preparado pela
  feature 003-orquestracao-analise-risco; a minimização de dados pessoais em si não é
  responsabilidade desta feature.
- O modelo de linguagem é tratado como um serviço externo de avaliação; sua implementação,
  hospedagem ou provedor específico está fora do escopo desta especificação de negócio.
- O limiar de confiança abaixo do qual a decisão é sempre revertida para "revisar" é configurável
  por ambiente, sem valor fixo exigido por esta especificação.
- O tempo limite (timeout) para aguardar resposta do modelo é configurável, consistente com o
  prazo limite tratado pela orquestração do ciclo de vida (003-orquestracao-analise-risco).
- Custo de uma avaliação é expresso em uma unidade comparável ao longo do tempo (ex.: unidade de
  consumo do modelo), sem exigir moeda específica nesta especificação.
- O resultado "indeterminado" produzido por esta feature corresponde ao "resultado tecnicamente
  inconclusivo" tratado pela orquestração do ciclo de vida (003-orquestracao-analise-risco), que o
  encaminha para análise manual.

## Fora de Escopo

- Como o modelo de linguagem é hospedado, treinado ou selecionado (infraestrutura e escolha de
  provedor).
- O processo editorial de criação, atualização e aprovação de novas versões do critério de
  avaliação — apenas o registro da versão usada e a suíte de regressão contra casos de referência
  estão cobertos aqui.
- A orquestração do ciclo de vida da solicitação em torno do resultado desta avaliação (transições
  de estado aprovada/reprovada/análise manual) — coberta por 003-orquestracao-analise-risco.
- A interface de trabalho do analista humano.

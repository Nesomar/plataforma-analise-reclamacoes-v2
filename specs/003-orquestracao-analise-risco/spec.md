# Feature Specification: Orquestração do Ciclo de Vida da Solicitação em torno da Análise de Risco

**Feature Branch**: `003-orquestracao-analise-risco`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Construa a orquestração do ciclo de vida da solicitação em torno da
análise de risco. Toda solicitação recebida deve ser encaminhada para análise antifraude, e o
resultado dessa análise deve atualizar o estado da solicitação de forma consistente e auditável.
Ao receber uma solicitação, o sistema prepara um conjunto mínimo de dados relevantes para a
análise de risco e a encaminha, marcando a solicitação como "em análise". Os dados encaminhados
para análise contêm apenas o necessário para avaliar risco; dados pessoais que não influenciam a
avaliação são omitidos ou anonimizados. Quando o resultado da análise retorna, a solicitação passa
para aprovada, reprovada ou encaminhada para análise manual, conforme a recomendação recebida. Um
resultado que chegue para uma solicitação já concluída é ignorado sem erro e sem alterar o estado
— o histórico registra a ocorrência. Um resultado técnico inconclusivo encaminha a solicitação
para análise manual; nunca para reprovação. Se o resultado não retornar dentro de um prazo limite
configurável, a solicitação é encaminhada para análise manual automaticamente. Todo resultado é
armazenado junto à solicitação com sua justificativa, permitindo a um auditor entender depois por
que aquela decisão foi tomada. Estados finais são definitivos: uma solicitação aprovada ou
reprovada nunca volta a estados anteriores por processamento automático. Fora do escopo: como a
análise de risco é executada internamente e a interface de trabalho do analista humano."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Encaminhamento para análise com dados mínimos (Priority: P1)

Assim que uma solicitação é recebida, o sistema monta um conjunto mínimo de dados relevantes para
avaliação de risco — sem dados pessoais que não influenciam a avaliação — e a encaminha para
análise, marcando a solicitação como "em análise".

**Why this priority**: é o gatilho de todo o ciclo de vida orquestrado por esta feature — sem ele
nenhuma solicitação jamais recebe uma decisão.

**Independent Test**: submeter uma solicitação recém-recebida e confirmar que ela passa para o
estado "em análise" e que o pacote de dados encaminhado contém apenas os campos relevantes para
risco, sem dados pessoais dispensáveis.

**Acceptance Scenarios**:

1. **Given** uma solicitação recém-recebida, **When** o sistema a processa, **Then** ele monta um
   pacote de dados apenas com o necessário para avaliar risco e encaminha para análise.
2. **Given** o pacote de dados encaminhado para análise, **When** ele é inspecionado, **Then**
   nenhum dado pessoal que não influencia a avaliação de risco está presente, seja omitido ou
   anonimizado.
3. **Given** uma solicitação encaminhada para análise, **When** o encaminhamento é concluído,
   **Then** o estado da solicitação passa a ser "em análise".

---

### User Story 2 - Estado atualizado conforme a recomendação da análise (Priority: P1)

Quando o resultado da análise de risco retorna com uma recomendação, a solicitação transiciona
para o estado correspondente: aprovada, reprovada, ou encaminhada para análise manual.

**Why this priority**: é o motivo de existir da orquestração — fechar o ciclo entre "em análise" e
uma decisão. Sem isso a solicitação fica presa indefinidamente.

**Independent Test**: para uma solicitação "em análise", simular a chegada de um resultado com
cada uma das três recomendações possíveis e confirmar que o estado final corresponde
exatamente à recomendação recebida.

**Acceptance Scenarios**:

1. **Given** uma solicitação "em análise", **When** chega um resultado com recomendação de
   aprovação, **Then** a solicitação transiciona para "aprovada".
2. **Given** uma solicitação "em análise", **When** chega um resultado com recomendação de
   reprovação, **Then** a solicitação transiciona para "reprovada".
3. **Given** uma solicitação "em análise", **When** chega um resultado com recomendação de análise
   manual, **Then** a solicitação transiciona para "análise manual".

---

### User Story 3 - Timeout aciona análise manual automaticamente (Priority: P2)

Se o resultado da análise não retornar dentro de um prazo limite configurável, a solicitação é
automaticamente encaminhada para análise manual, sem depender de intervenção.

**Why this priority**: garante que nenhuma solicitação fique presa indefinidamente em "em análise"
por indisponibilidade ou lentidão do motor de análise — proteção de continuidade, um degrau abaixo
do caminho feliz porque só se manifesta na ausência de resposta.

**Independent Test**: submeter uma solicitação para análise e não devolver resultado algum;
confirmar que, ao expirar o prazo configurado, a solicitação é movida automaticamente para
"análise manual".

**Acceptance Scenarios**:

1. **Given** uma solicitação "em análise" cujo prazo limite configurado se esgota sem resultado,
   **When** o prazo expira, **Then** o sistema move a solicitação automaticamente para "análise
   manual".
2. **Given** uma solicitação movida para "análise manual" por timeout, **When** o resultado
   original (atrasado) chega depois, **Then** ele é tratado como resultado tardio (ver User Story
   5), sem reabrir "em análise".

---

### User Story 4 - Resultado inconclusivo nunca leva à reprovação (Priority: P2)

Quando o resultado da análise é tecnicamente inconclusivo (falha na avaliação, não uma
recomendação válida), a solicitação é encaminhada para análise manual — nunca para reprovação.

**Why this priority**: é uma garantia de postura conservadora e justa com o cliente — uma falha
técnica do motor de análise não pode ser tratada como evidência de risco. Mesmo nível dos demais
comportamentos de exceção do ciclo de vida.

**Independent Test**: simular um resultado tecnicamente inconclusivo para uma solicitação "em
análise" e confirmar que ela é movida para "análise manual", nunca para "reprovada".

**Acceptance Scenarios**:

1. **Given** uma solicitação "em análise", **When** o resultado retorna tecnicamente inconclusivo,
   **Then** a solicitação transiciona para "análise manual".
2. **Given** um resultado tecnicamente inconclusivo, **When** o sistema decide o próximo estado,
   **Then** em nenhuma circunstância a solicitação transiciona para "reprovada" nesse caminho.

---

### User Story 5 - Resultado tardio para solicitação já concluída é ignorado (Priority: P2)

Um resultado chega para uma solicitação que já está em estado final ou já foi encaminhada para
análise manual. O sistema ignora esse resultado sem alterar o estado atual e sem gerar erro, mas
registra a ocorrência no histórico da solicitação.

**Why this priority**: garante a consistência do ciclo de vida diante de reentregas, resultados
duplicados ou fora de ordem — comum em sistemas assíncronos com múltiplas tentativas.

**Independent Test**: levar uma solicitação a um estado concluído (aprovada, reprovada, ou análise
manual) e então simular a chegada de um segundo resultado; confirmar que o estado não muda, que
nenhum erro é retornado ao remetente, e que o histórico registra a ocorrência do resultado tardio.

**Acceptance Scenarios**:

1. **Given** uma solicitação já aprovada, **When** um novo resultado chega para ela, **Then** o
   sistema o ignora, mantém o estado "aprovada", não gera erro, e registra a ocorrência no
   histórico.
2. **Given** uma solicitação já reprovada, **When** um novo resultado chega para ela, **Then** o
   sistema o ignora, mantém o estado "reprovada", não gera erro, e registra a ocorrência no
   histórico.
3. **Given** uma solicitação já encaminhada para análise manual, **When** um novo resultado
   automático chega para ela, **Then** o sistema o ignora, mantém o estado "análise manual", não
   gera erro, e registra a ocorrência no histórico.

---

### User Story 6 - Toda decisão fica auditável com justificativa (Priority: P3)

Cada resultado recebido é armazenado junto à solicitação com a justificativa que o acompanha,
permitindo que um auditor reconstrua, depois, por que aquela decisão foi tomada.

**Why this priority**: é uma capacidade de conformidade e confiança valiosa desde o primeiro dia,
mas não bloqueia o funcionamento das transições de estado em si — pode ser verificada
independentemente delas.

**Independent Test**: levar uma solicitação a cada um dos três estados de decisão e confirmar que,
para cada uma, o registro armazenado inclui a justificativa recebida junto ao resultado.

**Acceptance Scenarios**:

1. **Given** um resultado de análise recebido para qualquer solicitação, **When** ele é
   processado, **Then** o resultado e sua justificativa são armazenados junto à solicitação.
2. **Given** uma solicitação em qualquer estado de decisão (aprovada, reprovada, análise manual),
   **When** um auditor consulta seu histórico, **Then** consegue reconstruir a justificativa da
   decisão tomada.

---

### Edge Cases

- O que acontece quando o resultado da análise chega com uma recomendação fora do conjunto
  conhecido (nem aprovação, nem reprovação, nem análise manual)? É tratado como resultado
  tecnicamente inconclusivo, seguindo a mesma regra da User Story 4 — encaminhado para análise
  manual, nunca para reprovação.
- Como o sistema resolve a corrida entre o prazo limite expirando e o resultado chegando quase ao
  mesmo tempo? Apenas uma transição prevalece de forma determinística; se o timeout já moveu a
  solicitação para "análise manual", o resultado que chega em seguida é tratado como tardio (User
  Story 5).
- O que acontece se um resultado chegar referenciando uma solicitação que o sistema não reconhece
  (nunca foi encaminhada para análise)? É ignorado sem erro visível ao remetente e a ocorrência é
  registrada para investigação, da mesma forma que um resultado tardio.
- Uma solicitação pode ficar "em análise" por mais de um ciclo de encaminhamento? Não — cada
  solicitação é encaminhada para análise uma única vez por esta orquestração; reencaminhamentos
  (retry) são responsabilidade da camada de entrega de mensagens, não desta lógica de estado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Ao receber uma solicitação, o sistema MUST montar um conjunto mínimo de dados
  relevantes para avaliação de risco e encaminhá-lo para análise.
- **FR-002**: O pacote de dados encaminhado para análise MUST conter apenas os dados necessários
  para avaliar risco; dados pessoais que não influenciam a avaliação MUST ser omitidos ou
  anonimizados antes do encaminhamento.
- **FR-003**: Ao encaminhar uma solicitação para análise, o sistema MUST marcar seu estado como
  "em análise".
- **FR-004**: Quando um resultado retornar com recomendação de aprovação para uma solicitação "em
  análise", o sistema MUST transicioná-la para "aprovada".
- **FR-005**: Quando um resultado retornar com recomendação de reprovação para uma solicitação "em
  análise", o sistema MUST transicioná-la para "reprovada".
- **FR-006**: Quando um resultado retornar com recomendação de análise manual para uma solicitação
  "em análise", o sistema MUST transicioná-la para "análise manual".
- **FR-007**: Quando um resultado for tecnicamente inconclusivo, ou trouxer uma recomendação fora
  do conjunto conhecido, o sistema MUST encaminhar a solicitação para "análise manual" e MUST NOT,
  em nenhuma circunstância, transicioná-la para "reprovada" nesse caminho.
- **FR-008**: Se o resultado da análise não retornar dentro de um prazo limite configurável, o
  sistema MUST encaminhar automaticamente a solicitação para "análise manual".
- **FR-009**: Um resultado que chegar para uma solicitação já em estado final ("aprovada" ou
  "reprovada") ou já em "análise manual" MUST ser ignorado, sem alterar o estado atual e sem gerar
  erro visível ao remetente do resultado.
- **FR-010**: Toda ocorrência de resultado ignorado por chegar tarde (FR-009) ou por referenciar
  solicitação desconhecida MUST ser registrada no histórico da solicitação correspondente (quando
  identificável).
- **FR-011**: Todo resultado de análise processado MUST ser armazenado junto à solicitação com sua
  justificativa, de forma que permita reconstrução posterior do motivo da decisão.
- **FR-012**: Os estados finais "aprovada" e "reprovada" MUST ser definitivos — nenhum
  processamento automático desta orquestração MUST transicionar uma solicitação desses estados
  para qualquer outro estado.
- **FR-013**: O sistema MUST encaminhar cada solicitação para análise no máximo uma vez por esta
  orquestração; reentrega de mensagem em nível de transporte não MUST resultar em novo
  encaminhamento lógico.

### Key Entities

- **Solicitação**: entidade cujo ciclo de vida esta feature orquestra. Estado transita por
  "recebida" → "em análise" → {"aprovada", "reprovada", "análise manual"}. "aprovada" e
  "reprovada" são finais e definitivos; "análise manual" é um encaminhamento cuja resolução final
  é responsabilidade de outra feature.
- **Pacote de Dados para Análise**: subconjunto minimizado e, quando necessário, anonimizado dos
  dados da solicitação, contendo apenas o que influencia a avaliação de risco.
- **Resultado de Análise**: recomendação recebida (aprovar, reprovar, análise manual, ou
  tecnicamente inconclusivo), acompanhada de justificativa, associada à solicitação que a
  originou.
- **Histórico da Solicitação**: registro cronológico auditável de eventos da solicitação,
  incluindo transições de estado, resultados armazenados com justificativa, e ocorrências de
  resultados ignorados por chegarem tarde ou fora de contexto.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% das solicitações recebidas são encaminhadas para análise e marcadas "em
  análise", verificado em teste de ponta a ponta.
- **SC-002**: 0% dos pacotes de dados encaminhados para análise contêm dado pessoal que não
  influencia avaliação de risco, verificado por amostragem auditável.
- **SC-003**: 100% dos resultados tecnicamente inconclusivos, em teste, resultam em "análise
  manual" e 0% resultam em "reprovada".
- **SC-004**: 100% das solicitações cujo prazo limite configurado expira sem resultado são movidas
  para "análise manual" dentro de uma janela curta após a expiração, sem intervenção manual.
- **SC-005**: 100% dos resultados que chegam para solicitação já concluída (ou já em análise
  manual) são ignorados sem alteração de estado, e 100% dessas ocorrências aparecem no histórico.
- **SC-006**: 100% das solicitações em estado de decisão (aprovada, reprovada, análise manual) têm
  a justificativa da decisão reconstruível a partir do histórico armazenado.

## Assumptions

- Esta feature depende da solicitação já existir e estar persistida (feature
  002-recepcao-propostas), incluindo seu identificador único e o identificador de correlação que
  acompanha os registros de histórico.
- "Solicitação já concluída", para efeito de ignorar resultado tardio, inclui os estados finais
  ("aprovada", "reprovada") e também "análise manual" — uma vez que a solicitação foi roteada para
  decisão humana, um resultado automático atrasado não deve sobrescrever esse roteamento.
- O mecanismo interno de execução da análise de risco (motor, modelo, chamadas externas) é externo
  a esta feature; aqui trata-se exclusivamente da orquestração de estado ao redor do resultado
  recebido.
- O prazo limite (timeout) para aguardar o resultado da análise é configurável por ambiente, sem
  valor fixo exigido por esta especificação.
- A transição de "análise manual" para um estado final (aprovada/reprovada) por decisão de um
  analista humano é responsabilidade de outra feature, fora do escopo desta especificação.

## Fora de Escopo

- Como a análise de risco é executada internamente (motor, modelo, regras, chamadas externas).
- A interface de trabalho do analista humano responsável pela análise manual.
- A resolução final (aprovação/reprovação) de uma solicitação em "análise manual" pela decisão
  humana em si — apenas o encaminhamento para esse estado está coberto aqui.
- A recepção inicial da proposta e a criação da solicitação (coberta por
  002-recepcao-propostas).

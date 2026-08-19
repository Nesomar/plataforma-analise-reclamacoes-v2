# Feature Specification: Acompanhamento de Status em Tempo Real

**Feature Branch**: `005-status-tempo-real`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Construa a capacidade de manter o cliente informado sobre o
andamento da sua solicitação sem que ele precise recarregar a página ou perguntar. Toda mudança de
status de uma solicitação chega à interface do cliente que está acompanhando aquela solicitação. O
cliente vê o status atual, e quando há decisão, vê o resultado com uma explicação compreensível —
sem jargão técnico e sem expor detalhes internos da avaliação de risco que possam ajudar alguém a
burlar o sistema. Se a conexão do cliente cair e voltar, ele recebe o estado atual correto, sem
perder atualizações ocorridas durante a queda. Um cliente só recebe atualizações das próprias
solicitações. A interface indica claramente quando a solicitação está aguardando análise, para que
a espera não pareça travamento."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Mudança de status chega sem recarregar a página (Priority: P1)

Um cliente está acompanhando sua solicitação na interface. Assim que o status dela muda, a
interface reflete a mudança automaticamente, sem que o cliente precise recarregar a página ou
perguntar.

**Why this priority**: é a razão de existir da feature — a promessa central de manter o cliente
informado sem esforço dele.

**Independent Test**: abrir a interface acompanhando uma solicitação, provocar uma mudança de
status no sistema, e confirmar que a interface reflete o novo status sem qualquer ação manual do
cliente.

**Acceptance Scenarios**:

1. **Given** um cliente acompanhando sua solicitação na interface, **When** o status da
   solicitação muda, **Then** a interface exibe o novo status automaticamente.
2. **Given** uma solicitação sem nenhuma mudança de status ainda, **When** o cliente abre a
   interface, **Then** ele vê o status atual correto desde o primeiro momento.

---

### User Story 2 - Resultado da decisão explicado sem jargão nem detalhe interno (Priority: P1)

Quando a solicitação recebe uma decisão, o cliente vê o resultado acompanhado de uma explicação em
linguagem compreensível — sem termos técnicos de avaliação de risco e sem qualquer detalhe interno
que possa ajudar alguém a burlar o sistema.

**Why this priority**: entrega o valor do "andamento" na sua forma mais importante — a decisão em
si — com a mesma prioridade da User Story 1 porque uma decisão exibida sem essa proteção seria uma
falha de segurança, não apenas uma funcionalidade incompleta.

**Independent Test**: levar uma solicitação a uma decisão (aprovada e reprovada, em testes
separados) e confirmar que a explicação exibida ao cliente é compreensível, sem jargão técnico, e
não contém score, motivo bruto do modelo ou qualquer outro detalhe interno da avaliação de risco.

**Acceptance Scenarios**:

1. **Given** uma solicitação aprovada, **When** o cliente vê o resultado, **Then** a explicação é
   compreensível e não expõe detalhe interno da avaliação de risco.
2. **Given** uma solicitação reprovada, **When** o cliente vê o resultado, **Then** a explicação é
   compreensível, não expõe detalhe interno da avaliação de risco, e não fornece informação que
   ajude a contornar o sistema em uma nova tentativa.

---

### User Story 3 - Reconexão recupera o estado atual correto (Priority: P2)

A conexão do cliente com a interface cai e depois volta. Ao reconectar, ele vê o estado atual e
correto da solicitação, sem que nenhuma atualização ocorrida durante a queda fique retida sem
aparecer.

**Why this priority**: garante a confiabilidade da promessa de acompanhamento em tempo real diante
de instabilidade de rede, comum no uso real — depende da User Story 1 já existir.

**Independent Test**: abrir a interface acompanhando uma solicitação, simular queda de conexão,
provocar uma ou mais mudanças de status durante a queda, restabelecer a conexão, e confirmar que a
interface exibe o estado atual correto, refletindo a mudança mais recente ocorrida durante a
queda.

**Acceptance Scenarios**:

1. **Given** a conexão do cliente caiu e o status da solicitação mudou durante a queda, **When** a
   conexão é restabelecida, **Then** a interface exibe o estado atual correto, sem informação
   desatualizada.
2. **Given** a solicitação chegou a uma decisão final enquanto o cliente estava desconectado,
   **When** a conexão é restabelecida, **Then** o cliente vê a decisão e sua explicação, sem
   depender de tê-la recebido no momento em que ocorreu.

---

### User Story 4 - Cliente só recebe atualizações das próprias solicitações (Priority: P2)

Um cliente jamais recebe atualização de status de uma solicitação que não é sua, mesmo que
conheça o protocolo dela.

**Why this priority**: é a garantia de isolamento entre clientes aplicada ao canal de
acompanhamento em tempo real — mesma prioridade das demais garantias de privacidade do sistema.

**Independent Test**: autenticar como cliente A e tentar acompanhar uma solicitação pertencente ao
cliente B (usando um protocolo real de B); confirmar que nenhuma atualização daquela solicitação é
entregue ao cliente A.

**Acceptance Scenarios**:

1. **Given** um cliente autenticado tentando acompanhar uma solicitação que não é sua, **When**
   o status dessa solicitação muda, **Then** nenhuma atualização é entregue a esse cliente.
2. **Given** um cliente autenticado acompanhando legitimamente sua própria solicitação, **When**
   outra solicitação de outro cliente muda de status, **Then** essa mudança não é entregue a ele.

---

### User Story 5 - Espera por análise é indicada claramente (Priority: P2)

Enquanto a solicitação ainda não recebeu decisão (está recebida ou em análise), a interface indica
claramente esse estado de espera, para que o cliente entenda que o sistema está processando, não
travado.

**Why this priority**: sustenta a confiança do cliente durante o período mais longo do ciclo de
vida da solicitação — sem essa indicação, a experiência de "esperar sem saber" mina o valor da
própria feature.

**Independent Test**: acompanhar uma solicitação enquanto ela está recebida e depois em análise, e
confirmar que a interface indica claramente, em ambos os momentos, que a solicitação está sendo
processada e aguardando resultado.

**Acceptance Scenarios**:

1. **Given** uma solicitação recém-recebida, ainda sem análise iniciada, **When** o cliente a
   acompanha, **Then** a interface indica claramente que ela está aguardando análise.
2. **Given** uma solicitação em análise, **When** o cliente a acompanha, **Then** a interface
   indica claramente esse estado, distinto de uma falha ou travamento.

---

### Edge Cases

- O que acontece quando a solicitação passa por múltiplas mudanças de status durante uma queda de
  conexão do cliente (ex.: em análise → análise manual)? Ao reconectar, o cliente vê o estado
  atual correto; não é garantido que ele veja cada transição intermediária individualmente, apenas
  que nenhuma decisão ou mudança de estado fica omitida do resultado final exibido.
- O que acontece quando um cliente abre a interface pela primeira vez depois que a solicitação já
  chegou a uma decisão final? Ele vê imediatamente o estado atual (a decisão e sua explicação),
  mesmo sem ter acompanhado a mudança em tempo real.
- Como a interface trata uma solicitação isolada para inspeção humana por falha técnica no
  encaminhamento (fora do fluxo normal de decisão)? É exibida como "aguardando análise" do ponto
  de vista do cliente, sem expor a natureza técnica da falha interna.
- O que acontece se a explicação de uma decisão reprovada, por conter informação sensível sobre o
  motivo, não puder ser simplificada com segurança? O sistema exibe uma explicação genérica que
  comunica a reprovação sem detalhar o motivo específico, priorizando a proteção contra burla sobre
  o detalhamento.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST entregar, à interface do cliente que está acompanhando uma
  solicitação, toda mudança de status dessa solicitação, sem exigir que o cliente recarregue a
  página ou consulte manualmente.
- **FR-002**: A interface MUST exibir o status atual da solicitação em qualquer momento em que o
  cliente a esteja acompanhando, incluindo no primeiro acesso.
- **FR-003**: Quando a solicitação tiver uma decisão (aprovada ou reprovada), o sistema MUST
  apresentar ao cliente o resultado acompanhado de uma explicação em linguagem compreensível, sem
  jargão técnico.
- **FR-004**: A explicação apresentada ao cliente MUST NOT expor detalhe interno da avaliação de
  risco (como score, nível de confiança, ou motivos brutos produzidos pela avaliação) que possa
  ajudar alguém a burlar o sistema.
- **FR-005**: Se a conexão do cliente cair e for restabelecida, o sistema MUST apresentar o estado
  atual e correto da solicitação, sem exibir informação desatualizada nem omitir uma decisão
  ocorrida durante a queda.
- **FR-006**: Um cliente MUST receber atualizações apenas de solicitações que lhe pertencem;
  MUST NUNCA receber atualização de solicitação de outro cliente, mesmo conhecendo seu protocolo.
- **FR-007**: Enquanto a solicitação estiver aguardando análise (recebida ou em análise), a
  interface MUST indicar claramente esse estado de espera, de forma distinguível de uma falha ou
  travamento.

### Key Entities

- **Assinatura de Acompanhamento**: vínculo entre um cliente autenticado e uma solicitação
  específica, usado para determinar quem deve receber quais atualizações.
- **Atualização de Status**: evento entregue à interface do cliente contendo o status atual da
  solicitação e, quando aplicável, a explicação do resultado.
- **Explicação do Resultado**: versão em linguagem compreensível da decisão tomada, adequada para
  exibição ao cliente, sem detalhe interno da avaliação de risco.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% das mudanças de status de uma solicitação chegam à interface do cliente que a
  acompanha sem exigir recarregar a página ou consultar manualmente, verificado em teste ponta a
  ponta.
- **SC-002**: 100% das decisões finais exibidas ao cliente vêm acompanhadas de explicação em
  linguagem compreensível, sem termo técnico de avaliação de risco, verificado por revisão de
  amostra.
- **SC-003**: 0% das explicações exibidas ao cliente contêm detalhe interno de avaliação de risco
  (score, motivo bruto do modelo, critério de detecção), verificado por auditoria de amostra.
- **SC-004**: 100% dos clientes que reconectam após uma queda de conexão veem o estado atual
  correto da solicitação, sem exibir informação desatualizada, verificado em teste de
  reconexão.
- **SC-005**: 0% das tentativas de um cliente receber atualização de solicitação que não é sua são
  bem-sucedidas, verificado em teste de isolamento.
- **SC-006**: 100% do tempo em que uma solicitação está "recebida" ou "em análise", a interface
  exibe indicação clara de espera, verificado em teste de interface.

## Assumptions

- O cliente já está autenticado ao acompanhar uma solicitação (feature 001-autenticacao); a
  permissão para acompanhá-la segue a mesma regra de posse já especificada ali — apenas o dono da
  solicitação recebe suas atualizações.
- "Sem perder atualizações ocorridas durante a queda" significa que, ao reconectar, o cliente
  sempre vê o estado atual e correto, incluindo qualquer decisão já tomada; não exige
  necessariamente reproduzir, uma a uma, cada transição de estado intermediária ocorrida durante a
  desconexão.
- A tradução da decisão para linguagem compreensível é uma camada de apresentação, separada da
  justificativa técnica armazenada para auditoria (003-orquestracao-analise-risco,
  004-avaliacao-risco-llm); esta feature consome o resultado já decidido, não o produz.
- Canais de notificação fora da interface que o cliente está usando para acompanhar (e-mail, SMS,
  push de aplicativo) não são exigidos por esta especificação.

## Fora de Escopo

- Como a avaliação de risco é executada ou como o resultado é decidido internamente (coberto por
  003-orquestracao-analise-risco e 004-avaliacao-risco-llm).
- A interface de trabalho do analista humano.
- Canais de notificação fora da interface de acompanhamento (e-mail, SMS, push de aplicativo).
- A recepção inicial da proposta e a criação da solicitação (coberta por
  002-recepcao-propostas).

# Feature Specification: Garantias Operacionais de Confiabilidade em Produção

**Feature Branch**: `006-confiabilidade-operacional`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Construa as garantias operacionais que tornam o sistema confiável em
produção. Mensagens que falham repetidamente no processamento são isoladas para inspeção humana em
vez de serem descartadas ou reprocessadas indefinidamente, e a operação é alertada quando isso
acontece. Um operador consegue reconstruir o histórico completo de qualquer solicitação a partir
de um único identificador, atravessando todos os componentes envolvidos. O time consegue
responder, a qualquer momento: quantas solicitações estão aguardando análise, qual a taxa de
aprovação, qual a taxa de resultados indeterminados, qual a latência e o custo médio por análise.
Uma degradação do provedor externo de análise é detectada automaticamente e não derruba a
capacidade do sistema de continuar recebendo solicitações. Existe um procedimento documentado e
testado para reprocessar mensagens isoladas após a correção de um defeito."

## Clarifications

### Session 2026-08-18

- Q: FR-002/SC-001 implicam um alerta por mensagem isolada; o Edge Case diz que o alerta reflete a condição, sem exigir alerta individual por mensagem. Qual granularidade vale? → A: Um alerta por evento de isolamento, deduplicado dentro de uma janela curta quando a mesma causa se repete.
- Q: SC-003 não define limite de atraso para as métricas operacionais ("refletindo o estado real"). Qual o limite aceitável? → A: Poucos minutos de atraso aceitável (até 5 min).
- Q: FR-009/SC-004 não definem prazo para detectar degradação do provedor externo. Qual a janela aceitável? → A: Janela curta de minutos (1–5 min), para evitar falso positivo por falha isolada.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Mensagens com falha repetida são isoladas e a operação é alertada (Priority: P1)

Uma mensagem falha repetidamente ao ser processada. Em vez de ser descartada ou de continuar sendo
reprocessada indefinidamente, ela é isolada para inspeção humana, e a operação recebe um alerta
informando que isso aconteceu.

**Why this priority**: é a garantia central que evita perda silenciosa de dado e reprocessamento
descontrolado — a base sobre a qual todas as demais garantias operacionais desta feature se
apoiam.

**Independent Test**: forçar a falha repetida do processamento de uma mensagem e confirmar que,
após as tentativas se esgotarem, ela é isolada e um alerta chega à operação.

**Acceptance Scenarios**:

1. **Given** uma mensagem que falha repetidamente no processamento, **When** as tentativas se
   esgotam, **Then** ela é isolada para inspeção humana, preservando contexto suficiente para
   diagnóstico.
2. **Given** uma mensagem recém-isolada, **When** o isolamento ocorre, **Then** a operação recebe
   um alerta informando a ocorrência.
3. **Given** uma mensagem isolada, **When** o tempo passa sem ação humana, **Then** o sistema não
   a reprocessa automaticamente nem a descarta.

---

### User Story 2 - Histórico completo de uma solicitação é reconstruível por um identificador (Priority: P1)

Um operador, de posse de um único identificador, reconstrói o histórico completo de qualquer
solicitação, incluindo os eventos ocorridos em todos os componentes pelos quais ela passou.

**Why this priority**: é a base de qualquer diagnóstico ou auditoria operacional — sem
rastreabilidade completa, as demais garantias (alerta, métricas, reprocessamento) não têm como ser
investigadas quando algo dá errado.

**Independent Test**: levar uma solicitação por múltiplos componentes do sistema e, ao final,
reconstruir seu histórico completo usando apenas seu identificador único, confirmando que todos os
eventos relevantes de todos os componentes aparecem, em ordem.

**Acceptance Scenarios**:

1. **Given** uma solicitação que passou por múltiplos componentes do sistema, **When** um operador
   consulta seu histórico usando o identificador único, **Then** consegue ver os eventos de todos
   os componentes envolvidos, em ordem cronológica.
2. **Given** uma solicitação que falhou em algum ponto do processamento antes de ser concluída,
   **When** um operador consulta seu histórico, **Then** consegue ver até onde ela chegou, sem
   erro nem lacuna silenciosa.

---

### User Story 3 - Métricas operacionais disponíveis a qualquer momento (Priority: P2)

O time consegue responder, a qualquer momento, quantas solicitações estão aguardando análise, qual
a taxa de aprovação, qual a taxa de resultados indeterminados, e qual a latência e o custo médio
por análise.

**Why this priority**: sustenta a operação contínua e a tomada de decisão do time sobre o
comportamento do sistema — depende dos eventos gerados pelos demais componentes já existirem, mas
não bloqueia o funcionamento deles.

**Independent Test**: gerar um volume conhecido de solicitações em diferentes estados e resultados,
e confirmar que as métricas consultadas (fila aguardando análise, taxa de aprovação, taxa de
indeterminado, latência média, custo médio) refletem corretamente esse volume.

**Acceptance Scenarios**:

1. **Given** solicitações aguardando análise no sistema, **When** o time consulta a métrica de
   quantidade aguardando análise, **Then** o número reportado corresponde à realidade atual.
2. **Given** um conjunto de solicitações já analisadas, **When** o time consulta taxa de aprovação
   e taxa de indeterminado, **Then** os valores reportados são consistentes com os resultados
   reais dessas solicitações.
3. **Given** um conjunto de análises concluídas, **When** o time consulta latência média e custo
   médio por análise, **Then** os valores reportados refletem as análises efetivamente realizadas.

---

### User Story 4 - Degradação do provedor externo não derruba a recepção de solicitações (Priority: P2)

O provedor externo de análise de risco degrada (fica lento ou indisponível). O sistema detecta essa
degradação automaticamente e continua aceitando novas solicitações normalmente, mesmo enquanto a
degradação persiste.

**Why this priority**: protege a disponibilidade do ponto de entrada do sistema diante de uma
dependência externa fora de seu controle — garantia de resiliência de mesmo nível que as demais
proteções operacionais desta feature.

**Independent Test**: simular degradação (lentidão ou indisponibilidade persistente) do provedor
externo de análise e confirmar que o sistema continua aceitando e confirmando novas solicitações
normalmente durante a simulação, e que a degradação é detectada sem intervenção manual.

**Acceptance Scenarios**:

1. **Given** o provedor externo de análise degradado, **When** uma nova solicitação chega, **Then**
   o sistema a aceita e confirma normalmente, independentemente da degradação.
2. **Given** o provedor externo de análise degradado, **When** a degradação persiste, **Then** o
   sistema a detecta automaticamente, sem depender de um operador identificá-la manualmente.
3. **Given** o provedor externo de análise que se recupera após uma degradação, **When** a
   recuperação ocorre, **Then** o sistema retoma o envio de solicitações para análise
   automaticamente, sem intervenção manual.

---

### User Story 5 - Procedimento testado para reprocessar mensagens isoladas (Priority: P2)

Existe um procedimento documentado que descreve como reprocessar mensagens isoladas depois que o
defeito que causou o isolamento foi corrigido, e esse procedimento foi validado por teste.

**Why this priority**: fecha o ciclo aberto pela User Story 1 — isolar sem um caminho de volta
apenas adia o problema. Depende do isolamento já existir, mas é o que devolve valor ao trabalho de
correção do defeito.

**Independent Test**: isolar deliberadamente um lote de mensagens, simular a correção do defeito
que causou o isolamento, executar o procedimento documentado, e confirmar que as mensagens voltam
a ser processadas corretamente.

**Acceptance Scenarios**:

1. **Given** um lote de mensagens isoladas e o defeito que as causou já corrigido, **When** o
   procedimento documentado é executado, **Then** as mensagens voltam a ser processadas
   corretamente.
2. **Given** o procedimento de reprocessamento, **When** ele é executado, **Then** requer uma ação
   humana deliberada — não ocorre de forma automática ou agendada.

---

### Edge Cases

- O que acontece quando várias mensagens são isoladas pela mesma causa raiz em sequência (ex.:
  mesma falha do provedor externo)? Os alertas dessas mensagens são deduplicados dentro da janela
  curta e configurável (ver FR-002), gerando um único alerta para a janela em vez de um alerta por
  mensagem.
- O que acontece se um operador tentar reconstruir o histórico de um identificador que nunca
  existiu no sistema? A consulta indica claramente que nenhum histórico foi encontrado, sem erro
  técnico expondo detalhe interno.
- Como o sistema trata uma tentativa de reprocessar mensagens isoladas antes da causa raiz ter sido
  de fato corrigida? O procedimento não impede tecnicamente a tentativa, mas por ser sempre um ato
  humano deliberado, a responsabilidade de confirmar a correção prévia é de quem o executa — o
  procedimento documentado inclui essa verificação como um passo explícito.
- O que acontece com as métricas operacionais durante uma janela em que nenhuma solicitação nova
  chega? Os valores refletem o estado real (ex.: zero solicitações aguardando), sem erro nem valor
  indefinido.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST isolar, em vez de descartar ou reprocessar indefinidamente, uma
  mensagem que falha repetidamente no processamento após um número configurável de tentativas.
- **FR-002**: O sistema MUST alertar a operação a cada evento de isolamento; quando múltiplas
  mensagens forem isoladas pela mesma causa dentro de uma janela curta e configurável, o sistema
  MUST deduplicar esses alertas em um único alerta para essa janela, em vez de gerar um alerta por
  mensagem.
- **FR-003**: Um operador MUST conseguir reconstruir o histórico completo de qualquer solicitação a
  partir de um único identificador, incluindo os eventos de todos os componentes pelos quais ela
  passou.
- **FR-004**: O sistema MUST expor, a qualquer momento, a quantidade de solicitações aguardando
  análise.
- **FR-005**: O sistema MUST expor, a qualquer momento, a taxa de aprovação das solicitações já
  analisadas.
- **FR-006**: O sistema MUST expor, a qualquer momento, a taxa de resultados indeterminados das
  solicitações já analisadas.
- **FR-007**: O sistema MUST expor, a qualquer momento, a latência média das análises realizadas.
- **FR-008**: O sistema MUST expor, a qualquer momento, o custo médio por análise realizada.
- **FR-009**: O sistema MUST detectar automaticamente uma degradação do provedor externo de
  análise dentro de uma janela curta de minutos (1 a 5 min) desde o início da degradação, sem
  depender de identificação manual, evitando falso positivo por uma única falha isolada.
- **FR-010**: Uma degradação detectada do provedor externo de análise MUST NOT impedir o sistema de
  continuar aceitando e confirmando novas solicitações.
- **FR-011**: Quando a degradação do provedor externo de análise cessar, o sistema MUST retomar o
  envio de solicitações para análise automaticamente, sem intervenção manual.
- **FR-012**: MUST existir um procedimento documentado descrevendo como reprocessar mensagens
  isoladas após a correção do defeito que causou o isolamento.
- **FR-013**: O procedimento de reprocessamento MUST ser validado por teste, demonstrando que
  mensagens isoladas voltam a ser processadas corretamente quando o procedimento é executado.
- **FR-014**: O reprocessamento de mensagens isoladas MUST ser um ato deliberado; MUST NOT ocorrer
  de forma automática ou agendada.

### Key Entities

- **Mensagem Isolada**: mensagem retirada do fluxo automático de processamento após falhar
  repetidamente, preservada com contexto suficiente para diagnóstico e para reprocessamento
  posterior.
- **Alerta Operacional**: notificação à equipe de operação, gerada quando uma mensagem é isolada ou
  quando uma degradação do provedor externo de análise é detectada.
- **Histórico da Solicitação**: linha do tempo auditável de eventos de uma solicitação,
  reconstruível a partir do seu identificador único, cobrindo todos os componentes pelos quais ela
  passou.
- **Painel de Métricas Operacionais**: conjunto de indicadores consultáveis a qualquer momento —
  quantidade aguardando análise, taxa de aprovação, taxa de indeterminado, latência média, custo
  médio por análise.
- **Estado de Degradação do Provedor Externo**: condição, detectada automaticamente, que reflete se
  o provedor externo de análise está operando normalmente ou degradado.
- **Procedimento de Reprocessamento**: sequência documentada e testada de passos para retomar o
  processamento de mensagens isoladas após a correção de um defeito.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% das mensagens que esgotam as tentativas de processamento, em teste de falha
  induzida, são isoladas, e 100% dos eventos de isolamento (deduplicados por causa e janela)
  geram um alerta à operação.
- **SC-002**: 100% das solicitações testadas, tendo passado por múltiplos componentes, têm
  histórico completo reconstruível a partir de um único identificador.
- **SC-003**: as métricas de quantidade aguardando análise, taxa de aprovação, taxa de
  indeterminado, latência média e custo médio por análise estão disponíveis a qualquer consulta,
  com atraso máximo de 5 minutos em relação ao evento mais recente.
- **SC-004**: 100% dos cenários de degradação simulada do provedor externo de análise são
  detectados automaticamente dentro de 5 minutos do início da degradação, e 0% deles impedem a
  recepção de novas solicitações durante a degradação, em teste de falha induzida.
- **SC-005**: o procedimento de reprocessamento de mensagens isoladas, executado em teste após a
  simulação da correção de um defeito, restaura 100% das mensagens do lote isolado ao
  processamento normal.
- **SC-006**: 100% dos cenários de degradação simulada que se recuperam durante o teste resultam
  em retomada automática do envio para análise, sem intervenção manual.

## Assumptions

- Esta feature consolida, em uma garantia operacional observável por toda a plataforma, capacidades
  já parcialmente especificadas em outras features: o isolamento para inspeção humana e o
  identificador de correlação (003-orquestracao-analise-risco), e o registro de tempo e custo por
  avaliação (004-avaliacao-risco-llm).
- "Todos os componentes envolvidos" no histórico de uma solicitação inclui, no mínimo, a recepção
  (002-recepcao-propostas), a orquestração de risco (003-orquestracao-analise-risco), a avaliação
  de risco (004-avaliacao-risco-llm) e o acompanhamento de status (005-status-tempo-real).
- O número de tentativas antes do isolamento e a janela de deduplicação de alertas por causa
  comum são configuráveis, sem valores fixos exigidos por esta especificação. O critério exato de
  sensibilidade para detecção de degradação (ex.: quantas falhas ou qual latência caracterizam
  degradação) também é configurável, respeitando o limite máximo de 5 minutos para detecção
  definido em FR-009/SC-004.
- "Alertar a operação" significa que a equipe recebe uma notificação por um canal operacional já
  estabelecido pela organização; o canal específico (e-mail, chat, acionamento de plantão) está
  fora do escopo desta especificação.

## Fora de Escopo

- Como cada componente individualmente processa ou decide — coberto pelas respectivas features
  (002-recepcao-propostas, 003-orquestracao-analise-risco, 004-avaliacao-risco-llm,
  005-status-tempo-real).
- A interface de trabalho do analista humano.
- O canal específico de entrega de alertas operacionais (e-mail, chat, acionamento de plantão).
- A escolha de ferramentas ou infraestrutura de monitoramento — esta especificação define o
  comportamento observável exigido, não como ele é implementado.

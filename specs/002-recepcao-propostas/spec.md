# Feature Specification: Recepção Assíncrona e Confiável de Propostas

**Feature Branch**: `002-recepcao-propostas`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Construa a capacidade de receber propostas de clientes de forma
assíncrona e confiável. Um cliente preenche um formulário com seus dados pessoais (documento,
nome, e-mail, data de nascimento) e os dados da proposta (valor, moeda, prazo em meses,
finalidade), junto com metadados de contexto capturados automaticamente (canal de origem, IP,
identificação do dispositivo). Ao enviar, o cliente recebe imediatamente um número de protocolo e
a confirmação de que a proposta foi recebida — sem esperar por nenhuma análise. A partir daí, ele
pode consultar o andamento da solicitação usando esse protocolo. O sistema aceita a proposta
apenas se os dados obrigatórios estiverem presentes e válidos; caso contrário, informa ao cliente
exatamente quais campos estão incorretos, sem criar solicitação alguma. Uma proposta aceita se
torna uma solicitação persistida, com identificador único, status inicial "recebida" e data de
criação. Nenhuma proposta aceita pode se perder, mesmo que o processamento posterior esteja
indisponível no momento. Se o mesmo envio chegar duas vezes por falha de rede ou clique duplo,
apenas uma solicitação é criada, e a segunda tentativa recebe o mesmo protocolo da primeira. O
cliente consegue consultar sua solicitação pelo protocolo e ver o status atual, a data de criação
e a data da última atualização. Consultar um protocolo inexistente devolve uma resposta clara de
não encontrado, sem vazar informação sobre outras solicitações. Um cliente nunca consegue
consultar a solicitação de outro cliente. Toda solicitação é rastreável do envio até a conclusão
por um identificador de correlação único que aparece em todos os registros do sistema. Uma
proposta que não consiga ser encaminhada para processamento após tentativas sucessivas é isolada
para inspeção humana, com contexto suficiente para diagnóstico, em vez de ser perdida ou
reprocessada indefinidamente."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Envio de proposta válida com confirmação imediata (Priority: P1)

Um cliente preenche o formulário com seus dados pessoais e os dados da proposta. Ao enviar, o
sistema aceita, persiste a solicitação e devolve imediatamente um número de protocolo, sem que o
cliente precise esperar qualquer análise.

**Why this priority**: é o caminho feliz — a razão de existir da feature. Sem ele não há proposta
para acompanhar, nem base para as demais garantias.

**Independent Test**: enviar um formulário com todos os dados obrigatórios válidos e confirmar
que a resposta chega com um protocolo, status "recebida" e sem exigir espera por processamento.

**Acceptance Scenarios**:

1. **Given** um cliente preenche todos os dados obrigatórios corretamente, **When** ele envia a
   proposta, **Then** o sistema persiste uma solicitação com identificador único, status
   "recebida" e data de criação, e devolve o protocolo imediatamente.
2. **Given** uma solicitação recém-criada, **When** o processamento posterior (análise) ainda não
   começou, **Then** a resposta ao cliente já contém a confirmação de recebimento, sem depender
   desse processamento.

---

### User Story 2 - Rejeição de proposta com dados obrigatórios ausentes ou inválidos (Priority: P1)

Um cliente envia o formulário com algum dado obrigatório ausente ou inválido (ex.: e-mail mal
formatado, valor negativo). O sistema recusa a proposta, informa exatamente quais campos estão
incorretos, e não cria solicitação alguma.

**Why this priority**: impede que dado inválido ou incompleto entre no sistema — proteção
essencial de qualidade de dado, na mesma prioridade do caminho feliz porque os dois juntos
definem o contrato de entrada da API.

**Independent Test**: enviar formulários com diferentes campos obrigatórios ausentes/inválidos, um
de cada vez, e confirmar que cada resposta aponta exatamente o(s) campo(s) problemático(s) e que
nenhuma solicitação é criada.

**Acceptance Scenarios**:

1. **Given** um formulário com um campo obrigatório ausente, **When** o cliente envia, **Then** o
   sistema rejeita e informa exatamente qual campo está ausente, sem criar solicitação.
2. **Given** um formulário com um campo em formato inválido (ex.: data de nascimento
   incompatível, moeda desconhecida, prazo não numérico), **When** o cliente envia, **Then** o
   sistema rejeita e informa exatamente qual campo é inválido, sem criar solicitação.
3. **Given** um formulário com múltiplos campos incorretos, **When** o cliente envia, **Then** a
   resposta lista todos os campos incorretos, não apenas o primeiro encontrado.

---

### User Story 3 - Reenvio duplicado não cria solicitação duplicada (Priority: P2)

O mesmo envio chega ao sistema duas vezes, por falha de rede ou clique duplo do cliente. Apenas
uma solicitação é criada; a segunda tentativa recebe o mesmo protocolo da primeira, como se
tivesse sido bem-sucedida.

**Why this priority**: garante a confiabilidade prometida pela feature — sem ela, uma proposta
poderia virar duas solicitações por uma falha de rede, quebrando a promessa de "cada proposta,
uma solicitação".

**Independent Test**: enviar o mesmo formulário duas vezes em sequência rápida e confirmar que
apenas uma solicitação existe no sistema e que ambas as respostas trazem o mesmo protocolo.

**Acceptance Scenarios**:

1. **Given** uma proposta já aceita e persistida, **When** o mesmo envio chega novamente, **Then**
   o sistema não cria uma segunda solicitação e devolve o protocolo da primeira.
2. **Given** dois envios idênticos quase simultâneos (condição de corrida), **When** ambos chegam
   ao sistema, **Then** apenas uma solicitação é persistida e ambas as respostas trazem o mesmo
   protocolo.

---

### User Story 4 - Consulta de andamento pelo protocolo (Priority: P2)

O cliente, de posse do protocolo recebido no envio, consulta sua solicitação e vê o status atual,
a data de criação e a data da última atualização.

**Why this priority**: entrega o valor de "acompanhar" prometido no envio assíncrono — depende da
User Story 1 já existir, mas é o motivo pelo qual o protocolo é útil.

**Independent Test**: criar uma solicitação, consultar pelo protocolo devolvido, e confirmar que a
resposta traz status, data de criação e data de última atualização coerentes com o registro
persistido.

**Acceptance Scenarios**:

1. **Given** uma solicitação existente pertencente ao cliente, **When** ele consulta pelo
   protocolo, **Then** o sistema devolve status atual, data de criação e data da última
   atualização.
2. **Given** uma solicitação cujo status foi atualizado após a criação, **When** o cliente
   consulta, **Then** a data de última atualização reflete a mudança mais recente.

---

### User Story 5 - Consulta negada para protocolo inexistente ou de outro cliente (Priority: P2)

Um cliente consulta um protocolo que não existe, ou que pertence a outro cliente. Em ambos os
casos o sistema devolve uma resposta de "não encontrado", sem revelar se o protocolo existe e
pertence a outra pessoa.

**Why this priority**: é a garantia de isolamento entre clientes — mesma prioridade da consulta
em si, porque uma consulta sem essa proteção é uma falha de privacidade, não uma funcionalidade
incompleta.

**Independent Test**: consultar um protocolo inventado (nunca emitido) e um protocolo real
pertencente a outro cliente; confirmar que as duas respostas são indistinguíveis entre si e não
revelam a existência da solicitação alheia.

**Acceptance Scenarios**:

1. **Given** um protocolo que nunca foi emitido, **When** qualquer cliente consulta, **Then** o
   sistema devolve resposta de "não encontrado".
2. **Given** um protocolo real pertencente a outro cliente, **When** um cliente diferente
   consulta, **Then** o sistema devolve a mesma resposta de "não encontrado" usada para protocolo
   inexistente, sem indicar que o recurso existe.

---

### User Story 6 - Isolamento de proposta que falha no encaminhamento (Priority: P3)

Uma proposta já aceita e persistida não consegue ser encaminhada para processamento após
tentativas sucessivas (ex.: componente de processamento indisponível de forma persistente). Em vez
de ser perdida ou de ficar tentando indefinidamente, ela é isolada com contexto suficiente para um
operador diagnosticar e agir.

**Why this priority**: é uma garantia operacional de última linha — importante para não perder
dado, mas de prioridade mais baixa porque só se manifesta quando os componentes downstream já
falharam repetidamente, um cenário de exceção.

**Independent Test**: simular indisponibilidade persistente do encaminhamento para processamento
após uma solicitação aceita, e confirmar que, após as tentativas configuradas se esgotarem, a
solicitação aparece isolada para inspeção humana com contexto de diagnóstico, sem novas tentativas
automáticas indefinidas.

**Acceptance Scenarios**:

1. **Given** uma solicitação aceita cujo encaminhamento para processamento falha repetidamente,
   **When** as tentativas sucessivas se esgotam, **Then** a solicitação é isolada para inspeção
   humana com contexto suficiente para diagnóstico.
2. **Given** uma solicitação isolada para inspeção humana, **When** nenhuma ação humana ocorre,
   **Then** o sistema não a reprocessa automaticamente nem a descarta.

---

### Edge Cases

- O que acontece quando um reenvio chega com o mesmo identificador de deduplicação mas conteúdo de
  formulário diferente? A primeira versão aceita prevalece; o reenvio é tratado como o mesmo envio
  original e recebe o mesmo protocolo, sem criar segunda solicitação nem substituir os dados já
  persistidos.
- O que acontece se a persistência da solicitação falhar no momento do envio (ex.: armazenamento
  indisponível)? O cliente não recebe protocolo nem confirmação de recebimento — recebe um erro
  claro indicando falha temporária, e nenhuma solicitação parcial é criada.
- Como o sistema trata um protocolo sintaticamente inválido (formato incorreto) numa consulta? É
  tratado da mesma forma que um protocolo inexistente — resposta de "não encontrado".
- O que acontece quando os metadados de contexto (IP, dispositivo, canal) não puderem ser
  capturados por algum motivo técnico? A proposta ainda é aceita se os dados obrigatórios do
  formulário forem válidos; metadados de contexto ausentes não bloqueiam o recebimento.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST validar a presença e o formato de todos os dados obrigatórios da
  proposta (documento, nome, e-mail, data de nascimento, valor, moeda, prazo em meses, finalidade)
  antes de aceitar o envio.
- **FR-002**: Quando algum dado obrigatório estiver ausente ou inválido, o sistema MUST informar
  ao cliente exatamente quais campos estão incorretos e MUST NOT criar nenhuma solicitação.
- **FR-003**: Quando a proposta for válida, o sistema MUST persistir de forma durável uma
  solicitação com identificador único, status inicial "recebida" e data de criação antes de
  confirmar o recebimento ao cliente.
- **FR-004**: O sistema MUST devolver ao cliente o protocolo e a confirmação de recebimento
  imediatamente após a persistência da solicitação, sem aguardar qualquer análise ou processamento
  posterior.
- **FR-005**: Nenhuma proposta aceita (persistida) MUST se perder, mesmo quando os componentes de
  processamento posterior estiverem indisponíveis no momento do envio.
- **FR-006**: O sistema MUST capturar e associar à solicitação os metadados de contexto do envio
  (canal de origem, IP, identificação do dispositivo) quando disponíveis, sem que sua ausência
  bloqueie o recebimento da proposta.
- **FR-007**: O sistema MUST detectar quando o mesmo envio chega mais de uma vez (falha de rede,
  clique duplo) e MUST devolver, em toda tentativa repetida, o mesmo protocolo da solicitação
  originalmente criada, sem criar uma segunda solicitação.
- **FR-008**: O sistema MUST permitir que o cliente consulte sua solicitação pelo protocolo,
  devolvendo status atual, data de criação e data da última atualização.
- **FR-009**: Uma consulta a um protocolo inexistente MUST devolver uma resposta clara de "não
  encontrado", sem revelar informação sobre outras solicitações existentes.
- **FR-010**: Uma consulta a uma solicitação que não pertence ao cliente que consulta MUST ser
  negada com a mesma resposta usada para protocolo inexistente, de forma indistinguível.
- **FR-011**: O sistema MUST atribuir, no momento do recebimento, um identificador de correlação
  único à solicitação, que MUST aparecer em todo registro do sistema relacionado a ela, do envio
  até a conclusão do processamento.
- **FR-012**: Quando uma proposta aceita não conseguir ser encaminhada para processamento após um
  número configurável de tentativas sucessivas, o sistema MUST isolá-la para inspeção humana, com
  contexto suficiente para diagnóstico, em vez de perdê-la ou de continuar tentando
  indefinidamente.

### Key Entities

- **Proposta**: dados enviados pelo cliente em um único envio — dados pessoais (documento, nome,
  e-mail, data de nascimento), dados da proposta (valor, moeda, prazo em meses, finalidade) e
  metadados de contexto capturados automaticamente (canal de origem, IP, identificação do
  dispositivo). Não persiste por si só — só existe como solicitação após aceita.
- **Solicitação**: entidade persistida resultante de uma proposta aceita. Possui identificador
  único (protocolo), status (inicia em "recebida"), data de criação, data da última atualização, e
  o identificador de correlação que a acompanha por todo o ciclo de vida. É o objeto consultável
  pelo cliente.
- **Identificador de Correlação**: valor único gerado no recebimento, presente em todo registro do
  sistema referente à solicitação, usado para rastreá-la do envio até a conclusão.
- **Registro de Isolamento para Inspeção Humana**: contexto de diagnóstico preservado quando uma
  solicitação aceita não consegue ser encaminhada para processamento após tentativas esgotadas —
  inclui a solicitação original e informação suficiente para um operador investigar a causa.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: a confirmação de recebimento chega ao cliente em menos de 500 ms no percentil 95,
  medida do envio da proposta até a resposta com protocolo.
- **SC-002**: 100% das propostas aceitas resultam em solicitação persistida correspondente,
  mesmo em cenário de indisponibilidade dos componentes de processamento posterior, verificado em
  teste de caos/carga.
- **SC-003**: 0% dos envios duplicados (mesma proposta enviada mais de uma vez) resultam em mais
  de uma solicitação persistida.
- **SC-004**: 100% das solicitações produzidas em um período de teste são rastreáveis do envio até
  a conclusão por um único identificador de correlação presente em todos os registros associados.
- **SC-005**: 100% das propostas que esgotam as tentativas de encaminhamento para processamento
  aparecem isoladas para inspeção humana, com zero perda silenciosa observada em teste de falha
  induzida.

## Assumptions

- O cliente já está autenticado ao enviar e consultar propostas (feature 001-autenticacao); a
  identidade usada para associar a solicitação ao cliente e para decidir se uma consulta é
  permitida vem da credencial validada, não de dado enviado no formulário — esta feature reutiliza
  a garantia de isolamento entre clientes já especificada em 001-autenticacao (FR-007/FR-008) e
  foca apenas no comportamento de resposta (protocolo inexistente vs. alheio, indistinguíveis).
- A deduplicação de envio repetido usa um identificador de deduplicação derivado do próprio envio
  (fornecido pelo cliente ou calculado a partir do conteúdo), reaproveitado por tempo suficiente
  para cobrir reenvios por falha de rede ou clique duplo em uma mesma sessão de preenchimento.
- O número de tentativas de encaminhamento para processamento antes do isolamento para inspeção
  humana é configurável, sem valor fixo exigido por esta especificação.
- Moeda segue um padrão de código reconhecido (ex.: ISO 4217); formato de documento segue o padrão
  do país de operação da plataforma.
- Validação nesta feature cobre presença e formato dos dados; regras de elegibilidade de negócio
  (ex.: idade mínima, limites de valor por perfil) pertencem à análise antifraude, fora de escopo
  aqui.

## Fora de Escopo

- A análise antifraude em si (avaliação de risco, decisão de aprovação/recusa da proposta).
- A notificação de mudança de status ao cliente (e-mail, push, etc.).
- A decisão final sobre a proposta e qualquer regra de elegibilidade de negócio associada.
- O fluxo operacional de reprocessamento manual de uma solicitação isolada para inspeção humana —
  esta feature cobre apenas o isolamento, não a ação humana subsequente.

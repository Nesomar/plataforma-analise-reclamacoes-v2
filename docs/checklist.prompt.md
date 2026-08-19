# Prompt para `/speckit.checklist`

Rode depois do `/speckit.tasks` e antes do `/speckit.analyze`. O `/speckit.checklist` gera
"testes unitários para o texto da spec" — ele verifica se os requisitos estão completos,
claros e consistentes, não se o código funciona.

---

/speckit.checklist

Gere um checklist de qualidade para esta feature, cobrindo os eixos abaixo. Para cada item,
a resposta precisa ser verificável olhando apenas para a spec e o plano — se não der para
responder com o que está escrito, é uma lacuna a reportar.

**Completude do fluxo assíncrono**

- Todo caminho de mensagem tem produtor e consumidor definidos, com contrato versionado?
- O comportamento sob reentrega da mesma mensagem está especificado explicitamente?
- O comportamento quando o resultado nunca chega está especificado (prazo limite e destino)?
- O comportamento quando um resultado chega para uma solicitação já finalizada está especificado?
- Toda fila tem destino definido para mensagens que falham repetidamente, com limite de
  tentativas explícito?
- Está definido o que acontece com uma mensagem depois que ela chega ao destino de falha — quem
  é alertado, quem inspeciona, como se reprocessa?
- O payload carrega contexto suficiente para diagnosticar a falha sem consultar outro sistema?

**Estados e transições**

- Todos os estados possíveis da solicitação estão enumerados?
- Toda transição tem gatilho, pré-condição e efeito descritos?
- Estados finais estão marcados como irreversíveis por processamento automático?
- Existe algum estado alcançável do qual não se saia?

**Tratamento de falha**

- Cada dependência externa tem comportamento de falha especificado?
- Está explícito que falha técnica nunca produz reprovação?
- Falhas parciais (mensagem enviada mas estado não atualizado, e vice-versa) foram consideradas?

**Contrato do LLM**

- O formato exigido da resposta está definido de forma verificável por schema?
- O comportamento diante de resposta malformada está definido, incluindo quantas tentativas?
- Está especificado que conteúdo fornecido pelo usuário não pode alterar o comportamento da
  análise?
- Existe critério objetivo para acionar revisão manual por baixa confiança?

**Privacidade e segurança**

- Está definido exatamente quais campos cruzam a fronteira para o provedor externo?
- Está definido o tratamento de dados pessoais em logs e em armazenamento de auditoria?
- Está definido quem pode consultar quais solicitações?

**Mensurabilidade**

- Todo critério de aceitação tem número, unidade e percentil quando aplicável?
- Existe algum requisito escrito com termo vago ("rápido", "confiável", "amigável") sem
  definição operacional?

**Paridade local**

- Todo componente desta feature é exercitável no ambiente local?
- Se algum recurso não for emulável, o plano B está escrito, ou apenas mencionado?
- A diferença entre local e produção está limitada a configuração, sem caminho de código
  alternativo?

**Skills e CI/CD**

- Para cada tarefa de frontend do plano, está claro que `frontend-design`, `react-expert` e
  `ui-ux-prox-max` precisam rodar antes da implementação?
- Para cada tarefa de backend Python, está claro que `fullstack-dev-skills` precisa rodar antes
  da implementação?
- Está explícito que `security-guidance` e `code-review`, nessa ordem, rodam ao final de cada
  tarefa de desenvolvimento — e não apenas uma vez ao final da feature inteira?
- Se a feature adiciona um novo pacote, serviço ou comando de teste, o plano diz o que precisa
  mudar em `ci.yml` para que ele seja coberto pelo pipeline?

**Rastreabilidade**

- Cada requisito da spec tem ao menos uma tarefa correspondente?
- Cada tarefa remete a um requisito? Existe tarefa órfã?
- Cada princípio da constituição tem verificação correspondente em alguma tarefa?

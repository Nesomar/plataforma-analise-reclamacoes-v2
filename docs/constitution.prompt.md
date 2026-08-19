# Prompt para `/speckit.constitution`

Cole tudo abaixo da linha depois do comando.

---

/speckit.constitution

Crie a constituição do projeto "Sistema de Solicitação com Antifraude via LLM" — uma
plataforma assíncrona, orientada a mensagens, em AWS (API Gateway, SQS, ECS), com dois
microserviços em Python que se comunicam exclusivamente por filas (cada uma com DLQ), acesso
controlado por Lambda Authorizer, estado em DynamoDB e análise de risco executada por um LLM
externo (Google Gemini, fora da AWS). O contexto completo está em `docs/`.

Estabeleça os seguintes princípios como não-negociáveis:

1. **Desacoplamento por mensageria.** Microserviços nunca se chamam de forma síncrona. Toda
   comunicação entre domínios acontece por fila. Qualquer proposta de chamada HTTP direta
   entre MS Solicitação e MS Antifraude é uma violação arquitetural.

2. **Idempotência obrigatória.** A entrega de mensagens é *at-least-once*. Todo consumidor
   deve produzir o mesmo efeito ao processar a mesma mensagem N vezes. Transições de estado
   usam escrita condicional. A API de escrita aceita `Idempotency-Key`.

3. **Falha técnica não é decisão de negócio.** Indisponibilidade, timeout ou saída inválida do
   LLM jamais resultam em reprovação da solicitação. Produzem um resultado indeterminado que
   encaminha o caso para revisão humana. O sistema falha em direção à revisão, nunca à negativa.

4. **Saída de LLM é entrada não confiável.** Toda resposta do modelo é validada contra schema
   antes de qualquer uso. Nunca é executada, nunca vira instrução para outro componente, nunca
   é extraída por regex de texto livre. Dados do usuário entram no prompt como conteúdo
   delimitado, jamais como instrução.

5. **Rastreabilidade ponta a ponta.** Um `correlationId` único acompanha o fluxo do primeiro
   clique até a notificação final, presente em toda mensagem e em toda linha de log
   estruturado. Nenhum caminho de código pode perdê-lo.

6. **Privacidade por minimização.** Dados pessoais são reduzidos ao mínimo necessário antes de
   cruzar qualquer fronteira de serviço, e especialmente antes de sair para o provedor de LLM.
   Segredos vivem apenas em cofre gerenciado.

7. **Contratos explícitos e versionados.** Toda mensagem carrega `tipo` e `versao`. Mudança
   incompatível exige nova versão, com período de convivência. Contratos são a fonte de
   verdade entre times.

8. **Qualidade verificável.** Regras de domínio (máquina de estados, idempotência, parsing de
   veredito) têm teste automatizado antes da implementação. O caminho completo tem ao menos um
   teste ponta a ponta. Cobertura mínima de 80% no domínio.

9. **Observabilidade como requisito, não como extra.** Logs estruturados, tracing distribuído
   através das filas e métricas de negócio (taxa de aprovação, taxa de indeterminados, latência
   e custo por análise) fazem parte da definição de pronto de qualquer feature.

10. **Resiliência explícita.** Toda fila tem DLQ com alarme, e o reprocessamento a partir dela
    é sempre um ato humano deliberado. Toda chamada externa tem timeout, retry com backoff e
    jitter, e circuit breaker. Escala é dirigida por profundidade de fila.

11. **Controle de acesso que falha fechado.** Nenhuma operação ocorre sem identidade verificada.
    A identidade é determinada apenas pela credencial apresentada — nada que o solicitante
    escreva no corpo da requisição a influencia. Indisponibilidade do controle de acesso nega,
    nunca libera.

12. **Paridade entre ambiente local e produção sem código condicional.** A configuração
    diferencia os ambientes; o código, não. Nenhum adapter contém condicional de ambiente.
    Ao mesmo tempo, verde no emulador local nunca é tratado como prova de correção em nuvem —
    permissões, limites e comportamento sob carga exigem validação em ambiente real.

Inclua também estes dois princípios de processo:

11. **Skills obrigatórias por tipo de tarefa.** Todo trabalho de frontend carrega as skills
    `frontend-design`, `react-expert` e `ui-ux-prox-max` antes de qualquer código ser escrito.
    Todo trabalho de backend Python carrega `fullstack-dev-skills` da mesma forma. Ao final de
    qualquer tarefa de desenvolvimento, `security-guidance` roda antes de `code-review` — nessa
    ordem — antes de a tarefa ser considerada concluída.

12. **Todo commit passa pelo gate de CI antes do merge.** Push em branch dispara lint, checagem
    de tipos e testes contra o MiniStack; verde, abre PR automaticamente; o PR reexecuta o mesmo
    gate como status check obrigatório. Nenhum código chega a `main` sem esse gate verde.

Defina também os critérios de governança: como emendar a constituição, como registrar exceções
justificadas, e a exigência de que todo plano de implementação declare explicitamente sua
conformidade com cada princípio.

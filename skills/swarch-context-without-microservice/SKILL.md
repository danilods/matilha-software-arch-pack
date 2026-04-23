---
name: swarch-context-without-microservice
description: "Use when someone is about to quebrar em microserviços, asking 'preciso de microserviço agora?' ou 'ainda tá num monólito, tá bem?' — segura a decisão até o bounded context lógico se provar sob pressão de feature."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando alguém (humano ou agente) quer converter fronteira lógica
em fronteira de deploy antes da hora. Sinais típicos:

- "Precisamos quebrar isso em microserviços?"
- "Ainda tá num monólito, tá bem assim?"
- "Vamos separar esse módulo em serviço próprio pra escalar o time."
- "Microserviço desde o dia 1 é boa prática, não é?"
- "Esse módulo ficou grande, vamos extrair pra deploy separado."
- Agente propõe Lambda/ECS service novo sem ter provado separação lógica.

A regra de fundo: **BOUNDED CONTEXT É LINHA DE LINGUAGEM, NÃO LINHA DE
DEPLOY. Microserviço é consequência, não causa.** Prove o contexto lógico
como módulo num deploy único primeiro. Se a separação se sustenta sob
pressão de feature, aí considere separar o deploy.

## Preconditions

- Existe pelo menos um candidato a bounded context identificado
  (vocabulário próprio, invariantes próprias). Se nem isso está claro,
  o problema é anterior — use `swarch-context-by-vocabulary` primeiro.
- O módulo já existe em código ou está em desenho. Se é greenfield
  total, desenhe o módulo lógico primeiro; deploy vem depois.
- Existe alguém (você, o agente, o PR) defendendo a separação em serviço
  e alguém (ou ninguém ainda) perguntando *por quê agora*.

## Execution Workflow

1. Nomeie o bounded context candidato. "Notifications é um contexto
   próprio: vocabulário = channel, template, dispatch; invariantes =
   todo dispatch tem evento de origem rastreável."
2. Pergunte: **esse contexto já existe como módulo coerente dentro do
   deploy atual?** Módulo = pasta/package/schema com modelo interno
   coerente, fronteira de import explícita, comunicação com o resto
   via eventos ou API pública do módulo. Se não existe, comece por aí.
3. Aplique o teste de separação sob pressão de feature. Pegue uma
   feature realista do roadmap e pergunte:
   - Implementar essa feature toca só neste módulo ou cascateia em N?
   - Mudar schema interno deste módulo afeta outros módulos?
   - Se este módulo virar caixa-preta, os outros continuam funcionando?
   Se as respostas são limpas no monólito, o bounded context está
   provado — deploy separado é decisão separada, adiável.
4. Liste as razões concretas para separar o deploy **agora**:
   - Escala diferente (um módulo recebe 100x mais tráfego).
   - Tecnologia diferente (precisa de runtime/linguagem distinta).
   - Time separado com cadência de release incompatível.
   - Isolamento de falha crítico (este serviço não pode cair com os outros).
   Se nenhuma das quatro aplica, separar deploy é custo sem retorno.
5. Se decide manter no monólito, documente explicitamente: "Notifications
   é bounded context, vive como módulo `notifications/` dentro do deploy
   `argos-app`. Comunicação com outros contextos via eventos publicados
   em EventBridge. Separação de deploy reavaliada quando X." Deixe o
   gatilho de reavaliação escrito.
6. Se decide separar, documente: qual a dor concreta, qual o contrato
   público (eventos, API), plano de migração, como testar que a
   separação não quebrou o invariante. E lembra: microserviço traz
   operação distribuída (observabilidade, rede, deploy coordenado).

## Rules: Do

- Descobrir bounded contexts pelo vocabulário e invariantes primeiro.
  Deploy é decisão subordinada.
- Provar o contexto como módulo dentro do deploy atual antes de propor
  serviço novo. Um PR mostrando feature realista 100% contida no módulo
  é evidência forte.
- Registrar o gatilho concreto de reavaliação ("quando este módulo
  receber >X req/min" ou "quando time B for criado").
- Pagar o custo de microserviço só quando a dor for medida (limite de
  escala, incompatibilidade tecnológica, time separado, isolamento
  crítico).
- Preferir separar o deploy de **um** contexto por vez, mantendo
  os outros como módulos no monólito enquanto não há dor neles.

## Rules: Don't

- Não quebre em microserviços "pra ficar moderno" ou porque "é boa
  prática". Boa prática é bounded context claro; deploy separado é
  trade-off.
- Não aceite "microserviços desde o dia 1" sem contextos lógicos
  provados — você vai acabar com microservice-spaghetti (cadeia de
  dependências síncronas, schema vazando entre serviços).
- Não use pasta/package como proxy de bounded context sem modelo
  interno coerente. Pastas são gratuitas; contextos são disciplinados.
- Não extraia serviço porque "o módulo ficou grande". Módulo grande com
  modelo coerente é bem melhor que 5 microserviços acoplados.
- Não confunda "serviço separado = deploy independente" com "bounded
  context". Você pode ter contexto claro no monólito; e pode ter 5
  serviços sem nenhum contexto.

## Expected Behavior

Após a skill, a decisão fica explícita. Ou o bounded context já está
provado como módulo e não há dor medida que justifique deploy separado
(fica no monólito, com gatilho de reavaliação anotado), ou existe dor
concreta mapeada (escala, tech, time, isolamento) e a migração vai
acompanhada de contrato público, plano e custo operacional reconhecido.

Resultado de longo prazo: o sistema evolui com menos microservice-
spaghetti. Cada serviço que existe tem razão de existir (foi provado
como contexto primeiro, depois separado por dor real). Time ganha
velocidade em vez de pagar coordenação distribuída desnecessária.

## Quality Gates

- O bounded context candidato tem vocabulário e invariantes nomeados
  em uma frase cada.
- O módulo lógico existe como estrutura no código (pasta/package/schema)
  com fronteira de import explícita, ou há um plano curto para criá-lo
  antes de considerar deploy separado.
- Uma feature realista foi usada pra testar a separação sob pressão;
  o resultado (cascateou ou não) está anotado.
- Se a decisão é separar deploy: existe pelo menos uma das quatro
  razões (escala, tech, time, isolamento) documentada com evidência,
  não opinião.
- Se a decisão é manter no monólito: existe gatilho concreto de
  reavaliação escrito, para não virar tabu permanente.

## Companion Integration

**Pareia com `swarch-context-by-vocabulary`** (Family 4). Esta skill
assume que o contexto foi descoberto pelo vocabulário; se o contexto
ainda não está claro, aplique `swarch-context-by-vocabulary` primeiro,
depois volta aqui pra decidir deploy.

**Pareia com `swarch-acl-between-contexts`** — o ACL garante que
comunicação entre contextos segue contrato de evento mesmo quando eles
rodam no mesmo processo. Contexto lógico no monólito sem ACL é
módulo vazando modelo interno, o que você pagará caro quando/se
separar deploy.

**Complementa `matilha-sysdesign-pack:sysdesign-*`** (pack genérico
de system design). Sysdesign cobre quando usar microservices vs
monolith em termos estruturais; esta skill traz a disciplina concreta
de "prove contexto no monólito primeiro, separe deploy com razão
medida" a partir dos casos Argos + Gravicode.

## Output Artifacts

- Registro curto da decisão: contexto candidato, vocabulário/invariantes,
  teste de separação sob feature, decisão (ficar ou separar), gatilho
  de reavaliação.
- Se manter no monólito: nota no README/ADR dizendo "X é bounded context,
  vive como módulo. Reavaliar quando Y."
- Se separar deploy: ADR completo com contrato público do serviço, plano
  de migração, custo operacional reconhecido (observabilidade, deploy
  coordenado, rede).

## Example Constraint Language

- Use "must" para: nomear vocabulário e invariantes do contexto
  candidato antes de qualquer proposta de deploy separado; documentar
  o gatilho de reavaliação quando decide manter no monólito.
- Use "should" para: provar o contexto como módulo (feature realista
  100% contida) antes de considerar separar; listar explicitamente qual
  das quatro razões (escala, tech, time, isolamento) justifica separar.
- Use "may" para: qual formato usar pro registro (ADR, nota em README,
  comentário em plano), qual exato gatilho numérico escolher.

## Troubleshooting

- **"Mas microserviço é boa prática moderna"**: boa prática é bounded
  context claro. Microserviço é deploy. Muitos sistemas excelentes
  rodam bounded contexts claros num deploy único e ganham com isso
  (tempo de deploy, observabilidade integrada, custo baixo de rede).
- **"Esse módulo ficou grande demais"**: grande não é justificativa de
  separar deploy. É sinal de que talvez sejam dois contextos misturados
  — use `swarch-context-by-vocabulary` pra decidir se quebra em dois
  módulos dentro do mesmo deploy primeiro.
- **"Outro time vai cuidar disso"**: time separado é um dos quatro
  gatilhos válidos, mas só quando a cadência de release ou autonomia
  técnica exige. Se o outro time tá feliz mergeando no mesmo repo com
  CI separado, isso resolve sem deploy novo.
- **"Queremos escalar só esse módulo"**: isso é gatilho válido, mas
  meça antes (`swarch-measure-before-scale`). "Acho que vai escalar
  diferente" sem número é chute.
- **"Já temos microserviços, como saber se acertamos?"**: aplique os
  testes de fumaça do rule de origem: se time A dobrar, time B muda
  velocidade? se A trocar de linguagem, B muda código? se A deleta
  tabela interna, B percebe? Respostas desejadas: não, não, não.

## Concrete Example

Argos — quatro bounded contexts (Target Management, Changes, Knowledge
Base, Notifications) rodam como quatro módulos dentro de um deploy
lógico único (mesma AWS account, mesmo Postgres cluster com schemas
separados, mesmo EventBridge bus). Não são microserviços — são módulos
com fronteira de linguagem clara.

Teste de pressão real aplicado: "queremos Slack receber só alertas
CRITICO no canal #editais, INFORMATIVO vai pra #geral". Antes de
bounded contexts claros, essa feature teria tocado produtor, schema
de evento, cliente Slack — cascateamento clássico. Com bounded
contexts, a feature ficou 100% dentro de Notifications. O produtor
já publica `ContentProcessed` com `classification`; Notifications
decide o roteamento. Resultado: ~4 horas, 1 PR, zero toque nos outros
contextos.

Decisão registrada: manter os quatro como módulos. Gatilho de
reavaliação: "se Changes receber >10x o tráfego de Target Management
consistentemente, extrair Changes pra deploy próprio". Até lá,
observabilidade integrada e deploy único pagam mais que separação.

## Sources

- [[docs/rules/Bounded Contexts na Prática]]
- Síntese direta da regra do Danilo. Preserva "BOUNDED CONTEXT É LINHA
  DE LINGUAGEM, NÃO LINHA DE DEPLOY", "Microserviço é consequência,
  não causa", o caso concreto dos quatro módulos Argos no deploy único,
  e os sinais de alerta de microservice-spaghetti.

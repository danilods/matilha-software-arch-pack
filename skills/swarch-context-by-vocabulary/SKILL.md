---
name: swarch-context-by-vocabulary
description: "Use when alguém pergunta se algo deve virar serviço novo ou onde desenhar a fronteira — Bounded Context é descoberto pelo vocabulário e invariantes do domínio, não pelo org chart ou pelo deploy."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando aparece a pergunta "isso é um novo serviço?", "onde
desenho a fronteira?", "é a mesma entidade ou são duas coisas
diferentes?", ou quando o time está criando módulo/pasta nova sem
uma heurística de onde cortar. Fira também quando duas equipes
discutem o "nome certo" de um conceito ("target" vs "canal" vs
"destinatário") — sinal clássico de bounded contexts sobrepostos.
Este skill é sobre **descobrir a fronteira pelo vocabulário e pelos
invariantes**. A conversa "preciso quebrar em microserviço?" vive
em skill irmão (`swarch-context-without-microservice`); "evento tá
vazando modelo interno?" vive em `swarch-acl-between-contexts`.
Este skill para antes: identifica que existe fronteira.

## Preconditions

- Existe domínio o suficiente para haver vocabulário rico. Em
  sistema de 200 linhas com 3 endpoints CRUD, bounded context é
  overengineering — use um módulo e siga em frente.
- Há acesso a **domain expert** (product, cliente, especialista)
  ou transcrição de conversa dele. Sem vocabulário vindo do
  domínio real, você está modelando sua própria confusão.
- O time aceita que "bounded context" ≠ "microserviço". Se o
  primeiro instinto é "cada contexto vai virar um deploy", pause
  e ancore: contexto é decisão de **modelo**; deploy é decisão
  separada.
- Existe espaço para documentar fronteiras (ADR, wiki, context map).
  Contexto descoberto em conversa e nunca escrito não sobrevive
  rotatividade.

## Execution Workflow

1. **Sente com o domain expert** (ou leia a transcrição de uma
   conversa com ele) e peça para explicar o fluxo ponta-a-ponta
   **com as palavras dele**. Não interrompa para traduzir em
   termos técnicos.
2. **Escute trocas de vocabulário.** Quando ele começa a usar
   palavras diferentes para a mesma "coisa" — "aqui a gente chama
   de *target*, mas quando chega na parte de notificação a equipe
   do Slack chama de *canal*" — marque o ponto. Isso é fronteira
   em desenvolvimento.
3. **Escute trocas de invariantes.** "Aqui pode ser desativado a
   qualquer momento; lá, uma vez criado não pode sumir." "Aqui
   tem dono; lá tem destinatário." Invariante diferente para o
   que parece ser a mesma entidade = dois modelos.
4. **Mapeie os modelos locais.** Para cada fronteira candidata,
   escreva:
   - **Vocabulário** (5-10 termos centrais daquele contexto).
   - **Invariantes** (3-5 regras que sempre valem naquele
     contexto).
   - **Decisores** (quem toma decisão sobre mudanças ali).
5. **Aplique o teste de fumaça** (entre dois candidatos a
   contexto):
   - Se o time A dobra de tamanho, B precisa mudar ritmo de
     release? Desejado: **não**.
   - Se A troca de linguagem, B muda código? Desejado: **não**.
   - Se A deleta tabela interna, B percebe? Desejado: **não**.
   Respostas "sim" indicam fronteira vazando ou inexistente.
6. **Nomeie o contexto explicitamente** (Target Management,
   Changes, Knowledge Base, Notifications). Nome vira identidade
   — evite genéricos ("service", "module", "core").
7. **Documente o context map.** Diagrama simples com nome +
   vocabulário central + formato de interação com outros contextos
   (evento, query, ACL). ASCII basta. Vive no repo.

## Rules: Do

- Descubra contextos **pelo vocabulário e pelos invariantes**, não
  pelo org chart. Dois times fazendo CRUD no mesmo modelo não
  viram dois contextos; um time operando dois modelos distintos
  pode ser dois contextos.
- Nomeie cada contexto com substantivo do domínio
  (Target Management, Changes). Nomes técnicos (API, Service,
  Backend) apagam o significado e destroem o benefício.
- Teste a fronteira sob pressão de feature. "Nova feature: Slack
  só recebe CRITICO no canal #editais." Implementação deve ser
  **local a um contexto** (Notifications). Se vazar para N
  contextos, a fronteira está errada.
- Prove o bounded context num **módulo dentro de deploy único**
  antes de cogitar microserviço. Se a separação lógica não se
  sustenta no monólito, separar fisicamente só amplifica o
  problema.
- Documente o context map no repo. Conversa não persiste; ADR +
  diagrama sim.

## Rules: Don't

- Não confunda bounded context com camada arquitetural
  (frontend/backend/database). Camadas organizam **como** o
  software roda; contextos organizam **o que** o software modela.
- Não crie contexto por pasta vazia. Pastas são gratuitas;
  contextos exigem modelo interno coerente com invariantes. Uma
  pasta chamada `notifications/` com 3 arquivos sem invariantes
  próprios não é contexto, é diretório.
- Não force o mesmo schema em todos os lugares. Se "cliente" tem
  8 campos no Billing e 12 no CRM porque cada contexto precisa de
  atributos diferentes, isso é saudável — não bug. Schema único
  global é o anti-padrão.
- Não resolva "mesma palavra, significados diferentes" escolhendo
  "a definição certa". Geralmente ambas estão certas, em
  contextos diferentes. Dê nomes diferentes por contexto
  (`Target` no Management, `MonitoredURL` no Changes).
- Não deixe deploy decidir contexto. "Esse código está no mesmo
  repo, então é o mesmo contexto" ignora que repo é artefato de
  CI, não fronteira de modelo.

## Expected Behavior

Depois do skill, o time tem lista nomeada de bounded contexts, com
vocabulário central + invariantes + decisores por contexto. A
conversa "isso é novo serviço?" passa a ter etapa intermediária:
"isso é novo **contexto**? se sim, módulo primeiro; serviço depois
se e quando fizer sentido."

Discussões de nomenclatura param de ser brigas ("mas target
significa isso") e viram mapeamento ("no contexto A é target-
configuração, no B é target-estado"). Features ganham tamanho
previsível: alterações internas a um contexto raramente tocam
outros.

## Quality Gates

- Lista nomeada de contextos existe no repo (ADR ou context map).
- Cada contexto tem vocabulário central (5-10 termos) + invariantes
  (3-5 regras) documentados.
- Teste de fumaça aplicado entre pares de contextos; resultados
  registrados.
- Ao menos uma feature recente foi implementada "dentro de um
  contexto" sem cascata — evidência de que a fronteira segura.
- Context map versionado e atualizado quando contextos nascem ou
  se fundem.

## Companion Integration

- **Antecede** `swarch-context-without-microservice`: aquele skill
  decide deploy (módulo vs microserviço); este decide que existe
  fronteira.
- **Antecede** `swarch-acl-between-contexts`: aquele trata
  tradução/ACL entre contextos identificados aqui.
- **Dialoga com** `swarch-fact-vs-command-events` e
  `swarch-event-gateway-boundary`: eventos são o contrato entre
  contextos. Bounded context claro → eventos limpos entre
  contextos.
- **Complementa** `matilha-sysdesign-pack:sysdesign-microservices-decomposition`:
  aquele skill trata a decomposição em larga escala (estratégias,
  anti-padrões); este foca na heurística de descoberta pelo
  vocabulário e invariantes, que é o insumo antes de decompor.
- Metodologia: fase 10 (discovery) e 20 (spec) — descoberta
  antes da implementação.

## Output Artifacts

- ADR / context map no repo listando contextos nomeados.
- Por contexto: vocabulário central + invariantes + decisores.
- Diagrama simples (ASCII ou mermaid) mostrando contextos + canais
  de comunicação (evento, query).
- Nota sobre quando o contexto deveria ser promovido a deploy
  separado (critério explícito, não "quando sentirmos").

## Example Constraint Language

- Use "deve" para: descobrir por vocabulário + invariantes, nomear
  contexto com substantivo do domínio, documentar context map,
  provar a fronteira num módulo antes de separar deploy.
- Use "pode" para: começar com 2-3 contextos e descobrir mais
  depois, usar nomes locais diferentes para o mesmo termo entre
  contextos, manter vários contextos num deploy único
  indefinidamente.
- Use "nunca" para: confundir contexto com camada arquitetural,
  forçar schema único global, deixar dois times brigando sobre
  "nome certo" sem perceber que são dois conceitos.

## Troubleshooting

- **"Não consigo distinguir — parece a mesma coisa em todo lugar"**:
  sinal de que domínio ainda é raso ou você está modelando do
  lado técnico. Volte ao domain expert e peça **três exemplos
  concretos** (pedidos reais, casos reais). Vocabulário aparece
  na concretude.
- **"Time insiste em 'é tudo cliente, só tem dados diferentes'"**:
  pergunte pelos invariantes. "Cliente no Billing pode ter método
  de pagamento inválido?" vs "Cliente no Shipping pode ter
  endereço inválido?". Invariantes diferentes = modelos diferentes
  = contextos diferentes, mesmo que a palavra seja igual.
- **"Temos 15 contextos candidatos — é contexto demais?"**:
  provavelmente sim. Contextos demais sugere que você está
  cortando por função técnica (auth, logging, cache) em vez de
  por domínio. Contextos são **sobre o negócio**, não sobre
  infraestrutura. Consolide.
- **"Descobrimos os contextos, mas o deploy continua monolítico
  e tá tudo bem"**: excelente. Bounded context é decisão de
  modelo; deploy é separado. Mantenha assim enquanto o modelo
  se sustenta — separar só quando houver razão concreta (time
  separado, escala específica, tecnologia divergente).

## Concrete Example

Argos v3. Durante discovery, domain expert descreve o fluxo:
"A gente cadastra os **targets** (URLs com regras de monitoramento).
Daí o sistema faz crawl periódico — quando acha **mudança**
relevante, classifica como CRITICO, INFORMATIVO ou IRRELEVANTE e
extrai **entidades**. Aí a parte de notificação fan-out para
**canais** certos — cada canal tem template próprio."

Vocabulário detectado:
- **Target Management**: target, domain, schedule, rate limit,
  allowed methods. Invariante: todo target tem dono; todo target
  tem domínio registrado.
- **Changes**: hash, diff, classification (CRITICO / INFORMATIVO
  / IRRELEVANTE), extracted entities. Invariante: toda mudança
  tem conteúdo antes/depois imutável em S3.
- **Knowledge Base**: entity, relation, confidence, source.
  Invariante: toda entidade tem ao menos uma fonte rastreável.
- **Notifications**: channel, template, dispatch,
  acknowledgment. Invariante: todo dispatch tem evento de origem.

Quatro bounded contexts descobertos. Todos rodam no mesmo deploy
(monolito lógico), mesmo EventBridge bus, mesmo cluster PG com
schemas separados. Não compartilham modelos; compartilham
**eventos**.

Teste de pressão: feature "Slack CRITICO em #editais,
INFORMATIVO em #geral". Implementação 100% interna a
Notifications. ~4h, 1 PR. Fronteiras confirmadas.

Contraste: num desenho anterior onde "notificação" era parte de
"Changes", a mesma feature exigiria mudar publisher + schema de
evento + cliente Slack — cascata por 3 camadas. Fronteira mal
posta = feature cara. Fronteira bem posta = feature barata.

## Sources

- `[[docs/rules/Bounded Contexts na Prática.md]]` — seção
  "Descubra contextos pelo vocabulário, não pelo org chart" +
  "Heurística para encontrar contextos" + teste de fumaça.
- Destilado das regras de Danilo — Argos v3 (4 bounded contexts)
  e Gravicode (Consulta Clínica vs Knowledge Graph).

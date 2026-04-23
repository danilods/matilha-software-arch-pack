---
name: swarch-event-gateway-boundary
description: "Use when 1 evento precisa chegar a múltiplos consumidores heterogêneos, when decidindo 'EventBridge ou chamada direta?', or when fan-out via fila/tópico está virando acoplamento por nome — desenha Event Gateway (bus dedicado + regras por consumer) como fronteira pública do bounded context."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando a decisão entre publicar evento num gateway vs.
fazer chamada direta está em jogo — ou quando o gateway atual
está virando bagunça. Sinais típicos:

- "Quando publico no EventBridge vs chamada direta?"
- "Tenho 3 consumidores com necessidades diferentes — como
  organizo?"
- "Um quer processar em ms, outro pode esperar minutos."
- "Minha fila SQS virou 4 filas; preciso mudar isso cada vez
  que adiciona consumidor?"
- O produtor sabe nominalmente quais consumidores existem (lista
  de filas, lista de Lambdas-alvo).

Regra raiz: **Event Gateway quando há N consumidores
heterogêneos.** Se 1 evento tem 1 consumidor → chamada direta
pode ser mais honesta. Se 1 evento tem 3+ consumidores com
latências e necessidades diferentes, um bus dedicado (EventBridge
bus + regras por consumer) desacopla de verdade. O bus é a
**fronteira pública** do bounded context.

## Preconditions

- Existe (ou está sendo proposto) um evento com múltiplos
  consumidores potenciais, ou o fan-out atual está dolorido.
- Os consumidores podem ter perfis diferentes (latência, shape
  de dado interessante, reliability). Se todos são idênticos,
  SNS + N filas SQS resolve — esta skill é pra fan-out
  heterogêneo.
- O substrato está disponível: EventBridge, Kafka com múltiplos
  consumer groups, ou similar com filtragem por consumer.

## Execution Workflow

1. Liste os consumidores do evento. Para cada um, anote: (a)
   latência tolerada, (b) quais campos do payload realmente
   usa, (c) reliability requerido (at-least-once, exactly-once,
   best-effort), (d) pertence ao mesmo bounded context ou
   cruza fronteira.
2. Conte consumidores heterogêneos. Se é **1**, considere
   chamada direta (HTTP, Lambda invoke, fila dedicada). Se é
   **3+** com perfis diferentes, o gateway se paga.
3. Escolha o substrato. EventBridge bus dedicado com regras por
   consumer é o default AWS. Nomeie o bus pelo contexto
   produtor (`argos.platform`, `gravicode.orchestrator`) — nunca
   pelo consumidor.
4. Desenhe uma regra por consumidor, com filtro baseado nos
   metadados do payload (`classification`, `category`,
   `severity`). A regra tem alvo (Lambda, fila SQS, HTTP
   endpoint) — o produtor não referencia nenhum.
5. Distinga **mecanismo interno** (estado + Streams dentro do
   bounded context — ver `swarch-lambda-chain-shape`) do
   **contrato público** (EventBridge bus entre bounded
   contexts). Não confunda os dois; um serve coreografia
   interna, o outro serve fronteira.
6. Documente o bus como API pública. Quem consome do
   `argos.platform` está dentro do contrato — quem precisa
   pode subscrever sem pedir permissão; quem produz não pode
   quebrar o schema sem versionar.

## Rules: Do

- Use Event Gateway (bus dedicado + regras por consumer)
  quando há 3+ consumidores heterogêneos, **ou** quando o
  número é 2 mas cresce rapidamente, **ou** quando a fronteira
  é entre bounded contexts distintos.
- Nomeie o bus pelo produtor/contexto (`argos.platform`) — o
  bus pertence a quem produz, não a quem consome.
- Filtre por metadados do payload (ver
  `swarch-fact-vs-command-events` — quanto mais metadado
  estável, mais granular a filtragem).
- Mantenha **uma** detail-type por evento lógico. Se precisa
  publicar "a mesma coisa" em buses diferentes, está duplicando
  contrato público — consolida.
- Separe coreografia interna (estado + Streams) do contrato
  público (EventBridge). Dentro da pipeline, estado; entre
  bounded contexts, evento.

## Rules: Don't

- Não use EventBridge para 1 consumidor. Adiciona latência sem
  desacoplar. Chamada direta é mais honesta e debugável.
- Não faça o produtor referenciar o nome do consumidor em
  código. Se a lista de alvos está hardcoded, a fronteira
  vazou.
- Não use o bus default da AWS (`default`) para eventos
  públicos do seu domínio. É poluição cruzada com AWS
  Service Events — crie bus dedicado.
- Não publique a mesma `detail-type` em buses diferentes para
  consumidores diferentes. Agora tem dois contratos públicos
  pra manter sincronizados — é armadilha.
- Não use EventBridge entre cada passo de uma pipeline do
  mesmo bounded context. Para isso, estado + Streams é mais
  simples e barato (ver `swarch-lambda-chain-shape`).

## Expected Behavior

Após aplicar a skill, o evento público é publicado uma vez num
bus nomeado pelo produtor. Cada consumidor tem uma regra
EventBridge filtrando o que lhe interessa, com seu próprio
alvo. O produtor é ignorante sobre consumidores. Adicionar um
consumidor novo é criar uma regra + alvo — zero linha no
produtor.

No longo prazo: o bus vira API pública estável do contexto
produtor. Novos times se plugam subscrevendo, evoluem no seu
ritmo, e o produtor não precisa coordenar. É a propriedade que
permite a plataforma Argos ter quatro consumidores (pedagógica,
dashboard, Slack, auditoria) evoluindo independentemente.

## Quality Gates

- Bus tem nome do produtor/contexto, não da feature ou do
  consumidor. `grep` confirma convenção.
- Produtor não referencia nenhum consumidor em código —
  apenas publica no bus. Lista de alvos vive na infra como
  regras EventBridge (Terraform/CloudFormation).
- Cada consumidor tem uma regra própria, filtrando por
  metadados do payload. Regras documentadas em IaC.
- Mesma `detail-type` não é publicada em buses diferentes.
- Coreografia interna da pipeline não usa o bus público
  (usa estado + Streams ou fila interna).

## Companion Integration

**Complementa `matilha-sysdesign-pack:sysdesign-event-streaming-kafka`** —
sysdesign decide **Kafka vs alternativas** a partir de NFRs
(throughput, retention, replay). Esta skill desenha a **fronteira
de Event Gateway** entre bounded contexts — válida em qualquer
substrato (EventBridge no AWS, Kafka com múltiplos consumer
groups, Pub/Sub no GCP). Use em conjunto: sysdesign escolhe a
tecnologia, swarch desenha a topologia de fan-out.

Pareia também com `swarch-fact-vs-command-events` (o bus só
desacopla se os eventos forem fatos, não comandos) e com
`swarch-lambda-chain-shape` (o bus é a saída pública do final
da pipeline — o que vem antes é coreografia interna).

## Output Artifacts

- Nome do bus (ex.: `argos.platform`, `gravicode.orchestrator`)
  e justificativa do escopo (bounded context produtor).
- Lista de consumidores com perfil (latência, campos de
  interesse, reliability).
- Regras EventBridge (ou equivalente) — uma por consumidor, com
  filtro baseado em metadados do payload.
- Diagrama mostrando produtor → bus → {regra₁ → alvo₁, regra₂ →
  alvo₂, ...}.

## Example Constraint Language

- Use "must" para: nomear bus pelo produtor; proibir que o
  produtor referencie consumidores; exigir uma regra por
  consumidor.
- Use "should" para: preferir filtragem granular por metadados
  em vez de "todo mundo recebe tudo"; preferir IaC
  (Terraform/CloudFormation) para as regras — nunca clicado no
  console sem rastro.
- Use "may" para: granularidade das regras (uma por consumer vs
  uma por combinação severity×category); usar SNS fan-out em
  vez de EventBridge se o filtro é trivial e o substrato já
  existe.

## Troubleshooting

- **"Tenho 2 consumidores — vale o bus já?"**: depende de
  quanto eles divergem e de quanto você espera crescer. Se um
  quer latência <100ms e o outro é batch noturno, já vale. Se
  são idênticos e improvável crescer, chamada direta dupla é
  mais barata.
- **"Meu consumidor B precisa ler dado que o A gera — posso
  acoplar?"**: não. Dados que o A gera são do A; se o B
  precisa, o A emite outro fato (`SomethingProduced`) que o B
  consome. Consumidor nunca lê tabela interna de consumidor
  produtor — só consome evento.
- **"Queria usar a fila default pra não pagar bus dedicado"**:
  EventBridge custom bus custa quase zero em escalas típicas;
  o default bus recebe AWS Service Events e polui filtros.
  Sempre bus dedicado para seus domínios.
- **"Cada consumidor quer schema diferente"**: o schema é um
  só; os consumidores filtram e projetam o que precisam. Se
  precisam de shape totalmente diferente, provavelmente é
  **outro fato** acontecendo — discuta em `swarch-fact-vs-command-events`.

## Concrete Example

Argos publica `ContentProcessed` uma única vez no bus
`argos.platform`. Quatro consumidores heterogêneos:

```
argos.platform (bus)
  ├─ Rule: classification=CRITICO AND category=edital
  │    → Lambda da Plataforma Pedagógica (gera conteúdo)
  ├─ Rule: * (tudo)
  │    → Lambda do Admin Dashboard (projeta em Postgres)
  ├─ Rule: classification IN [CRITICO, INFORMATIVO]
  │    → Lambda do Slack Notifier
  └─ Rule: * (tudo)
       → Firehose → S3 particionado por data (auditoria)
```

Cada consumidor vive no seu próprio time, com seu próprio
deploy, e nenhum deles aparece em código do Argos — a lista
está na infra como regras EventBridge Terraform.

Quando a Plataforma Pedagógica pediu "também quero reagir a
`category=sumula`", o diff foi em **um arquivo Terraform do
time deles** — zero linhas no Argos. Essa é a separação que o
Event Gateway compra.

## Sources

- [[docs/rules/Event-Driven Decoupling]]
- Síntese da regra "Event Gateway quando há N consumidores
  heterogêneos" e da seção "Argos — Event Gateway
  `argos.platform`". Preserva a tabela de decisão
  evento-vs-chamada-direta, o bus nomeado pelo produtor, e a
  distinção entre coreografia interna (estado + Streams) e
  contrato público (EventBridge).

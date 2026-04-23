---
name: swarch-pull-over-push-orchestration
description: "Use when orquestração propõe Lambda chamando ecs:RunTask/lambda:Invoke/dynamodb:PutItem em loop, quando a pergunta é 'como disparo 10k tasks?', ou quando ThrottlingException aparece em carga — força plano de controle por pull (fila/stream)."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando o desenho está prestes a orquestrar escala via chamada
de API síncrona em loop — plano de controle que quebra antes do plano
de dados. Sinais típicos:

- "Lambda chama `ecs:RunTask` num loop pra cada alvo, tá bom?"
- "Como disparo 10k tasks ao mesmo tempo?"
- "Coordinator invoca N workers Lambda um por um."
- "Vou fazer `dynamodb:PutItem` em loop pra backfillar."
- `ThrottlingException` aparecendo nos logs sob carga.
- `CapacityProviderReservation` ou erros de ENI em Fargate.
- Agente propõe "orquestrador" que chama API AWS por entidade.

A regra de fundo: **NUNCA ORQUESTRE ESCALA VIA API SÍNCRONA EM LOOP.
PLANO DE CONTROLE (APIs AWS) QUEBRA ANTES DO PLANO DE DADOS (RUNTIME).
WORKERS FAZEM PULL DE FILA/STREAM. COORDENADOR SÓ DISTRIBUI TRABALHO.**

## Preconditions

- Há N unidades de trabalho (tasks, mensagens, jobs) a serem
  processadas. N ≥ ~100 por burst já justifica o padrão; N ≥ 1k é
  obrigatório.
- O processamento pode ser feito por workers idempotentes (Fargate
  tasks, Lambdas, ECS services, containers em Kubernetes). Se
  precisa de orquestração stateful com forte sincronização, é outro
  padrão (SFN, DAG engine).
- Existe alguém (você, agente, PR) propondo dispatch síncrono ou
  houve falha em produção relacionada a throttling/ENI/capacity.
- A ordem estrita entre todas as N unidades **não** é requisito
  (ou existe partition key que isola grupos). Se é, veja
  `swarch-ordering-decision`.

## Execution Workflow

1. Nomeie o loop síncrono candidato. "Coordinator Lambda vai chamar
   `ecs:RunTask` 10k vezes." "Vou fazer `lambda:Invoke(InvocationType=
   Event)` em loop." "Orchestrator faz `dynamodb:PutItem` por linha."
   Escreve em voz alta.
2. Estime o TPS real do loop proposto. `ecs:RunTask` é ~20 TPS por
   região sem throttling pesado; `lambda:Invoke` concurrency tem
   limite de conta; `dynamodb:PutItem` sem batch fica caro rápido.
   Compare com o burst que você quer (10k tasks em segundos).
3. Identifique que você está usando plano de controle (APIs AWS
   administrativas) onde deveria usar plano de dados (runtime
   consumindo queue/stream). APIs de infra têm TPS limits baixos;
   runtime stateless horizontal aguenta throughput de ordens de
   grandeza maior.
4. Reformule pra pull:
   - Coordinator publica N mensagens na fila SQS (ou stream Kinesis).
     Operação: `sqs:SendMessageBatch` (10 por call) ou `kinesis:
     PutRecords`. Burst suporta milhões/min trivial.
   - Workers **pré-escalados** (Fargate service com auto-scaling, pool
     de Lambdas com event source mapping) fazem pull no ritmo que
     aguentam.
   - Coordinator **não invoca** workers; só publica trabalho.
   - Feedback loop: métrica SQS (ApproximateNumberOfMessagesVisible)
     dispara auto-scaling do worker pool.
5. Se precisa de `ecs:RunTask` pra escalar o pool, use service +
   auto-scaling (Fargate service escala via CPU/queue lag), não task
   ad-hoc por unidade de trabalho. Task = execução efêmera; worker =
   processo consumindo fila.
6. Se precisa de writes em massa (backfill 1M linhas no Dynamo), use
   `BatchWriteItem` (25 por call) + paralelização controlada, ou
   stream processor com commit em batch. Jamais `PutItem` em loop
   serial.
7. Documente trade-offs: latência de pull (1-10s vs invocação
   direta), necessidade de idempotência (retry é esperado), custo de
   fila (SQS ~$0.40/M messages é barato comparado a ThrottlingException
   em produção).

## Rules: Do

- Coordinator publica trabalho em fila/stream. Workers fazem pull.
  Essa é a forma canônica de orquestrar escala em AWS.
- Use `SendMessageBatch` (SQS) ou `PutRecords` (Kinesis) — sempre
  batched. Single-message writes são caros e lentos em volume.
- Workers pré-escalados via auto-scaling vinculado a métricas de fila
  (ApproximateAgeOfOldestMessage, visible messages, Kinesis lag).
- Idempotência nos workers. Retry é esperado no padrão pull
  (visibility timeout, DLQ).
- Timeout + retry com backoff exponencial + jitter em tudo que fala
  com API externa. Retries sem backoff em storm esgotam
  throughput de volta.
- Fila com DLQ configurada desde o primeiro dia. Poison pill sem
  DLQ explode custo silenciosamente.

## Rules: Don't

- Nunca orquestre escala com `ecs:RunTask` em loop. Anti-padrão
  canônico — esgota TPS da API ECS em minutos.
- Não use `lambda:Invoke(InvocationType=Event)` em loop pra disparar
  N workers. Concurrency limit e burst limit batem no mesmo problema.
- Não faça `PutItem`/`DeleteItem`/`UpdateItem` em loop serial pra
  bulk operations. `BatchWriteItem` ou stream-based processing.
- Não dimensione pool de workers por chamada de API administrativa
  dentro do hot path. Service + auto-scaling desacoplado.
- Não confunda "baixa latência" com "dispatch síncrono". Fila SQS
  adiciona 1-10s de latência — insignificante comparado ao custo de
  throttling em produção.
- Não assuma que retry resolve throttling. Sem mudar o padrão (pull
  em vez de push), throttling continua; retry só adiciona storm.

## Expected Behavior

Após aplicar a skill, o desenho tem coordinator publicando batched em
fila/stream e worker pool pré-escalado consumindo por pull. APIs AWS
administrativas (RunTask, Invoke, PutItem single) ficam fora do hot
path. Throughput do sistema é limitado pelo runtime dos workers
(escalável horizontal), não por limits de plano de controle.

Resultado de longo prazo: `ThrottlingException` some dos logs.
Escalar é decisão de capacidade do pool, não de reengenharia. Picos
reais (10x tráfego por 1h) são absorvidos pela fila e processados
gradualmente, sem quebrar.

## Quality Gates

- Nenhuma API AWS administrativa chamada em loop no hot path
  (`ecs:RunTask`, `lambda:Invoke`, `PutItem` serial, `CreateTask`,
  etc).
- Trabalho flui via fila (SQS, Kinesis, SNS+SQS) ou stream
  (Kinesis, DynamoDB Streams, Kafka).
- Workers fazem pull (event source mapping, polling, consumer
  loop). Idempotência documentada.
- Auto-scaling do worker pool vinculado a métrica de fila/stream,
  não a CPU do coordinator.
- Batching em operações que têm batch API (`SendMessageBatch`,
  `BatchWriteItem`, `PutRecords`).
- DLQ configurada. Timeout + retry com backoff exponencial em
  operações externas.

## Companion Integration

**Complementa `matilha-sysdesign-pack:sysdesign-scalability-horizontal-vs-vertical`**
— sysdesign é estrutural (horizontal/vertical/stateful); esta skill
traz o caso concreto Argos (v1 com `ecs:RunTask` em loop que
esgotava TPS em minutos, v2 com SQS + workers pulling). Use sysdesign
pra orientar a decisão geral; volte aqui pra desenhar dispatch
especificamente.

**Pareia com `swarch-ticker-vs-rule-per-entity`** (Family 5). São
lados da mesma moeda: Ticker é *como* gerar trabalho pros workers
de forma agendada; pull-over-push é *como* esses workers consomem
sem esgotar APIs. O par completo aparece em qualquer sistema com N
entidades agendadas.

**Pareia com `swarch-event-gateway-boundary`** (Family 2) — fan-out
via EventBridge também é forma de pull (consumer configurado via
regra), só que pra eventos heterogêneos. Mesmo princípio: coordinator
publica, consumer lê quando aguenta.

**Pareia com `swarch-measure-before-scale`** — antes de refatorar pra
pull, confirme que o problema é throttling/limites de API, não só
"parece lento". Medir evita mudar padrão por sintoma errado.

## Output Artifacts

- Diagrama do fluxo antes/depois: antes (coordinator → API em loop →
  workers) vs depois (coordinator → fila → workers pull).
- Código/CDK do coordinator publicando batched, do worker fazendo
  pull, e da auto-scaling policy do pool.
- Documento de trade-offs: latência adicional aceita, idempotência
  de workers, DLQ strategy, custo de fila vs custo de throttling.
- Opcional: ADR explicando migração de v1 (push síncrono) para v2
  (pull assíncrono) com métricas antes/depois.

## Example Constraint Language

- Use "must" para: coordinator publica em fila (não invoca workers);
  workers idempotentes; DLQ configurada; batching em APIs que têm.
- Use "should" para: auto-scaling vinculado a métrica de fila;
  backoff exponencial + jitter em retries; timeout explícito em
  toda chamada externa.
- Use "may" para: SQS vs Kinesis (depende de ordem e fan-out); tipo
  de worker (Fargate, Lambda, ECS); exato tuning de batch size e
  visibility timeout.

## Troubleshooting

- **"Dispatch síncrono é mais simples, por que complicar com fila?"**:
  é mais simples até o primeiro spike. No volume esperado
  (evidenciado por números), fila é menos complexidade total que
  gerenciar throttling, retries em storm, e explicar pro time por
  que o sistema fica instável em pico.
- **"Fila adiciona latência, não podemos pagar"**: quantifica a
  latência. SQS é 1-10s tipicamente, com long polling bem calibrado.
  Se o SLA é "processo em menos de 10s", OK. Se é "em menos de 50ms",
  você não quer orquestração de 10k workers de qualquer forma —
  repense o problema.
- **"Mas uso SQS FIFO pra ordem, tem limite de 300 msg/s"**: confirma
  se a ordem é de fato requisito (`swarch-ordering-decision`). 99%
  das vezes não é — Standard SQS escala trivial. Se for, use
  partition keys (Kinesis) ou múltiplos message group IDs.
- **"`ThrottlingException` voltou mesmo com fila"**: algum hot path
  ainda chama API administrativa em loop. Grep no código por
  `RunTask`, `Invoke`, `PutItem` e confirma que nenhum está dentro
  de iteração sobre unidades de trabalho.
- **"Backfill de 10M linhas demora horas com BatchWriteItem"**: OK,
  é isso mesmo. Paraleliza múltiplos workers consumindo chunks da
  fila. Se ainda é lento, são limits do storage (provision capacity
  ou on-demand do Dynamo), não do dispatch.
- **"Lambda cold start mata o fluxo"**: event source mapping com
  provisioned concurrency resolve pro hot path. Não provisione o
  mundo inteiro; provisione o que medir mostra que dói.

## Concrete Example

Argos v1 (anti-padrão, foi refatorado):

```
EventBridge Rule
    ↓
Coordinator Lambda
    ↓ (for url in due_urls: ecs.run_task(...))  ← LOOP SÍNCRONO
ECS API (limite ~20 TPS)
    → ThrottlingException em minutos
    → fila de retries storm
    → custo subindo, tasks perdidas
```

Resultado real: em carga moderada (1000 URLs due num minuto), API ECS
throttleava, retries entravam em storm, metade das tasks não rodava,
conta AWS subindo.

Argos v2 (padrão atual):

```
EventBridge Rule (*/1 min)
    ↓
Ticker Lambda
    ↓ (Query GSI → batch de 1000 targets due)
SQS SendMessageBatch (10 msgs por call → 100 calls pra 1000 targets)
    ↓
SQS Crawl Queue
    ↓ (pull pelos workers)
Fargate Service (auto-scaled por ApproximateNumberOfMessagesVisible)
```

Resultado: `ecs:RunTask` nunca é chamada no hot path. ECS Service +
auto-scaling gerencia capacidade do pool. Workers pulling consomem no
ritmo que aguentam. API AWS chamada só na taxa real de processamento,
nunca em burst. ThrottlingException sumiu dos logs.

Custo de fila SQS pra 1M messages/mês: ~$0.40. Custo de debugar
throttling storm em produção: ordens de grandeza mais.

## Sources

- [[docs/rules/Escalabilidade sem Prematuridade]]
- Síntese direta da regra "Plano de controle quebra antes do plano
  de dados" do Danilo. Preserva o princípio (nunca `ecs:RunTask`/
  `lambda:Invoke`/`PutItem` em loop), os failure modes concretos
  (ThrottlingException, CapacityProviderReservation), o caso v1→v2
  do Argos, e a forma canônica (workers pulling, coordinator só
  distribui).

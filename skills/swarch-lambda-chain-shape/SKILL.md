---
name: swarch-lambda-chain-shape
description: "Use when designing a chain of Lambdas (Cleaner → Classifier → Extractor → Router → Notifier style), when handler está virando bagunça, or when perguntando 'uma Lambda ou duas?' — desenha pipeline como adaptadores finos acoplados por estado/evento, nunca por import."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando você está desenhando ou revisando uma pipeline
serverless composta por múltiplas Lambdas (ou equivalente em
Cloud Functions / Azure Functions). Sinais típicos:

- "Como desenho essa cadeia de Lambdas?"
- "Meu handler tá virando bagunça — quebro em duas Lambdas ou
  mantenho uma?"
- "O Classifier também deveria extrair as entidades, ou faço
  Extractor separado?"
- "Preciso que a Lambda A chame a Lambda B — qual a forma certa?"
- Uma Lambda cresceu pra 300 linhas e ninguém quer tocar.
- Duas Lambdas estão importando uma da outra.

Regra raiz: **Lambda Chain é uma pipeline, não uma bagunça.** Cada
Lambda é um adaptador finíssimo (~30 linhas úteis) que faz parse
do evento, chama função de domínio pura, e escreve estado/publica
evento. Nenhuma Lambda conhece a próxima — acoplamento é via
estado (DynamoDB/PG) ou via evento (EventBridge), nunca via import.

## Preconditions

- Você está desenhando uma cadeia de processamento assíncrono com
  2+ Lambdas (ou equivalente) sobre o mesmo domínio funcional.
- O domínio já tem (ou pode ter) uma camada de regras puras
  separada das Lambdas. Se não tem, comece por
  `swarch-dependency-direction` e
  `swarch-handler-as-adapter` antes.
- Existe um substrato de acoplamento disponível: DynamoDB Streams,
  EventBridge, SQS, ou máquina de estados explícita.

## Execution Workflow

1. Desenhe a pipeline em caixas. Nomeie cada passo pelo verbo
   (Clean, Classify, Extract, Route, Notify). Se um passo tem
   "E" (ex.: "Classify **e** Extract"), é provavelmente duas
   Lambdas.
2. Para cada caixa, defina três coisas: (a) evento/estado de
   entrada, (b) função de domínio pura chamada, (c) evento/estado
   de saída. Se não couber em três, a Lambda está fazendo demais.
3. Escolha o mecanismo de acoplamento entre passos. Dentro do
   mesmo bounded context com ordem sequencial: **estado +
   DynamoDB Streams**. Entre contextos diferentes ou com fan-out:
   **evento no EventBridge**. Nunca `boto3.client("lambda").invoke()`.
4. Verifique a forma do handler de cada Lambda. O gabarito é:
   - `payload = parse_event(event)` (adapter — 3 linhas)
   - `result = domain_function(payload.x, payload.y)` (domínio puro)
   - `persist_result(result)` / `publish_event(result)` (infra)
   Se não cabe nesse shape em ~30 linhas, extraia.
5. Verifique que nenhuma Lambda importa outra. `classifier_handler`
   jamais faz `from extractor_handler import ...`. Se precisa, o
   código compartilhado pertence ao domínio.
6. Teste a reprocessabilidade: quebre uma Lambda intencionalmente
   (deploy com bug), corrija, e reprocese — deve bastar atualizar
   o `status` no DynamoDB ou republicar o evento. Se precisa de
   script manual complicado, o acoplamento está errado.

## Rules: Do

- Uma Lambda = uma responsabilidade nomeável por verbo. Cleaner
  limpa. Classifier classifica. Extractor extrai. Lambdas coesas
  são baratas de manter e reprocessar.
- Handler fino: parse → domínio puro → side-effect. ~30 linhas
  úteis no máximo.
- Comunique via **estado** (escreve `status=READY_TO_CLASSIFY`
  no Dynamo, Stream dispara próxima) ou via **evento público**
  no EventBridge, dependendo de ser dentro do mesmo contexto ou
  cruzando fronteira.
- Separe o contrato público (EventBridge `ContentProcessed` no
  fim da pipeline) do mecanismo interno (estado + Streams entre
  passos). São camadas diferentes com semânticas diferentes.
- Torne o reprocessamento trivial: basta atualizar o status no
  Dynamo para reacionar a Stream. Sem script custom, sem
  reengenharia.

## Rules: Don't

- Não faça uma Lambda chamar outra via `boto3.client("lambda").invoke()`.
  Isso recria orquestração síncrona push-driven que a coreografia
  event-driven veio resolver.
- Não importe código de uma Lambda em outra (nem via layer comum
  que expõe lógica de handler). Código compartilhado entre
  Lambdas é domínio — extraia para `domain/`.
- Não faça uma Lambda fazer trabalho de duas. Se `classifier`
  também extrai entidades, o teste fica difícil e o reprocesso
  fica caro.
- Não use EventBridge entre cada passo dentro do mesmo bounded
  context com ordem rígida. Para pipeline sequencial coesa, estado
  + Streams é mais simples e barato.
- Não deixe a pipeline silenciosa — cada passo deve escrever
  status explícito e emitir métricas de progresso (número de
  itens por status).

## Expected Behavior

Após aplicar a skill, a pipeline fica visível como sequência de
caixas com contratos claros. Cada Lambda tem ~30 linhas, é
testável isoladamente (mock de domínio e de infra), e pode ser
substituída sem tocar as vizinhas. Adicionar um passo novo (ex.:
Enricher entre Classifier e Extractor) vira: nova Lambda + novo
status + regra Streams — zero alteração nas Lambdas existentes.

No longo prazo: reprocessar lotes (porque o modelo melhorou,
porque um bug corrompeu classificação) vira `UPDATE Dynamo SET
status='READY_TO_CLASSIFY' WHERE ...`. Essa é a propriedade que
justifica toda a discipline de shape.

## Quality Gates

- Cada Lambda handler mede ~30 linhas úteis ou menos. Verificável
  por contagem.
- Nenhuma Lambda importa outra. `grep "from .*_handler import"`
  em `adapters/lambdas/` retorna vazio.
- Nenhum `boto3.client("lambda").invoke()` entre passos da
  pipeline. Comunicação é via estado ou evento.
- Reprocessamento documentado: "para reprocessar, atualize
  `status=X` — Stream faz o resto". Procedimento existe no
  runbook.
- Métrica por status na pipeline (quantos itens em
  `READY_TO_EXTRACT`, `READY_TO_ROUTE`, etc.) — dá visibilidade
  operacional.

## Companion Integration

**Complementa `matilha-harness-pack:harness-orchestrator-workers`** —
mesma forma abstrata (orchestrator + workers encadeados),
substratos diferentes. `harness-orchestrator-workers` desenha
arquiteturas de LLM-agents (workers paralelos reportando para um
orchestrator); esta skill desenha pipelines de AWS Lambdas
(workers sequenciais acoplados por estado/evento). Quando você
estiver escolhendo entre os dois, pense em: agentes que raciocinam
sobre decisões compartilhadas (orchestrator-workers) vs. passos
mecânicos sobre fluxo de dados (lambda chain).

Pareia também com `swarch-dependency-direction` (regra da seta)
e com `swarch-handler-as-adapter` (testabilidade do handler
individual). Esta skill é a composição; aquelas são os
componentes.

## Output Artifacts

- Diagrama ou ASCII-art da pipeline com caixas nomeadas por verbo
  e arrows anotadas com mecanismo (Stream / EventBridge).
- Lista de status explícitos no DynamoDB (`PENDING`,
  `READY_TO_CLASSIFY`, `READY_TO_EXTRACT`, ..., `COMPLETED`,
  `FAILED`).
- Runbook de reprocessamento ("update status = X to retrigger
  step N").

## Example Constraint Language

- Use "must" para: uma responsabilidade por Lambda; handler
  ≤ ~30 linhas úteis; comunicação entre Lambdas via estado ou
  evento (nunca import ou invoke).
- Use "should" para: preferir estado + Streams dentro do mesmo
  bounded context; preferir EventBridge no contrato público final
  da pipeline; nomear Lambdas por verbo.
- Use "may" para: granularidade do status (incluir timestamps,
  metadados, tentativas); quantas Lambdas formam "uma pipeline"
  antes de virar duas.

## Troubleshooting

- **"Lambda A precisa do resultado de Lambda B agora"**: isso é
  chamada síncrona, não pipeline. Use chamada direta (HTTP,
  Lambda invoke sync) — mas então aceite que não é uma cadeia
  event-driven, é um serviço síncrono.
- **"Acabei com 8 Lambdas para um processo simples"**: reveja
  quais passos realmente pertencem ao **mesmo** bounded context
  com **a mesma** semântica. Provavelmente 2-3 deles podem virar
  funções de domínio chamadas pelo mesmo handler.
- **"A pipeline ficou lenta porque cada passo espera Stream"**: o
  lag de DynamoDB Streams é ~1s. Se a latência end-to-end
  crítica é <500ms, pipeline event-driven pode ser a ferramenta
  errada — considere Step Functions síncrono ou uma única Lambda
  maior.
- **"Reprocessar deu retrabalho porque Lambdas são não-idempotentes"**:
  idempotência por status é parte do contrato. Cada Lambda deve
  checar "já processei esse item?" antes de escrever. Sem isso,
  reprocessamento corrompe dados.

## Concrete Example

Pipeline Argos `ContentProcessed`:

```
Cleaner    → status=READY_TO_CLASSIFY  →[Stream]→ Classifier
Classifier → status=READY_TO_EXTRACT   →[Stream]→ Extractor
Extractor  → status=READY_TO_ROUTE     →[Stream]→ Router
Router     → status=READY_TO_NOTIFY    →[Stream]→ Notifier
Notifier   → status=COMPLETED          →[EventBridge "argos.platform"]→ consumers
```

Dentro da pipeline: **estado + Streams** (mesmo bounded context,
ordem sequencial, só o domínio de processamento de conteúdo
enxerga). No fim: **EventBridge** (contrato público, fan-out para
N consumidores heterogêneos — plataforma pedagógica, dashboard,
Slack, auditoria).

Handler do Classifier (shape alvo):

```python
def lambda_handler(event, context):
    payload = parse_content_processed(event)          # adapter
    classification = classify_content(payload.text)   # domínio (puro)
    persist_classification(payload.id, classification) # infra
    publish_if_critical(payload.id, classification)    # infra
```

Quando o modelo de classificação mudou de Phi-3-mini para Gemini
Flash, o diff ficou inteiro em `infrastructure/ai_client.py`.
Quando uma categoria nova foi adicionada (Súmulas), o diff ficou
inteiro em `domain/classification.py`. Nenhuma outra Lambda
precisou ser reescrita — essa é a forma funcionando.

## Sources

- [[docs/rules/Layering e Dependency Direction]]
- Síntese da seção "Lambda Chain é uma pipeline, não uma bagunça"
  e do padrão observado na prática do Argos. Preserva o shape
  de handler (~30 linhas), a coreografia via estado, a proibição
  de invoke direto, e o exemplo concreto da pipeline
  `ContentProcessed`.

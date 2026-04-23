---
name: swarch-hot-state-via-status-machine
description: "Use when precisa modelar o ciclo de vida de uma entidade processada em etapas (crawl, processamento, notificação) — estado explícito no Dynamo + Stream como trigger, reprocessar = atualizar status."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando o time precisa modelar entidade que passa por **etapas
sequenciais de processamento**: URL sendo crawled, pedido sendo
processado, job de ML em pipeline, conteúdo sendo cleaned → classified
→ extracted → routed → notified. Fira também quando alguém propõe
"array de booleans" (`crawled`, `classified`, `notified`), múltiplas
tabelas de flags, ou orquestração síncrona tipo "Lambda A chama
Lambda B chama Lambda C". O skill é sobre **modelagem**: estado
explícito como máquina + Stream como trigger. Não é sobre decidir
Dynamo vs Postgres (skill 8) nem sobre sincronização entre stores
(skill 9).

## Preconditions

- A entidade tem ciclo de vida com **transições ordenadas** (A → B →
  C), não só CRUD plano. Se é só "criar, editar, deletar", status
  machine é overhead.
- O substrato expõe **change stream** nativo (DynamoDB Streams,
  Mongo change streams, Postgres logical replication com trigger
  via LISTEN/NOTIFY). Sem stream, o padrão "Stream como trigger"
  não se aplica — caia para fila explícita.
- As transições são **idempotentes** (escrever `status=CLASSIFIED`
  duas vezes tem mesmo efeito) ou podem ser tornadas idempotentes
  com `updated_at` / version.
- Time aceita que reprocessar é "atualizar status para trás" e não
  "mandar mensagem especial de retry". Isso é o coração do padrão.

## Execution Workflow

1. **Liste os estados possíveis explicitamente.** Ex.: `PENDING`,
   `CRAWLING`, `READY_TO_CLASSIFY`, `CLASSIFYING`,
   `READY_TO_EXTRACT`, `EXTRACTING`, `READY_TO_ROUTE`, `ROUTING`,
   `READY_TO_NOTIFY`, `NOTIFYING`, `COMPLETED`, `FAILED`. Estados
   intermediários `*-ING` são opcionais mas ajudam a debugar
   processamento lento.
2. **Desenhe o diagrama de transições.** Quem pode ir pra quem.
   Identifique estados terminais (COMPLETED, FAILED) e transições
   válidas. Estado é imutável durante processamento — uma Lambda
   lê `READY_TO_CLASSIFY`, faz o trabalho, grava
   `READY_TO_EXTRACT`. Próxima Lambda dispara pelo Stream.
3. **Escreva uma função por estágio como Lambda independente.**
   Cada Lambda:
   - Filtra Stream events: "me interessa só change com
     `NewImage.status = READY_TO_CLASSIFY`".
   - Busca o item no Dynamo (ou lê do NewImage direto, se couber).
   - Faz o trabalho (domínio puro — ver
     `swarch-handler-as-adapter`).
   - Grava próximo status + timestamp + metadados necessários.
4. **Proíba transições acopladas.** Uma Lambda não chama a próxima
   via `InvokeFunction`. A Lambda só escreve status; o Stream
   dispara a próxima. Isso é o que dá reprocessamento trivial e
   zero acoplamento entre estágios.
5. **Reprocessar = atualizar status.** "Reprocessa todos os
   conteúdos CRITICO de ontem" vira `UPDATE SET status =
   READY_TO_CLASSIFY WHERE ...` (ou equivalente DDB). Stream
   dispara toda a chain novamente. Nenhum código muda.
6. **Trate erros com status `FAILED` + motivo.** Não fique retrying
   in-memory indefinidamente. 3 retries pela Lambda, depois transição
   para `FAILED` com `error_reason` e `error_at`. Reprocessamento
   explícito via mudança de status.

## Rules: Do

- Modele status como **enum finito** documentado. Strings livres
  ("processing", "done", "processing_v2") vão divergir entre
  Lambdas que escrevem e Lambdas que filtram.
- Escreva **timestamp** por transição (`cleaned_at`, `classified_at`,
  `extracted_at`). Isso dá observabilidade + permite SLO por
  estágio.
- Filtre o Stream pelo status que interessa na **regra de
  EventBridge Pipes** ou na **subscription filter** da Lambda, não
  no corpo do handler. Cada Lambda só é invocada com eventos
  relevantes — economia de cold start e de bugs.
- Use **GSI esparso** para estados de exceção (ex.: GSI só para
  `status=FAILED`) para query eficiente de "quantos em erro agora"
  sem scan.
- Documente o diagrama de estados como arquivo no repo
  (`docs/state-machine.md` com ASCII art ou mermaid). Estado
  implícito na cabeça de quem escreveu a Lambda não sobrevive a
  rotatividade.

## Rules: Don't

- Não use boolean flags (`is_crawled`, `is_classified`). Explode
  combinatorialmente e esconde estados impossíveis (`is_classified
  = true` + `is_crawled = false`).
- Não orquestre via `Lambda.invoke()` síncrono na chain. "Cleaner
  chama Classifier" acopla as duas, impede reprocessamento
  seletivo e duplica error handling.
- Não escreva no Postgres **e** no Dynamo dentro da mesma Lambda
  no hot path. Isso é dual-write — ver
  `swarch-cdc-over-dual-write`. Escreva no SoT (Dynamo pra hot
  state); projeção sai via Stream.
- Não publique evento EventBridge público entre estágios internos.
  Todos os estágios pertencem ao mesmo bounded context — eventos
  públicos são para fronteiras de contexto, não para coreografia
  interna. EventBridge só no final (contrato público tipo
  `ContentProcessed`).
- Não deixe a máquina de estados implícita no código. Se ninguém
  consegue listar estados + transições válidas em <2 min, a
  máquina não existe — tem uma coleção de ifs espalhados.

## Expected Behavior

Depois do skill, o ciclo de vida da entidade vira objeto de primeira
classe: enum documentado, diagrama no repo, GSI para queries
operacionais, timestamps por estágio. Reprocessamento vira operação
declarativa ("muda status de X para Y") em vez de código
específico. Adicionar um estágio novo vira criar uma Lambda + um
novo status; estágios existentes não mudam.

Debug de processamento lento muda de "segue o log" para "cadê o
item, em que status está, há quanto tempo?". Operação de
reprocessamento (pipeline falhou em um dia específico) vira query
+ update, não deploy.

## Quality Gates

- Enum de status documentado no repo com diagrama de transições.
- Cada estágio é Lambda (ou worker) independente, disparado por
  change stream filtrado pelo status de entrada.
- GSI esparso para estados operacionais (FAILED, stale) existe.
- Runbook "como reprocessar X" resume-se a query + update de
  status.
- Timestamp por transição gravado (observabilidade + SLO por
  estágio).

## Companion Integration

- **Usa** `swarch-handler-as-adapter`: dentro de cada Lambda de
  estágio, a lógica de negócio é função pura injetada; handler
  só faz parse + persist.
- **Alimenta** `swarch-fact-vs-command-events`: depois de
  `COMPLETED`, a última Lambda publica o evento **público** do
  bounded context (`ContentProcessed`) no EventBridge externo.
- **Depende de** `swarch-dual-store-source-of-truth`: o Dynamo
  precisa estar nomeado como SoT do hot state antes de aplicar
  este padrão.
- **Dialoga com** `swarch-cdc-over-dual-write`: a projeção do hot
  state para o admin DB usa CDC sobre o mesmo Stream que dispara
  as Lambdas de estágio.
- Metodologia: fase 30 (arquitetura) e 40 (implementação).

## Output Artifacts

- `docs/state-machine.md`: enum + diagrama de transições
  (ASCII/mermaid) + semântica de cada estado.
- GSI esparso para estados de exceção documentado no módulo de
  infra.
- Uma Lambda (ou worker) por estágio, com filtro de stream
  explícito.
- Runbook: "como reprocessar X" (1 página, basicamente uma query
  + update).

## Example Constraint Language

- Use "deve" para: enum finito documentado, timestamp por
  transição, GSI esparso para FAILED, estágios independentes.
- Use "pode" para: usar DynamoDB + Streams, Postgres + trigger
  LISTEN/NOTIFY, Mongo + change streams — o padrão é agnóstico
  de substrato, o exemplo Argos usa DDB.
- Use "nunca" para: boolean flags, invocação síncrona entre
  estágios, EventBridge público para coreografia interna.

## Troubleshooting

- **"Uma Lambda de estágio travou, toda a chain parou pra essa
  entidade"**: comportamento esperado. Fix: depois de N retries,
  transição para `FAILED` com motivo + alerta. Não deixe o item
  em estado `*-ING` indefinidamente — rode housekeeping periódico
  que marca `FAILED` itens presos por mais de X minutos em
  `*-ING`.
- **"Mudar status para trás dispara a chain e causa
  duplicação"**: estágios devem ser idempotentes — escrever
  `classification=CRITICO` duas vezes tem mesmo resultado.
  Efeitos colaterais externos (notificação) só acontecem no
  último estágio, que deve ter sua própria idempotência (dedup
  key).
- **"Time quer ver logs encadeados de uma entidade — difícil
  correlacionar Lambdas separadas"**: injete `correlation_id` no
  item (geralmente o próprio PK) e use structured logging com
  esse id. Ferramenta de log consolidado (CloudWatch Insights,
  Datadog) agrupa por correlation_id.
- **"Muitos estados, máquina difícil de acompanhar"**: isso é
  sintoma de granularidade errada ou de estados `*-ING`
  redundantes. Cinco estados "READY_TO_X" + quatro `*-ING` +
  COMPLETED/FAILED está bom. Vinte estados sugere que você tem
  duas máquinas sobrepostas (ex.: crawl + processamento + export
  separados em 3 bounded contexts).

## Concrete Example

Argos — Lambda Chain de processamento de conteúdo.

```
target_state (Dynamo)
  target_id (PK)
  status       PENDING | CRAWLING | READY_TO_CLEAN |
               CLEANING | READY_TO_CLASSIFY | CLASSIFYING |
               READY_TO_EXTRACT | EXTRACTING |
               READY_TO_ROUTE | ROUTING |
               READY_TO_NOTIFY | NOTIFYING |
               COMPLETED | FAILED
  updated_at   timestamp
  cleaned_at   timestamp?
  classified_at timestamp?
  extracted_at timestamp?
  classification CRITICO | INFORMATIVO | IRRELEVANTE ?
  error_reason string?
  error_at     timestamp?

  GSI (esparso): status = FAILED → query "quantos falharam hoje"
```

Fluxo: Cleaner Lambda é assinante do Stream com filtro
`NewImage.status = READY_TO_CLEAN`. Lê o item, faz o trabalho
(limpa conteúdo), escreve `status = READY_TO_CLASSIFY` + `cleaned_at
= now()`. Stream dispara Classifier. E por diante até NOTIFYING →
COMPLETED — e o Notifier, além de escrever COMPLETED, publica o
evento **público** `ContentProcessed` no EventBridge `argos.platform`
(único evento externo de todo o pipeline).

Reprocessar todos os editais de gran-cursos de ontem que falharam
na classificação? Query no GSI `status=FAILED` +
`error_reason LIKE 'classifier%' + updated_at > ontem` → batch
update `status = READY_TO_CLASSIFY`. Stream dispara. Zero deploy,
zero código novo.

## Sources

- `[[docs/rules/Dual-Store Architecture.md]]` — Argos `target_state`
  como hot state Dynamo.
- `[[docs/rules/Event-Driven Decoupling.md]]` — seção "Argos —
  Dentro da pipeline: coreografia via estado", que explicita o
  padrão Stream-como-trigger e o motivo de EventBridge ficar
  reservado para contrato público.
- Destilado das regras de Danilo — Lambda Chain do Argos v3.

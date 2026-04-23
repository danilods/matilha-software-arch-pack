---
name: swarch-ticker-vs-rule-per-entity
description: "Use when agendando N alvos (crawlers, jobs periódicos, pollers por entidade) e alguém propõe uma regra EventBridge/cron por alvo — força padrão Ticker com GSI sparse que escala pra milhões antes de esgotar limites AWS."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando o desenho de agendamento está prestes a criar uma regra
de cron por entidade — padrão que escala mal e esgota limites AWS.
Sinais típicos:

- "Como agendo N alvos com frequências diferentes?"
- "Cada alvo tem sua própria EventBridge rule?"
- "Vou criar uma regra cron por cliente/URL/job."
- "Tô criando schedule via API toda vez que um usuário cadastra um
  novo crawler."
- "Temos 50k entidades, cada uma com horário próprio."
- Desenho envolve `PutRule` / `CreateSchedule` como ação normal de
  admin (não só bootstrap de infra).

A regra de fundo: **GSI-BASED POLLING (TICKER) ESCALA PRA MILHÕES;
RULE-PER-ENTITY ESGOTA LIMITES AWS.** Uma única regra dispara uma
Lambda Ticker periodicamente que consulta um GSI sparse de "quem está
due" e enfileira trabalho.

## Preconditions

- Você tem N entidades com agendamento individual (cron, frequência
  variável, next_run_at). N ≥ ~100 já começa a justificar o padrão; N
  ≥ 10k é obrigatório.
- Cada entidade pode ser representada como linha num store indexado
  (DynamoDB, Postgres com índice) com campos `active` e `next_run_at`
  (ou equivalente: `next_due`, `cron_at`, `scheduled_for`).
- A resolução de segundos **não** é requisito duro. Se o domínio
  aceita 1 minuto de jitter no agendamento, Ticker aplica. Se exige
  precisão sub-segundo, escolha outra estratégia (mas confirme o
  requisito; geralmente é relaxável).
- O runtime de processamento não consegue (e não deve) ser chamado
  via API síncrona por burst. Workers precisam fazer pull de fila.

## Execution Workflow

1. Nomeie o problema em números: quantas entidades agora, quantas no
   próximo ano, qual frequência média, qual latência aceitável entre
   "hora chegou" e "trabalho começou". Sem números, você está
   adivinhando.
2. Descarte explicitamente a opção *uma regra por entidade*:
   - EventBridge tem limite de regras por região (na casa de milhares;
     varia por conta).
   - Cada `PutRule`/`DeleteRule` é chamada API custosa; admin operando
     entidades vira orquestração de infra.
   - Visibilidade fica ruim ("quais estão ativos?" = listagem
     paginada de regras).
3. Desenhe o Ticker:
   - Uma regra EventBridge (ou scheduler) com cron fixo (normalmente
     `*/1 min` — 1 invocação por minuto).
   - Lambda Ticker que consulta GSI sparse: `active=true AND
     next_run_at <= now`, ordenado por `next_run_at`.
   - Ticker publica batch em fila SQS (ou stream) com `DelaySeconds`
     aleatório pra achatar pico (ex.: 0-30 min).
   - Workers (Fargate, Lambda, ECS) fazem pull da fila no ritmo que
     aguentam — independente do pico de "due".
4. Desenhe o GSI corretamente:
   - Partition key: `status_bucket` (ex.: `"ACTIVE"`) ou similar —
     sparse index, só linhas com `active=true`.
   - Sort key: `next_run_at` (ISO timestamp ou epoch).
   - Query: `status_bucket = "ACTIVE" AND next_run_at <= now`, limit
     batch size (tipicamente 1000-5000 por invocação).
   - Atualização após agendar: worker (ou Ticker) atualiza `next_run_at`
     pro próximo horário, mantendo a linha na GSI pro próximo ciclo.
5. Jitter é feature, não bug: 10k URLs "due" às 08:00 não são
   processadas às 08:00:01 — são espalhadas em 5-30 min com
   `DelaySeconds` aleatório. Achata pico de conexões TCP, evita DDoS
   acidental em alvos, distribui carga nos workers.
6. Documente trade-offs aceitos: resolução de 1 min (vs segundos),
   possível atraso na cauda (se batch size < alvos due), failover via
   re-queue se worker falha (idempotência é obrigatória).

## Rules: Do

- Uma regra EventBridge fixa + Ticker Lambda + GSI sparse + fila SQS +
  workers pulling. Esse é o padrão de referência pra N entidades
  agendadas.
- Defina `next_run_at` como timestamp absoluto (não cron expression
  por linha). Cron é decisão do negócio; timestamp é o que o Ticker
  consulta.
- Use GSI sparse (`active=true` como partition key + `next_run_at`
  sort key). Sparse index é barato: só indexa linhas ativas.
- Adicione jitter (`DelaySeconds` aleatório) pra achatar pico de
  trabalho simultâneo. Funciona mesmo com fila Standard.
- Idempotência nos workers. Re-entrega de trabalho é esperada no
  padrão pull (visibility timeout, retry, DLQ).
- Admin UI muta linha no store (update de `next_run_at`, toggle
  `active`). Zero chamada de API AWS pra schedule.

## Rules: Don't

- Não crie regra EventBridge por entidade. Vai esgotar limite da conta
  antes da escala que você quer.
- Não use `CreateSchedule` por entidade (EventBridge Scheduler) como
  ação de admin rotineira. Mesmo problema em nova API.
- Não orquestre escala via `Lambda → ecs:RunTask` em loop a partir
  do Ticker. Esse é o anti-padrão v1 do Argos (esgotou TPS da API
  ECS em minutos). Ticker publica em fila, workers fazem pull.
- Não deixe Ticker processar 10k+ linhas em uma invocação única.
  Batch size controlado (1000-5000), paginação se necessário, ou
  múltiplos Tickers com buckets.
- Não exija resolução de segundos sem evidência de negócio. 99% dos
  casos aceita 1 minuto e você economiza complexidade real.
- Não publique batch inteiro pra SQS de uma vez sem jitter — vai
  chegar a fila num só momento e virar pico.

## Expected Behavior

Após aplicar a skill, o desenho tem uma regra fixa EventBridge, um
Ticker que consulta GSI, uma fila SQS, e workers pulling. Admin opera
entidades mutando linhas no DynamoDB/Postgres. Adicionar/remover/
repriorizar é update de linha — zero API AWS custosa.

Escala esperada: milhões de entidades aguentadas pelo DynamoDB
on-demand; limite real vira o throughput dos workers (que você escala
horizontalmente). Custo operacional: admin UI fala com DB, não com
AWS console.

Resultado de longo prazo: a conta não explode por limites regionais
de EventBridge, admin UI vira trivial (CRUD em DB), e
observabilidade centraliza no Ticker (uma Lambda com logs
consolidados) e na fila (métricas SQS nativas).

## Quality Gates

- Existe **uma** regra EventBridge (ou scheduler) pra disparar o
  Ticker. N extra regras por entidade = reprove.
- GSI (ou índice equivalente) indexado por `active` + `next_run_at`
  (sparse). Query do Ticker usa esse índice, não scan.
- Fila SQS entre Ticker e workers. Ticker não invoca trabalho
  diretamente (não chama `ecs:RunTask`, `lambda:Invoke` em loop).
- Workers idempotentes, com visibility timeout, retry e DLQ
  configurados.
- Jitter explícito (`DelaySeconds` aleatório ou equivalente) no
  publish do batch pra achatar pico.
- Admin UI (se houver) muta linhas no store, não chama API de
  schedule da AWS.

## Companion Integration

**Complementa `matilha-sysdesign-pack:sysdesign-scalability-horizontal-vs-vertical`**
— sysdesign cobre horizontal/vertical/stateful em termos estruturais;
esta skill traz o case concreto do Argos (Ticker GSI com números
reais de falha: `ecs:RunTask` esgotando TPS em minutos, 200k URLs
agendadas por uma regra fixa). Use sysdesign pra orientar a decisão
geral de escala; volte aqui pra desenhar Ticker especificamente.

**Pareia com `swarch-pull-over-push-orchestration`** (Family 5). São
lados da mesma moeda: Ticker é como *gerar* trabalho pros workers;
pull-over-push é como *consumir* esse trabalho sem esgotar APIs. Sempre
vêm juntos no desenho.

**Pareia com `swarch-measure-before-scale`** — Ticker é a resposta
certa pra N entidades agendadas, mas só se o problema é de fato
"como agendo N"; se N é pequeno (<100) e a dor é outra, medir antes
evita aplicar o padrão onde não pede.

**Pareia com `swarch-ordering-decision`** (Family 2) — Ticker + SQS
Standard é default; só use FIFO/Kinesis se a ordem entre alvos
**importa** pro domínio (quase sempre não importa).

## Output Artifacts

- Diagrama do fluxo: EventBridge → Ticker Lambda → GSI query → SQS
  (com jitter) → Workers pool.
- Schema do store com índice GSI nomeado, partition/sort keys
  definidas, e regra de atualização de `next_run_at`.
- Documento de trade-offs: resolução aceita, batch size, jitter
  range, latência esperada de agendamento.
- Código/CDK do Ticker e da regra EventBridge fixa.
- Opcional: dashboard com métricas (SQS lag, Ticker duration, batch
  size por invocação, `next_run_at` atraso médio).

## Example Constraint Language

- Use "must" para: uma única regra EventBridge (não N); GSI sparse
  com `active` + `next_run_at`; fila entre Ticker e workers.
- Use "should" para: batch size 1000-5000; jitter via `DelaySeconds`;
  resolução de 1 minuto (confirmar se domínio aceita, quase sempre
  aceita).
- Use "may" para: escolha de DynamoDB vs Postgres como store; exato
  range de jitter (5/15/30 min); nome do campo (`next_run_at`,
  `next_due`, `scheduled_for`).

## Troubleshooting

- **"Mas preciso de resolução de segundos"**: confirme com o negócio.
  "Edital publicado" detectado em 10s vs 60s faz diferença real? 99%
  das vezes não. Se faz (alertas de emergência, sistemas financeiros
  intraday), use abordagem diferente (stream + state machine). Não
  tente "Ticker a cada 5s" — vira custo sem benefício.
- **"EventBridge Scheduler (o novo) resolveria isso"**: EventBridge
  Scheduler escala melhor que EventBridge Rules pra muitos schedules
  (~milhões), então o gargalo de limite some. Mas operações por
  entidade ainda são API calls custosas, e a visibilidade via AWS
  console some pra qualquer escala séria. Ticker + store próprio
  mantém controle operacional (admin UI CRUD), mesmo quando o limite
  AWS não é mais o gargalo.
- **"A fila tá explodindo, workers não aguentam"**: bom — significa
  que a escala real chegou. Horizontalize workers (adicione
  capacity), aumente visibility timeout se necessário. O Ticker tá
  fazendo o trabalho dele (achatar spike), o problema é throughput
  do worker pool — decisão separada.
- **"Alvos due perdem horário quando batch size < due"**: aumenta
  batch size, ou adiciona paginação no Ticker (processa em múltiplos
  passes dentro da mesma invocação até vazio), ou divide em múltiplos
  Tickers por bucket (hash de target_id).
- **"Como faço admin pausar 1 alvo?"**: update de linha: `active =
  false`. Ticker deixa de ver na GSI sparse. Reativar: `active =
  true`. Zero API AWS.

## Concrete Example

Argos — problema: 200k URLs × 3 checks/dia = 600k "disparos"/dia.

**Descartado:** uma regra EventBridge por URL. 200k regras por região,
limite da conta AWS. Operação de criar/atualizar URL via `PutRule`
custosa; admin UI virava orquestração de infra AWS.

**Adotado:** Ticker pattern.

```
EventBridge (cron: */1 min)
    ↓ (uma invocação/min)
Lambda Ticker
    ↓ (Query GSI: active=true AND next_run_at <= now, limit 1000)
DynamoDB GSI sparse
    ↓ (batch de ~1000 alvos due)
Lambda Ticker
    ↓ (publica em SQS com DelaySeconds aleatório 0-30min)
SQS Crawl Queue
    ↓ (pull pelos workers)
Fargate Worker Pool (auto-scaled)
```

Trade-offs aceitos:

- Resolução de 1 min. Ninguém notifica edital em <1 min de atraso.
- Jitter de 0-30 min achata pico: 1000 URLs due às 08:00 não são
  processadas todas às 08:00:01; são espalhadas — evita DDoS
  acidental nos sites alvo.
- GSI sparse: só indexa `active=true`. Barato e rápido.
- Escala real: milhões de targets cabem (DynamoDB on-demand +
  workers horizontais).
- Admin opera `next_run_at` e `active` via DynamoDB — UI responde
  instantaneamente, zero spike de API AWS.

Anti-padrão v1 evitado: `Lambda → ecs:RunTask` em loop pra disparar
worker por target. Esgotava TPS da API ECS em minutos em carga real.
Ticker + pull inverte: API AWS chamada só na taxa que workers
processam, nunca em burst.

## Sources

- [[docs/rules/Escalabilidade sem Prematuridade]]
- Síntese direta da seção Ticker da regra do Danilo. Preserva o
  problema real (200k URLs × 3/dia = 600k disparos), o descarte da
  Opção A (rule-per-target), o desenho Opção B com GSI sparse +
  jitter + workers pulling, e os trade-offs conscientes (resolução
  1 min, escala pra milhões, admin UI sem friction).

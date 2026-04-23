---
name: swarch-projection-rebuild-discipline
description: "Use when uma projeção derivada vai pra produção — exige watermark/lag metric e procedimento de rebuild documentado antes de ir ao ar."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando uma projeção derivada (admin DB projetado do hot
state, índice de busca populado por events, denormalização pra
relatório) vai entrar em produção. Também fira quando alguém
pergunta "e se a projeção divergir?", "como reconstituo o admin
DB?", "a projeção tá com quanto de atraso?" — sinais de que a
disciplina não foi instalada antes. Este skill é sobre **prontidão
operacional**: watermark, lag, DLQ, rebuild. Decidir que existe
projeção (skill 8), como sincronizá-la (skill 9), e como o SoT se
organiza (skill 10) vêm antes.

## Preconditions

- Existe projeção derivada de um SoT, via CDC, outbox ou Stream
  trigger (output dos skills 8, 9, 10).
- Time consegue publicar métricas customizadas (CloudWatch,
  Datadog, Prometheus). Sem isso, watermark é texto em log — não
  serve.
- Existe local para runbook versionado (repo, Notion, wiki). Runbook
  informal na memória do engenheiro sênior não conta.
- SoT tem **retenção suficiente** do change stream para rebuild:
  DynamoDB Streams 24h (curto — pode exigir backfill do próprio
  table), Kinesis 24h-365d, Kafka configurável, Debezium + Kafka
  idem. Se retenção é 24h e rebuild leva 6h, margem é confortável;
  se retenção é 24h e rebuild leva 20h, você tem problema estrutural.

## Execution Workflow

1. **Defina watermark.** Métrica numérica que responde "quantos
   segundos de atraso a projeção carrega em relação ao SoT". Em
   DDB Streams: `ApproximateAgeOfOldestRecord`. Em Kafka:
   `consumer_lag`. Em outbox: `now() - min(created_at) WHERE
   published_at IS NULL`. Publique como métrica.
2. **Defina SLO de lag.** "p99 < 10s em horário de trabalho, p99 <
   60s durante jobs noturnos" — número concreto. Abaixo do SLO:
   verde. Acima: alarme. Muito acima: incidente.
3. **Escreva o procedimento de rebuild** ANTES de produção. Não
   "depois a gente vê". Três modos:
   - **Replay do stream**: se o SoT tem retenção suficiente, rebobina
     o consumer (reset offset / trim horizon) e reprocessa tudo.
     Idempotência do consumer é pré-requisito (ver skill 9).
   - **Rebuild do SoT direto**: lê o SoT item a item (scan
     paginado com parallelism) e repopula a projeção. Mais lento
     mas não depende de retenção de stream.
   - **Híbrido**: snapshot do SoT + replay do stream a partir do
     snapshot_at. Padrão quando dataset é grande e stream retém
     pouco.
4. **Estime tempo de rebuild.** 200k itens, upsert 100ms = 5-6h
   single-threaded; com 10 workers paralelos, 30-40min. Escreva o
   número. "Deve ser rápido" não ajuda às 3h da manhã.
5. **Teste o rebuild em staging.** Pelo menos uma vez, derrube a
   projeção em staging, rode o procedimento documentado, meça o
   tempo, valide idempotência. Descobrir que o rebuild tem bug
   durante incidente de produção é garantia de incidente longo.
6. **Configure alarmes acionáveis.** Lag > SLO por N minutos →
   ticket. Lag > 10× SLO → pagerduty. DLQ com mensagens não
   reprocessadas > N → ticket. Cada alarme tem runbook linkado.

## Rules: Do

- Publique watermark como métrica **no dia zero** da projeção,
  não "quando sobrar tempo". Sem watermark, você não sabe quando
  divergiu.
- Escreva runbook de rebuild **antes** de produção. O exercício
  de escrever revela gaps (retenção insuficiente, consumer não
  idempotente, dependência de campo que não está no stream).
- Teste rebuild em staging regularmente (trimestral). Rebuild
  quebra silenciosamente quando schema do SoT muda — só descobre
  rodando.
- Monitore DLQ como cidadão de primeira classe. Mensagens acumulando
  na DLQ é divergência silenciosa em progresso. Alarme por volume
  + idade.
- Inclua rebuild no DR plan do bounded context. Projeção perdida
  sem plano de rebuild é incidente que vira dias; com plano, é
  horas.

## Rules: Don't

- Não coloque projeção em produção sem watermark. É equivalente a
  voar sem altímetro.
- Não aceite "depois a gente documenta o rebuild". Não existe
  depois — existe 3h da manhã improvisando, no meio de incidente,
  sob pressão.
- Não publique eventos públicos derivados da projeção. Consumers
  downstream passam a depender de dados derivados — propagação de
  erros de sincronização. Eventos saem do SoT (via outbox ou log).
- Não deixe DLQ crescer sem alarme. Mensagens que não processam
  silenciosamente não somem — elas acumulam e na hora do rebuild
  explodem.
- Não dependa de retenção de stream menor que o tempo de rebuild.
  Se Streams retêm 24h e rebuild leva 20h, qualquer atraso e você
  perde tail — precisa de backfill do SoT.

## Expected Behavior

Depois do skill, projeção em produção tem dashboard com watermark
visível, alarmes acionáveis, runbook de rebuild testado e DLQ
monitorada. A pergunta "e se divergir?" tem resposta em 30s via
runbook, não improviso.

Cultura da equipe muda: projeção vira infraestrutura operacional
(monitoramento, runbook, teste regular), não código que "funciona
desde o deploy e nunca foi olhado". Divergência passa a ser evento
detectável em minutos em vez de "descoberto por reclamação de
cliente semanas depois".

## Quality Gates

- Watermark publicado como métrica + visível em dashboard.
- SLO de lag definido com número concreto + alarme quando violado.
- Runbook de rebuild versionado (repo/Notion), com tempo estimado.
- Rebuild testado em staging **ao menos uma vez** antes de produção.
- DLQ com alarme por volume + idade.
- Retenção do stream ≥ 2× tempo estimado de rebuild (margem).

## Companion Integration

- **Continua** `swarch-cdc-over-dual-write`: aquele define o
  mecanismo + idempotência + DLQ básica; este adiciona watermark,
  SLO, rebuild documentado, teste periódico.
- **Depende de** `swarch-dual-store-source-of-truth` e
  `swarch-hot-state-via-status-machine`: a projeção só faz sentido
  quando SoT e mecanismo de stream estão claros.
- **Alimenta** `swarch-event-schema-contract`: se a projeção
  consome eventos externos, mudanças de schema exigem plano de
  rebuild que contempla versões antigas.
- Metodologia: fase 40 (implementação) + fase 50 (review) —
  checklist de readiness operacional antes do go-live.

## Output Artifacts

- Dashboard com watermark + SLO claramente marcado.
- Alarme configurado (CloudWatch/Datadog) com destino (PagerDuty,
  Slack, email).
- `runbooks/rebuild-<projection-name>.md`: procedimento passo a
  passo, comandos, tempo estimado, pré-requisitos.
- Resultado do teste de rebuild em staging (log + tempo).
- Entrada no DR plan do bounded context.

## Example Constraint Language

- Use "deve" para: watermark publicado antes de produção, SLO
  numérico, runbook versionado, teste de rebuild em staging.
- Use "pode" para: escolher replay vs rebuild direto vs híbrido
  conforme substrato, publicar watermark no CloudWatch ou Datadog,
  rodar teste trimestral ou semestral conforme criticidade.
- Use "nunca" para: projeção em prod sem watermark, rebuild
  improvisado em incidente, publicar evento público a partir da
  projeção em vez do SoT, aceitar retenção de stream menor que
  tempo de rebuild.

## Troubleshooting

- **"Watermark está sempre em zero, parece que não mede"**:
  verifique a fonte. Em DDB Streams, use
  `ApproximateAgeOfOldestRecord` (não `IteratorAge` da Lambda, que
  é outra coisa). Em outbox, garanta que mede
  `min(created_at) WHERE published_at IS NULL`, não linha mais
  recente.
- **"Rebuild em staging levou 18h, produção tem mais dados — vai
  explodir"**: aumente paralelismo (mais workers, mais partições)
  ou mude para rebuild incremental a partir de snapshot. Se o
  tempo continua alto, a projeção tem complexidade indevida —
  considere reduzir escopo da projeção.
- **"DLQ cresce mas não consigo reprocessar — mensagens quebradas"**:
  comum quando schema do SoT mudou e consumer não tolera shape
  antigo. Dois caminhos: (1) patch no consumer para tolerar
  versões antigas + reprocessa; (2) skip e rebuild total. Decida
  conforme volume da DLQ.
- **"SLO está sendo violado em horário de job noturno, mas não é
  problema real"**: SLO precisa de dois níveis — horário de
  trabalho (apertado) + horário de job (relaxado). Não mascarar
  alarme; diferenciá-lo.

## Concrete Example

Argos — projeção `target_state_projection` em PG derivada do
Dynamo `target_state` via Streams.

Antes de go-live:
- Watermark: Lambda projetora publica métrica
  `projection.lag_seconds` baseada no `ApproximateCreationDateTime`
  do Stream record vs `now()`. Dashboard exibe p50 / p99.
- SLO: p99 < 5s em horário comercial, p99 < 60s durante batches
  noturnos de reprocessamento.
- Alarme: p99 > 30s por 5min → ticket. p99 > 120s por 5min →
  PagerDuty.
- Runbook `runbooks/rebuild-target-state-projection.md`:
  procedimento de scan paginado do Dynamo + UPSERT idempotente no
  PG. Tempo estimado: 30min para 200k entidades com 10 workers
  paralelos. Comandos exatos, variáveis de ambiente, como validar
  pós-rebuild (count(*) batendo com `describe-table` ItemCount).
- Teste trimestral em staging: último rodou em 27min; consumidor
  idempotente validado (rodado 2×, resultado idêntico).
- DLQ: SQS com alarme por >10 mensagens ou idade >1h.

Primeiro incidente real em 18 meses: schema do Dynamo ganhou campo
novo; projetora antiga não sabia dele, DLQ começou a encher. Alarme
disparou em 12 minutos. Patch no consumer + replay da DLQ: zero
divergência final. Sem o skill: descoberta via reclamação de admin
UI dias depois.

## Sources

- `[[docs/rules/Dual-Store Architecture.md]]` — seção "Você tem
  inconsistência iminente se" (watermark, rebuild, eventos da
  projeção) + teste de fumaça.
- Destilado das regras de Danilo — Argos v3 operando a projeção
  `target_state_projection` em PG com rebuild testado trimestralmente.

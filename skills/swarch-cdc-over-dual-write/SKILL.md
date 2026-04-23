---
name: swarch-cdc-over-dual-write
description: "Use when precisa sincronizar dois stores (Postgres + Dynamo, PG + ClickHouse, etc.) e alguém propõe escrever nos dois na aplicação — CDC ou outbox é quase sempre preferível a dual-write síncrono."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara depois que `swarch-dual-store-source-of-truth` já decidiu que
dual-store se justifica e nomeou o SoT — a pergunta agora é **como**
manter os dois sincronizados. Fira especificamente quando aparece
"a aplicação escreve nos dois", "chamo `pg.insert()` e `ddb.put()`
em sequência", "coloco os dois writes dentro de um try/except", ou
qualquer variação de dual-write síncrono na camada de aplicação. Este
skill é sobre o **mecanismo de sincronização**; não trate aqui
decisão de "preciso de dois bancos?" (skill 8) nem como reconstruir
projeção divergente (skill 11).

## Preconditions

- Existe SoT nomeado e projeção derivada (output de skill 8). Sem
  isso, não dá pra falar de CDC — você não sabe quem é origem.
- O SoT tem mecanismo de log/change capture nativo ou acessível:
  DynamoDB Streams, Postgres logical replication (WAL), MySQL
  binlog, Mongo change streams, Kafka como próprio store. Se o SoT
  não expõe changes, outbox é o caminho.
- Time consegue operar um processo adicional (Lambda, worker,
  Debezium, KCL consumer). CDC move complexidade de "dois writes
  síncronos" para "um write + um processo lendo log" — é trade-off,
  não magia.
- Consumidores da projeção toleram **alguns segundos** de lag. Se a
  projeção precisa estar consistente em <100ms com o SoT, CDC não
  serve — repense a arquitetura.

## Execution Workflow

1. **Descarte dual-write síncrono como default.** A menos que já
   haja justificativa escrita do motivo de não usar CDC/outbox,
   dual-write é anti-padrão. Ponto de partida da conversa.
2. **Escolha o mecanismo** conforme o SoT:
   - **DynamoDB Streams → Lambda → projeção**: default para DDB como
     SoT.
   - **Postgres logical replication (pgoutput) ou Debezium**: default
     para PG como SoT + projeção em sistema externo (ES, ClickHouse,
     DDB).
   - **Outbox pattern** (tabela `outbox` na mesma transação + worker
     lendo): quando não há CDC nativo bom ou quando o evento precisa
     de enriquecimento que o log do banco não carrega.
   - **Mongo change streams**: equivalente do Streams para Mongo.
3. **Garanta idempotência no consumer.** CDC entrega at-least-once.
   O processador da projeção precisa tolerar ver o mesmo change 2x,
   3x, N vezes. Use `ON CONFLICT DO UPDATE`, UPSERT, ou versioning
   com `if last_modified > current`.
4. **Defina DLQ + retry.** Falha processando um change não pode
   parar o stream inteiro. Mensagem problemática vai pra DLQ, stream
   continua. Alertas quando DLQ tem volume > N.
5. **Meça lag.** Watermark entre SoT e projeção publicado como
   métrica (segundos de atraso). Sem watermark, você não sabe quando
   divergiu. Ver `swarch-projection-rebuild-discipline` para a
   disciplina operacional completa.
6. **Outbox explicitamente quando CDC é ruim/caro.** Se o time não
   quer operar Debezium e o SoT é Postgres, outbox é simpler e
   seguro: tabela `outbox` escrita na **mesma transação** do
   domínio, worker separado publica e deleta.

## Rules: Do

- Trate CDC / outbox como default. Dual-write síncrono só entra
  como exceção com ADR escrito nomeando o motivo e as salvaguardas
  (idempotência em ambos os lados, reconciliação, janela tolerada
  de divergência).
- Escreva o consumer idempotente desde o dia zero. CDC + consumer
  não idempotente = projeção divergente na primeira falha.
- Instrumente lag com métrica publicada. "Não sei quantos segundos
  de atraso estamos" é sinônimo de "não sei se estamos quebrados".
- Use DLQ + alarme. Mensagem ruim na ponta nunca deve travar o
  stream — mas nunca deve ser silenciosamente descartada também.
- Outbox quando o evento precisa dizer mais do que o log do banco
  carrega (ex.: evento de domínio rico, não CRUD cru). O log do
  banco dá estado; outbox dá **intenção** publicada.

## Rules: Don't

- Não faça dual-write síncrono sem ADR. Os modos de falha são
  previsíveis e caros: write 1 comita, write 2 falha, estado
  inconsistente, reconciliação manual às 3h da manhã.
- Não pule idempotência. "Vou garantir que só entrega uma vez" não
  existe em CDC — a garantia é at-least-once. Exactly-once é
  mito operacional.
- Não ignore DLQ. Mensagem que o consumer não processa não pode
  simplesmente sumir. Vá pra DLQ, dispare alerta, investigue.
- Não publique eventos da **projeção** para consumers downstream.
  Eventos saem do SoT (via outbox) ou do log do SoT. Projeção
  publicando evento = downstream dependente de dados derivados =
  propagação de erros de sincronização.
- Não resolva dual-write adicionando try/except com retry manual.
  Isso é outbox mal implementado sem as garantias. Se vai fazer,
  faça direito.

## Expected Behavior

Depois do skill, toda sincronização entre stores é via log (CDC)
ou outbox; dual-write síncrono some do vocabulário da equipe. A
projeção tem watermark visível, DLQ configurada, consumer
idempotente — e falha na projeção é evento monitorável, não bug
silencioso.

A discussão passa de "e se falhar o segundo write?" (que não tem
boa resposta) para "quanto lag é aceitável, qual DLQ, qual plano
de rebuild?" (que têm respostas operacionais).

## Quality Gates

- Mecanismo de sync documentado (CDC nativo, Debezium, outbox) —
  sem "aplicação escreve nos dois".
- Consumer da projeção é **idempotente** (teste unitário que
  processa o mesmo change 3x e verifica resultado idêntico).
- Métrica de lag publicada e visível em dashboard.
- DLQ configurada com alarme por volume/idade.
- ADR escrito se dual-write síncrono for adotado (exceção
  justificada).

## Companion Integration

- **Depende de** `swarch-dual-store-source-of-truth`: este skill só
  faz sentido depois que SoT está nomeado.
- **Complementa** `matilha-sysdesign-pack:sysdesign-dual-write-event-sourcing`:
  aquele skill descreve dual-write como anti-padrão e apresenta
  event sourcing / CDC / saga como alternativas; este aterra o CDC
  / outbox no caso Argos-like (DynamoDB Streams → Lambda → PG
  projection, ou outbox em PG → EventBridge).
- **Alimenta** `swarch-projection-rebuild-discipline`: watermark + DLQ
  + rebuild são o ciclo completo.
- **Dialoga com** `swarch-event-schema-contract`: se a projeção
  emite evento público derivado do change, o schema desse evento é
  contrato — mesmas regras se aplicam.
- Metodologia: fase 30 (arquitetura) e fase 40 (implementação).

## Output Artifacts

- Documento curto (pode ser parte do ADR de dual-store): mecanismo
  de sync escolhido + por quê.
- Código do consumer idempotente + teste que demonstra
  idempotência.
- Métrica/dashboard de lag.
- Configuração de DLQ + alerta.
- Runbook: "DLQ enchendo — diagnosticar" (1-2 páginas).

## Example Constraint Language

- Use "deve" para: idempotência no consumer, DLQ configurada,
  métrica de lag publicada, ADR quando for dual-write.
- Use "pode" para: escolher entre Debezium vs logical replication
  direta, outbox com Lambda poller vs worker dedicado, DDB Streams
  vs Kinesis Data Streams conforme perfil de consumo.
- Use "nunca" para: dual-write sem ADR, consumer não idempotente,
  publicar evento público a partir da projeção em vez do SoT.

## Troubleshooting

- **"DDB Streams perde eventos quando Lambda falha muito"**: Streams
  retêm 24h. Se o consumer fica mais de 24h quebrado, você perde
  tail. Mitigação: DLQ para mensagens problemáticas (não parar a
  posição do shard) + alerta para consumer down > N min.
- **"Outbox worker ficou lendo a mesma linha e publicando
  duplicado"**: falta `SELECT ... FOR UPDATE SKIP LOCKED` ou
  equivalente. Worker deve pegar linhas, publicar, marcar como
  publicada numa transação. Sem isso, duas instâncias publicam
  o mesmo evento.
- **"Debezium é overhead demais pro time"**: válido. Outbox é
  alternativa mais simples se SoT é Postgres. O custo é uma tabela
  a mais + um worker. Se a equipe não opera Kafka Connect, outbox
  é a escolha honesta.
- **"Precisamos de exactly-once entrega"**: exactly-once não existe
  em sistemas distribuídos reais — existe at-least-once + consumer
  idempotente, que o usuário percebe como exactly-once. Aceite isso
  ou repense a arquitetura.

## Concrete Example

Argos v3. SoT de hot state = Dynamo (`target_state`). Admin UI
precisa listar status com filtro composto, então existe
`target_state_projection` no Postgres.

Tentação: cada Lambda da chain faz `ddb.update_item(...)` seguido
de `pg.execute(UPDATE ...)`. Consequência previsível: segundo write
falha ocasionalmente, projeção fica defasada, admin UI mostra
status errado, time descobre por reclamação de cliente.

Solução adotada: Lambda escreve **apenas no Dynamo**. DynamoDB
Stream dispara uma Lambda projetora que lê o change, aplica UPSERT
idempotente no `target_state_projection` em PG
(`INSERT ... ON CONFLICT (target_id) DO UPDATE SET status = EXCLUDED.status
WHERE target_state_projection.updated_at < EXCLUDED.updated_at`).
Lag publicado como métrica (`projection.lag_seconds`, p99 < 5s).
DLQ configurada — se o processador falha 3x na mesma mensagem, vai
para SQS DLQ + alarme. Procedimento de rebuild: ler item por item
do Dynamo e upserting no PG (~20 min para 200k entidades). Rodou
duas vezes em 18 meses; nunca precisou intervenção manual.

## Sources

- `[[docs/rules/Dual-Store Architecture.md]]` — seção "CDC >
  dual-write (quase sempre)" + Argos Postgres-SoT/Dynamo-hot-state
  + contraponto Fluency.
- Destilado das regras de Danilo a partir de CDC via DynamoDB
  Streams no Argos e dual-write justificado do Fluency (Cognee +
  PG).

---
name: swarch-dual-store-source-of-truth
description: "Use when alguém sugere usar Postgres + DynamoDB (ou dois bancos em geral) — decide se dual-store se justifica e qual é Source of Truth."
category: swarch
version: "1.0.0"
requires: []
optional_companions: ["matilha-sysdesign-pack"]
---

## When this fires

Dispara quando aparece a pergunta "Postgres ou Dynamo?", "preciso dos
dois?", "o relacional tá sofrendo, jogo parte no NoSQL?", ou quando
alguém propõe arquitetura com dois bancos operacionais convivendo.
Também fira quando a discussão é "vou escrever nos dois pra ter backup"
— sinal de dual-store mal entendido. Este skill é sobre a **decisão**:
dual-store vale a pena? Qual é SoT? Os **mecanismos** (CDC vs
dual-write, status machine, rebuild de projeção) vivem em skills
irmãos — não redistille aqui.

## Preconditions

- Existe pressão real ou prevista para adicionar um segundo store.
  Se a conversa ainda é "qual banco uso?", é decisão de store único
  — não acione este skill.
- O time consegue descrever as cargas de leitura/escrita com ordem
  de grandeza (writes/s, tamanho do dataset, padrão de query). Sem
  isso, você não tem como qualificar "cargas qualitativamente
  diferentes".
- Há alguém operando o banco atual em produção. Dual-store com time
  que ainda está aprendendo o primeiro banco é receita de desastre.
- Existe disposição de marcar **um** banco como SoT. Se o time
  insiste em "ambos são verdade", pare e resolva isso antes — é o
  ponto central do skill.

## Execution Workflow

1. **Profile o banco atual antes de duplicar.** Meça a query lenta,
   olhe EXPLAIN, verifique índice, cheque se é burst real ou pico
   de relatório mal agendado. 70% dos pedidos de dual-store caem
   aqui e terminam em "ajustei o índice".
2. **Descreva as duas cargas em voz alta.** Uma frase cada:
   - Carga A: "admin UI lista targets com filtro composto, <10 reqs/s,
     latência 200ms aceita".
   - Carga B: "hot path escreve status de 600k entidades/dia com
     picos, latência <10ms, leitura por PK".
   Se as frases são parecidas, é um banco só. Se são qualitativamente
   diferentes (tipo de query, volume, latência, modelo de consistência),
   dual-store pode se justificar.
3. **Nomeie o Source of Truth.** Use a ordem:
   - Durabilidade/auditoria legal (quem tem log imutável, PITR forte).
   - Schema estável e bem conhecido.
   - Mutação complexa com invariantes multi-campo → quem suporta
     transações.
   Não existe "ambos". Não existe "depende". Se não consegue nomear
   o SoT, ainda não tem arquitetura.
4. **Nomeie a projeção.** O outro banco é derivado. Ele pode ser
   reconstruído do SoT (via replay, recomputação, CDC retry).
   Se o outro banco tem dados que o SoT não tem, ou você está
   usando dual-write sem saber, ou a classificação SoT/projeção
   está errada.
5. **Desenhe o fluxo de sincronização.** Quem atualiza o quê, via
   qual mecanismo. Detalhes ficam em `swarch-cdc-over-dual-write`,
   mas o mecanismo tem que estar nomeado aqui.
6. **Documente o teste de fumaça.** "Se o store secundário cair por
   1h, o que acontece?" — resposta deve ser "admin UI fica defasado,
   hot path segue" (ou equivalente). Se a resposta é "sistema para",
   ele é SoT também e você não tem dual-store, tem dependência
   síncrona mal desenhada.

## Rules: Do

- Profile antes. Sempre. A pergunta "Postgres tá lento" sem número
  em mão é chute — dual-store prematura custa 3-6 meses de
  complexidade que não volta.
- Qualifique as cargas como diferentes em **tipo**, não só em
  volume. Volume alto resolve-se com vertical scaling, read replica,
  cache. Tipo diferente (latência <10ms + burst 10k/s vs JOIN
  complexo diário) é o que justifica dois bancos.
- Nomeie SoT explicitamente num ADR. O ADR deve responder: "em caso
  de divergência entre os dois, qual banco ganha?". Se a resposta
  não sai fácil, repita o skill.
- Desenhe o caminho de rebuild da projeção no dia zero. Não "depois
  a gente vê". Ver `swarch-projection-rebuild-discipline`.
- Faça o teste de fumaça mental (e, quando possível, real em staging):
  derrube o secundário e veja se o sistema sobrevive como esperado.

## Rules: Don't

- Não adote dual-store "por performance" sem profile. Índice faltando
  é a causa em quase todos os casos. Dual-store não cura lentidão de
  query mal escrita.
- Não deixe ambos os bancos como ports de entrada. SoT tem uma porta
  de entrada única; projeção é read-only do ponto de vista do hot
  path.
- Não duplique dados sem propósito ("vou espelhar tudo pra ter
  backup"). Backup é backup (snapshot, PITR). Dual-store é para
  **cargas diferentes**. Misturar as duas coisas produz dois
  problemas.
- Não aceite "ambos são verdade". É sempre receita de bug permanente.
  Se o time insiste, provavelmente existem dois bounded contexts
  disfarçados de um — ver `swarch-context-by-vocabulary`.
- Não espere o time aprender dois bancos em paralelo com dual-store
  em produção. Dobra superfície operacional, dobra runbook, dobra
  oncall.

## Expected Behavior

Depois do skill, o time tem um ADR que responde três perguntas:
"precisamos de dois bancos?", "qual é o SoT?", "o que acontece se o
secundário cair?". A conversa de arquitetura deixa de ser "Postgres
ou Dynamo?" e passa a ser "qual carga cada um atende e qual é o SoT?".

Quando a resposta honesta é "um banco basta, só precisa de índice",
o skill economiza 3-6 meses de complexidade. Quando dual-store se
justifica, o time entra com arquitetura clara em vez de improviso.

## Quality Gates

- Profile documentado do banco atual antes da decisão (EXPLAIN,
  métricas, golden signals). Sem número, sem decisão.
- ADR nomeia SoT explicitamente e justifica (durabilidade, schema,
  mutação complexa).
- Cargas descritas como **qualitativamente diferentes** (tipo de
  query/volume/latência/modelo), não só "mais rápido".
- Caminho de rebuild da projeção existe no dia zero.
- Teste de fumaça ("secundário cai por 1h") tem resposta esperada
  documentada e idealmente testada em staging.

## Companion Integration

- **Complementa** `matilha-sysdesign-pack:sysdesign-dual-write-event-sourcing`
  — aquele skill trata dual-write como anti-padrão geral e apresenta
  três alternativas (event sourcing, CDC, saga). Este skill foca no
  caso específico Postgres-SoT + Dynamo-hot-state via CDC quando
  **é** a resposta certa, com os critérios para decidir isso.
- **Usa** `swarch-cdc-over-dual-write` para o **mecanismo** de
  sincronização (uma vez decidido que dual-store se justifica).
- **Usa** `swarch-hot-state-via-status-machine` para modelar o
  secundário quando ele é hot state tipo Dynamo.
- **Usa** `swarch-projection-rebuild-discipline` para o lado
  operacional (watermark, rebuild).
- Metodologia: fase 20 (spec) e 30 (arquitetura) — decisão fundamental.

## Output Artifacts

- ADR "Dual-store: sim/não e por quê" com profile, cargas, SoT
  nomeado, teste de fumaça.
- Diagrama simples das duas cargas (mesmo ASCII basta) mostrando
  direção de dados: hot path → SoT → CDC → projeção → read-only
  queries.
- Entrada no runbook: "se secundário cair, o que acontece, o que
  fazer".

## Example Constraint Language

- Use "deve" para: profile antes de decidir, nomear SoT num ADR,
  documentar teste de fumaça, desenhar rebuild no dia zero.
- Use "pode" para: escolher Postgres+Dynamo, Postgres+Redis,
  Postgres+ClickHouse conforme cargas — o padrão é geral, o
  substrato varia.
- Use "nunca" para: dois masters sem coordenação custosa, dual-store
  sem profile, "ambos são verdade".

## Troubleshooting

- **"Profilei e a query é mesmo complexa, mas não sei se dual-store
  resolve"**: teste primeiro read replica dedicada ao relatório ou
  cache materializado. Se a carga é só leitura pesada, isso resolve
  sem dual-store. Dual-store é para cargas de **escrita** qualitativa-
  mente diferentes.
- **"Time quer separar só pra isolar risco de downtime do banco
  principal"**: isolamento de risco é HA/replica, não dual-store.
  Duplicar schema não reduz risco — adiciona. Invista em backup,
  multi-AZ, failover.
- **"Preciso dos dois porque quero queryar com SQL e também com
  PK rápida"**: DynamoDB com boa partition key + projeção PG para
  SQL é o padrão. Mas se é a mesma entidade com o mesmo papel, o
  problema pode ser de modelagem, não de store.
- **"Cargas parecem diferentes mas o time não operou Dynamo ainda"**:
  decisão de store ≠ decisão de decidir hoje. Faça dual-store
  depois que o primeiro estiver sólido. Roadmap de 3-6 meses é
  aceitável; produção em dual-store com time verde não é.

## Concrete Example

Argos v3. Duas cargas:
- Admin UI: "listar targets do domínio gran-cursos, ativos, nunca
  processados com sucesso, ordenados por última tentativa" — JOIN
  composto, `WHERE ... AND ... ORDER BY`. ~5 req/s. Latência 300ms
  tolerada.
- Hot path: 200k URLs × 3 checks/dia = 600k writes/dia de status,
  com picos. Latência <10ms exigida. Leitura por PK.

Tentação inicial do time: "tudo no Dynamo, ele aguenta". Profile
mostra que a query composta vira múltiplas GSIs + client-side
aggregation — pesadelo operacional. Tentação oposta: "tudo em PG".
Profile do hot path mostra que escrita em burst de 10k/s em
Postgres single-writer exige tuning que o time não quer gerenciar.

Decisão: PG é SoT para configuração (targets, schedules, knowledge
base); Dynamo é hot state para status de ciclo de vida. ADR nomeia
PG como SoT por durabilidade (PITR + backup diário legalmente
exigível) e schema rigoroso. Sync: configuração PG → Dynamo via
processo de replicação; status Dynamo → projeção PG via CDC
(Streams → Lambda → upsert). Teste de fumaça: "Dynamo cai 1h" →
admin UI mostra status defasado; processo de crawl continua (lê
config do PG no início de cada run). "PG cai 1h" → admin UI morre,
hot path segue com config cacheada; volta sozinho. Dual-store
justificado.

## Sources

- `[[docs/rules/Dual-Store Architecture.md]]` — seção "Um é Source
  of Truth. O outro é derivado." + "Motivação tem que ser
  qualitativa, não quantitativa" + "Decisão: qual banco é Source
  of Truth?".
- Destilado das regras de Danilo a partir do Argos v3 (PG +
  Dynamo) e contraponto Fluency (Cognee + PG).

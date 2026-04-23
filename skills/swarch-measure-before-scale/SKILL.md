---
name: swarch-measure-before-scale
description: "Use when someone says 'esse endpoint tá lento', 'acho que precisamos escalar', 'vamos cachear', 'vamos migrar pra Dynamo' sem número medido ao lado — força EXPLAIN/profile/golden signals antes de horizontalizar, cachear, ou migrar."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando alguém propõe otimização, horizontalização, cache, ou
migração sem métrica concreta ao lado. Sinais típicos:

- "Esse endpoint tá lento."
- "Acho que precisamos escalar."
- "Vamos cachear isso."
- "Precisamos migrar pra DynamoDB, Postgres não escala."
- "Vamos shardar o banco."
- "Esse serviço precisa de mais réplicas."
- "Vamos pré-otimizar pra aguentar o crescimento."
- Agente propõe cache/sharding/migração sem antes medir.
- Ninguém consegue dizer o throughput atual, só "parece ruim".

A regra de fundo: **MEÇA ANTES DE ASSUMIR QUE "NÃO ESCALA". "NÃO
ESCALA" SEM NÚMERO AO LADO É CHUTE.** A maioria dos "problemas de
escala" é query mal escrita, índice faltando, timeout mal calibrado,
ou orquestração errada — não limite real de hardware.

## Preconditions

- Há uma proposta de mudança de escala/performance na mesa: cachear,
  horizontalizar, migrar store, sharding, adicionar réplicas, re-
  arquitetar.
- A mudança custa tempo, complexidade operacional, ou ambos. Se é
  trivial e óbvia (adicionar índice na query lenta), pode seguir —
  mas mesmo assim, medir antes confirma que resolveu.
- Existe acesso ao sistema em produção (ou staging com carga
  representativa) pra medir: logs, métricas, APM, `pg_stat_statements`,
  CloudWatch, traces, ou capacidade de rodar EXPLAIN.

## Execution Workflow

1. Nomeie a otimização proposta. "Vamos migrar do Postgres pra
   DynamoDB." "Vamos cachear resposta do endpoint X." "Vamos
   horizontalizar o worker pool." Explícito.
2. Pergunte: **qual é o número medido que justifica?** Não "parece
   lento" — número. P99 de X ms, throughput atual Y req/s, erro rate
   Z%, CPU W% no red zone consistente, ou similar.
3. Se a resposta é "não sei" ou "parece", volta pra medir antes de
   qualquer mudança:
   - **Endpoint lento**: profile com APM (DataDog, New Relic,
     CloudWatch X-Ray) ou `pg_stat_statements` + EXPLAIN ANALYZE na
     query dominante. Separe latência de DB, de serialização, de
     HTTP externo.
   - **Query lenta**: `EXPLAIN ANALYZE` com dados reais (não dev).
     Índice faltando? Sequential scan em tabela grande? N+1? Join sem
     filtro no lado direito?
   - **"CPU alto"**: olhe os 4 golden signals (latency, traffic,
     errors, saturation). CPU sozinho pode ser saudável; só importa
     se tá acompanhado de latência subindo ou erros.
   - **"Não escala"**: compare throughput atual vs throughput alvo do
     próximo ano (com evidência de crescimento) vs limite teórico da
     arquitetura atual. Se alvo < limite, não tem problema de escala.
4. Aplique a hierarquia de otimizações baratas antes de qualquer
   mudança estrutural:
   - **Timeout mal calibrado?** Gratuito: ajusta.
   - **Índice faltando em query frequente?** Barato: adiciona, mede
     antes/depois.
   - **N+1 óbvio?** Barato: faz join ou batched fetch.
   - **Pool de conexão pequeno?** Quase gratuito: ajusta.
   - **Retry sem backoff?** Gratuito: adiciona backoff exponencial.
   - **Faltando cache trivial (memoize função pura)?** Barato.
5. Se otimização barata não resolve, aí entra a conversa de estrutura
   (horizontalizar, cachear distribuído, sharding, migrar store). Mas
   **com número**:
   - "P99 ainda é X ms após índice. Alvo: Y ms. Diff: Z%."
   - "Throughput atual é N req/s, limite teórico A req/s, alvo do
     ano B req/s (evidência de crescimento C%). Horizontalizar para
     aguentar."
   - "CPU saturation em 92% por 6h/dia. Vertical já maxado. Próximo
     passo: horizontalizar."
6. Se vai horizontalizar/cachear/migrar: meça antes da mudança.
   Aplica a mudança. Meça depois. Confirma que o número melhorou.
   "Parece melhor" não conta.
7. Documente a decisão com os 4 números obrigatórios:
   - Throughput/latência atual medido.
   - Throughput/latência alvo do próximo ano.
   - Limite teórico da arquitetura atual.
   - Próximo ponto de corte e custo.
   Se não conseguir responder os quatro, ou está escalando
   prematuramente, ou está por ser surpreendido.

## Rules: Do

- Meça antes. `EXPLAIN ANALYZE`, profile, golden signals, APM, logs
  estruturados. Número vem primeiro; estratégia vem depois.
- Trate as 4 golden signals (latency, traffic, errors, saturation)
  como painel obrigatório. Latência alta com CPU baixa = contention
  ou dependency; CPU alta com latência OK = saudável.
- Prefira otimização barata (índice, N+1, timeout, pool) antes de
  mudança estrutural (cache distribuído, sharding, migração).
- Quando horizontalizar/migrar, meça depois pra confirmar que o
  número melhorou. Regressão silenciosa é comum em refactor grande.
- Documente os 4 números (atual, alvo, limite, próximo corte) em
  cada decisão de escala. Se não conseguir responder, é chute.

## Rules: Don't

- Não aceite "parece lento" / "não escala" como justificativa. Sem
  número, é chute.
- Não migre de Postgres pra Dynamo "pra escalar" sem EXPLAIN na
  query dominante. Na maioria dos casos, um índice resolve.
- Não cacheie sem ter profile do que tá sendo lento. Cache esconde
  problemas (e vira coerência headache) em vez de resolver.
- Não horizontalize sem olhar golden signals. CPU a 30% com latência
  subindo é contention — horizontalizar não resolve, às vezes piora.
- Não shardeie banco antes de maxar um nó. Postgres moderno aguenta
  milhões de rows com bons índices.
- Não aceite "pra preparar pro futuro" sem número do futuro. Alvo do
  próximo ano precisa de evidência, não wishful thinking.
- Não otimize prematuramente em geral. Legibilidade primeiro;
  performance quando houver métrica real.

## Expected Behavior

Após aplicar a skill, a decisão de escala/performance deixa de ser
"vibe" e vira análise curta com número. Ou (a) o problema é barato
(índice, timeout, N+1) e resolve com PR pequeno, ou (b) o problema é
estrutural e a mudança vem justificada por números, com custo
reconhecido.

Resultado de longo prazo: o sistema evolui com menos "otimizações
fantasma" (mudanças que não mexeram o ponteiro) e menos surpresas de
escala (gargalo real aparece com 6 meses de aviso em vez de estourar
na véspera).

## Quality Gates

- Toda proposta de escala/performance tem número medido ao lado:
  latência atual, throughput atual, CPU/memória em saturação, erro
  rate, ou equivalente.
- Otimizações baratas (índice, N+1, timeout, pool, backoff) foram
  consideradas antes de mudanças estruturais.
- Para mudança estrutural (cache distribuído, horizontalizar,
  sharding, migração), os 4 números estão documentados: atual, alvo,
  limite, próximo corte.
- Medição *antes* e *depois* da mudança, para confirmar que o
  ponteiro moveu.
- Nenhum "pra preparar pro futuro" sem número do futuro baseado em
  evidência.

## Companion Integration

**Upstream de `swarch-ticker-vs-rule-per-entity` e
`swarch-pull-over-push-orchestration`** (Family 5). Antes de aplicar
esses padrões, confirma que o problema é de fato de escala. Se N é
pequeno (50 entidades, 10 jobs/min), Ticker é overkill; se não tem
throttling real, pull-over-push é refactor sem retorno.

**Upstream de `swarch-context-without-microservice`** (Family 4). A
decisão "vamos separar deploy pra escalar" exige número — qual a dor
medida de escala? Sem número, provavelmente é microservice-
spaghetti em vez de bounded context real.

**Upstream de toda Family 3 (Data architecture)**. "Vamos migrar pra
Dynamo" / "vamos usar CDC" / "vamos projeção" — cada uma exige número
medido. `swarch-dual-store-source-of-truth`,
`swarch-cdc-over-dual-write`, etc., ficam mais afiadas quando medir
antes é cultura.

**Pareia com `sweng-kiss-antidote-overengineering`** (software-eng-
pack). KISS e measure-before-scale são disciplinas irmãs: ambas
impedem complexidade sem retorno. KISS olha pro futuro imaginado
(não antecipa); measure olha pro presente medido (não assume).

## Output Artifacts

- Documento curto de análise: número medido, otimizações baratas
  tentadas (e resultado), justificativa pra mudança estrutural (ou
  decisão de não mudar), métricas antes/depois.
- Queries `EXPLAIN ANALYZE` anexadas quando o gargalo é DB.
- Screenshots/logs de golden signals quando o gargalo é runtime.
- Gatilho explícito pra reavaliação: "se throughput passar de X
  req/s, reabrir."
- Opcional: ADR quando a decisão é estrutural grande.

## Example Constraint Language

- Use "must" para: número medido antes de qualquer mudança de escala;
  medição depois pra confirmar que resolveu; os 4 números (atual,
  alvo, limite, próximo corte) pra mudanças estruturais.
- Use "should" para: tentar otimização barata antes da estrutural;
  usar APM/golden signals como painel; anexar EXPLAIN quando DB é
  suspeito.
- Use "may" para: qual APM usar, formato exato do doc de análise,
  onde guardar (ADR, PR description, confluence).

## Troubleshooting

- **"Não tenho APM, como meço?"**: logs estruturados com timestamp de
  início/fim + CloudWatch + `pg_stat_statements` cobrem 80% dos
  casos por quase zero custo. APM vira obrigatório quando o sistema
  passa de X serviços (geralmente 3-5), não desde o dia 1.
- **"EXPLAIN mostra sequential scan mas a query é rápida"**: tabela
  pequena, não importa. Só vira problema quando cresce. Monitora
  crescimento da tabela; se tá crescendo linearmente com tráfego,
  adiciona índice preventivo (baixíssimo custo). Se é tabela de
  metadados que nunca cresce, deixa.
- **"O time insiste em migrar"**: traga os 4 números pra mesa. "Qual
  o throughput atual medido? Qual o limite da arquitetura atual?
  Qual o alvo evidenciado do próximo ano?" Se não há números, a
  decisão é política, não técnica — pelo menos fica explícito.
- **"Já medi e realmente tá saturado"**: ótimo, agora temos conversa
  real. Hierarquia de resposta: otimização barata → cache
  localizado → horizontalização → refactor de arquitetura. Não pula
  níveis sem justificar.
- **"Mas cache é rápido de adicionar"**: é rápido de adicionar e
  lento de remover. Cache cria coerência problem, TTL decisions,
  invalidation bugs. Se tá lento por índice faltando, resolve o
  índice — é mais simples que cache e mais duradouro.
- **"Horizontalizei e não melhorou"**: comum quando o gargalo é
  stateful compartilhado (banco, Redis write-heavy) e não no que
  foi horizontalizado. Volta pros golden signals — onde a saturação
  realmente tá?

## Concrete Example

Proposta na mesa: "Vamos migrar o Postgres do Argos pra DynamoDB,
está lento".

Aplicando measure-before-scale:

1. "Quais queries estão lentas?" — ninguém sabia exato. Pausa.
2. Ativa `pg_stat_statements`. Roda por 24h em produção.
3. Top 3 queries:
   - Query A: `SELECT * FROM crawl_runs WHERE target_id = $1 ORDER
     BY started_at DESC LIMIT 1`. Chamada 200k vezes/dia. Média
     2.3s. EXPLAIN: sequential scan em tabela de 12M rows.
   - Query B: `SELECT * FROM targets WHERE active = true AND domain
     = $1`. 50k/dia. Média 400ms. Index existe mas seletividade
     baixa.
   - Query C: dashboard admin, 50 chamadas/dia, 5s. Irrelevante.
4. Otimizações baratas:
   - Query A: índice composto `(target_id, started_at DESC)`. Tempo
     cai pra 12ms. PR de 3 linhas.
   - Query B: índice parcial `WHERE active = true`. Tempo cai pra
     8ms. PR de 2 linhas.
5. Medição depois (24h): P99 do endpoint principal caiu de 3.1s pra
   180ms. CPU do Postgres caiu de 70% pra 15%.
6. Migração pra Dynamo: cancelada. Postgres atual aguenta, e com
   folga, o throughput do próximo ano (evidência: ~40% YoY).

Resultado: 2 PRs de 5 linhas no total, semanas de migração
economizadas. "Postgres não escala" era chute; o problema era índice
faltando.

Caso contrário (onde migração vale): hot state do Argos (high-write,
query por chave, TTL) — medido que Postgres com write-amplification
tava chegando no limite de write IOPS, e o perfil de acesso (PK
lookup + TTL) encaixa perfeito em DynamoDB. Nesse caso, os 4 números
justificaram a migração: atual (write IOPS no red zone), alvo (2x
crescimento evidenciado), limite (RDS instance size maxado), próximo
corte (DynamoDB on-demand, custo conhecido).

## Sources

- [[docs/rules/Escalabilidade sem Prematuridade]]
- Síntese direta da regra "Meça antes de assumir que 'não escala'" do
  Danilo. Preserva `EXPLAIN ANALYZE` antes de migrar, profile antes
  de cachear, 4 golden signals antes de horizontalizar, os 4 números
  obrigatórios (atual, alvo, limite, próximo corte), e o princípio
  raiz ("'não escala' sem número ao lado é chute").

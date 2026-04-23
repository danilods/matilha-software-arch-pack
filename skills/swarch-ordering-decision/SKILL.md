---
name: swarch-ordering-decision
description: "Use when decidindo 'preciso de FIFO?', 'Kinesis ou SQS?', 'Kafka com 1 partição pra garantir ordem?' — força escolha consciente: só pague por ordenação quando a semântica do domínio exige, não 'seria bom se chegasse em ordem'."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando a escolha entre mensageria ordenada e não
ordenada está sendo feita — ou quando o sistema atual está
sofrendo de ordenação mal decidida. Sinais típicos:

- "Preciso de FIFO pra esse caso?"
- "Uso Kinesis com partition key ou SQS standard?"
- "Kafka com 1 partição pra garantir ordem — tá ok?"
- "Minha fila FIFO tá com throughput baixo e eu paralelizei o
  consumidor."
- "Projeção do Postgres às vezes chega fora de ordem e quebra."
- "Seria bom se chegasse em ordem" (alerta: *seria bom* ≠
  *exigência*).

Regra raiz: **só pague por ordenação quando a semântica do
domínio exige.** Ordered (FIFO SQS, Kinesis com partition key,
Kafka) é caro: limita paralelismo, complica retry, exige
backpressure. Unordered (EventBridge, SNS, SQS standard)
escala horizontalmente sem dor. "Seria bom se chegasse em
ordem" é conveniência, não exigência.

## Preconditions

- Você está desenhando um fluxo de eventos/mensagens novo, ou
  escolhendo substrato para uma fila, ou debugando problema de
  ordem numa projeção existente.
- Existe pelo menos um caso concreto (evento X, consumidor Y)
  em discussão — não é debate abstrato.
- Você conhece (ou pode aprender) os requisitos de throughput
  esperado e a tolerância a duplicatas/reordenação do
  consumidor.

## Execution Workflow

1. Formule a pergunta: "se a mensagem M₂ chegar antes da M₁
   (onde M₁ foi produzida primeiro), o sistema **quebra**?"
   - Se quebra (saldo errado, estado inconsistente que não se
     auto-corrige) → domínio exige ordem.
   - Se só fica "esquisito por um tempo" e eventualmente
     converge → não exige; é conveniência.
2. Pergunte sobre escopo da ordem: ordem **global** (todas as
   mensagens entre si) ou ordem **por chave** (por entidade,
   por usuário, por content_id)? Ordem global é exponencialmente
   mais cara; ordem por chave é quase sempre suficiente.
3. Se ordem por chave resolve, use particionamento: Kinesis com
   partition key, Kafka com key routing, SQS FIFO com
   MessageGroupId. Throughput escala por número de chaves
   distintas.
4. Se a resposta ao passo 1 foi "não quebra — converge", use
   unordered e faça o consumidor **idempotente + ordenável por
   timestamp** (ver troubleshooting). Projeções de leitura
   convergem mesmo com reordenação se forem construídas para.
5. Se escolheu ordenado, documente o custo: paralelismo
   limitado (geralmente 1 por MessageGroupId), retry mais
   complexo (bloqueia grupo inteiro), backpressure
   necessário.
6. Revisite depois de uma semana em produção: a escolha se
   provou necessária? Ou o sistema teria funcionado unordered
   com idempotência? A reversão de ordered → unordered é
   barata; a migração inversa, não.

## Rules: Do

- Prefira unordered por default. EventBridge, SNS, SQS standard
  escalam sem dor. Ordene apenas quando o domínio exige.
- Escolha **ordem por chave** (partition key / MessageGroupId)
  em vez de ordem global sempre que possível. Quase todos os
  casos reais são "por usuário" ou "por entidade", não "global".
- Torne consumidores **idempotentes** — processam a mesma
  mensagem duas vezes sem quebrar. Isso permite retry seguro e
  tolera reordenação em consumidores unordered.
- Adicione **timestamp monotônico ou versão** no payload.
  Consumidores que constroem projeção podem descartar
  "reordenações antigas" (last-write-wins por timestamp).
- Documente a decisão: "escolhi ordered por X, paga o custo de
  Y" ou "escolhi unordered com idempotência por Z".

## Rules: Don't

- Não use FIFO SQS + consumidor paralelizado sem MessageGroupId
  bem escolhida. Você degrada para standard com overhead de
  FIFO (pior dos dois mundos).
- Não use Kafka com 1 partição "pra garantir ordem global". O
  throughput cai ao chão e você perde a razão de escolher Kafka.
  Se ordem global é requisito, Kafka é ferramenta errada — vá
  pra DB transacional.
- Não confunda "seria bom se chegasse em ordem" com "preciso
  que chegue em ordem". Conveniência não paga o custo de
  ordered.
- Não deixe o consumidor assumir ordem em substrato unordered.
  Se sua projeção quebra quando mensagem chega fora de ordem, o
  bug está no consumidor, não no transporte.
- Não misture ordered e unordered no mesmo fluxo sem design
  explícito. "Publica em SNS (unordered) e também em SQS FIFO"
  é sinal de que a semântica está confusa — resolva.

## Expected Behavior

Após aplicar a skill, a escolha de ordered vs unordered está
explícita com justificativa de domínio (não "pra garantir"
genérico). Consumidores são idempotentes e toleram
reordenação quando o substrato é unordered. Throughput real
do sistema bate com o que o design prevê — sem surpresas de
"por que tá tão lento?" depois de ir pra produção.

No longo prazo: o sistema escala horizontalmente pelo caminho
correto (unordered + idempotência, ou ordered + por chave com
granularidade certa), e a migração de um para outro não é
necessária porque a escolha inicial foi consciente. Esse é o
ganho em evitar custos ocultos de ordenação mal decidida.

## Quality Gates

- Decisão documentada: qual requisito de domínio justifica
  ordered (se aplicável)? Ou qual idempotência justifica
  unordered?
- Se ordered: MessageGroupId/partition key escolhida, throughput
  previsto × paralelismo confirma que atende a carga.
- Se unordered: consumidor idempotente, testado com duplicata e
  reordenação.
- Nenhum Kafka topic crítico com 1 partição "pra garantir
  ordem global".
- Projeção de leitura tem estratégia de convergência
  (timestamp, versão, last-write-wins) se o substrato é
  unordered.

## Companion Integration

**Pareia com `swarch-fact-vs-command-events`** (fatos com
timestamp rico permitem ordenar por timestamp no consumidor —
comandos imperativos tendem a forçar ordem no transporte) e
com `swarch-event-gateway-boundary` (no gateway público, a
maioria dos casos é unordered com filtros granulares; ordered
dentro do mesmo bounded context é caso especial, não padrão).

Pareia também com `matilha-sysdesign-pack:sysdesign-event-streaming-kafka`
(sysdesign analisa throughput/latência/retention de Kafka vs
alternativas — esta skill aplica a decisão de ordenação a
qualquer substrato) e com
`matilha-software-eng-pack:sweng-kiss-antidote-overengineering`
(escolher ordered "por precaução" é over-engineering com
custo alto — KISS é default unordered + idempotência).

## Output Artifacts

- Decisão documentada: ordered vs unordered, com justificativa
  ("domínio exige porque ...") e custo aceito ("paga paralelismo
  limitado a N por chave").
- Se ordered: schema de partition key / MessageGroupId
  escolhido.
- Se unordered: teste do consumidor com duplicata +
  reordenação, demonstrando idempotência.
- Opcional: métrica de lag / latência / throughput para
  monitorar a escolha em produção.

## Example Constraint Language

- Use "must" para: documentar justificativa de ordem quando
  escolher ordered; exigir idempotência de consumidor quando
  escolher unordered; proibir Kafka com 1 partição pra "ordem
  global".
- Use "should" para: preferir unordered por default; preferir
  ordem por chave em vez de ordem global quando ordered é
  necessário; adicionar timestamp/versão a todos os payloads.
- Use "may" para: granularidade da MessageGroupId (por
  usuário, por tenant, por entidade); substrato específico
  dentro do tipo (SQS FIFO vs Kinesis vs Kafka para ordered).

## Troubleshooting

- **"Usei FIFO SQS + 10 consumidores paralelos e tenho
  duplicatas"**: FIFO com paralelismo exige MessageGroupId
  **bem escolhida**. Se todo mundo tá no mesmo grupo, vira
  serial. Se tá em grupos aleatórios, vira standard com hat
  de FIFO. Escolha a chave que representa a unidade real de
  ordem (ex.: user_id).
- **"Kafka com 1 partição pra garantir ordem global"**:
  throughput de Kafka com 1 partição é ~dezenas de MB/s, não
  milhares. Se precisa ordem global + throughput, Kafka é a
  ferramenta errada. Reveja: provavelmente ordem **por
  chave** basta.
- **"Projeção em PG diverge porque eventos chegam fora de
  ordem"**: duas saídas — (a) adicione `last_updated_at` e
  descarte updates mais antigos que o estado atual
  (last-write-wins); (b) construa a projeção via CDC
  ordenado (Streams/WAL), não via evento público unordered.
- **"'Seria bom' que chegasse em ordem"**: não é exigência.
  Aceite unordered e torne consumidor tolerante. Ordered é
  caro em produção; "seria bom" não paga o custo.

## Concrete Example

Argos — `ContentProcessed` no bus `argos.platform`:

- **Decisão**: unordered (EventBridge default).
- **Justificativa de domínio**: cada `content_id` é
  independente. Consumidores processam itens sem interação
  entre eles. Se `content_A` chega antes de `content_B`,
  zero impacto — não existe "ordem entre conteúdos".
- **Consumidores**: cada um filtra por regra e processa
  idempotentemente (todos usam `content_id` como chave
  natural de deduplicação).

Dentro da pipeline Argos — coreografia via DynamoDB Streams:

- **Decisão**: ordered por chave (o próprio `content_id` é a
  partition key natural do Dynamo).
- **Justificativa de domínio**: a sequência
  `Cleaner → Classifier → Extractor → Router → Notifier` para
  **o mesmo content_id** precisa ser serial (estado é
  monotonicamente avançado). Entre content_ids diferentes,
  paralelo total.
- **Custo aceito**: throughput limitado por shard do Dynamo
  Stream — aceitável na escala do Argos.

Contraste: se alguém propusesse "vamos usar FIFO SQS pra isso",
seria ordenação **global** custando paralelismo desnecessário.
Ordem **por content_id** (via partition key do Dynamo) é
gratuita e suficiente.

## Sources

- [[docs/rules/Event-Driven Decoupling]]
- Síntese da regra "Ordered vs unordered — decida
  conscientemente" e da seção "Sinais de Alerta: ordenação
  mal decidida". Preserva o critério "semântica do domínio
  exige" vs "seria bom se", a prática de ordem por chave em
  vez de ordem global, e o alerta sobre Kafka com 1 partição
  e FIFO SQS paralelizado.

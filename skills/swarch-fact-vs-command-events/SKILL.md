---
name: swarch-fact-vs-command-events
description: "Use when naming an event ('SendEmail ou EmailRequested?'), when 'isso é evento ou RPC?', or when o payload carrega instruções em vez de fatos — força evento a descrever o que aconteceu no passado, não o que o consumidor deve fazer."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando um evento está sendo nomeado, revisado, ou quando
o payload está sendo desenhado. Sinais típicos:

- "Que nome dou pra esse evento? `SendEmail` ou `EmailRequested`?"
- "Isso é evento ou RPC disfarçado?"
- "O consumidor precisa saber o tom da mensagem — coloco no
  payload?"
- "O produtor fica fazendo polling depois de publicar pra ver
  se terminou."
- Um evento `NotifyUser` só tem um consumidor e sempre terá.
- O payload contém `action: "send_alert"` ou equivalente
  imperativo.

Regra raiz: **eventos não são RPC disfarçado. São fatos
publicados sobre o mundo.** O payload descreve **o que
aconteceu**, não **o que fazer**. Quem decide o que fazer é o
consumidor — sempre.

## Preconditions

- Você está desenhando um evento novo, revisando o nome de um
  evento existente, ou discutindo se um caso deve virar evento
  ou chamada direta.
- Existe produtor e pelo menos um consumidor identificáveis. Se
  ainda é abstrato demais, a decisão fica pra skill de bounded
  contexts antes.
- O substrato de mensageria está decidido ou em discussão
  (EventBridge, SNS, SQS, Kafka) — esta skill não escolhe o
  transporte; foca no contrato semântico.

## Execution Workflow

1. Escreva o nome candidato do evento. Pergunta: o verbo é no
   passado (`-ed`, `-ado`, `-ida`) ou no imperativo? Se é
   imperativo (`Send`, `Process`, `Notify`), é comando
   disfarçado — reescreva.
2. Liste os consumidores conhecidos. Se for exatamente um e
   provavelmente sempre será um, considere chamada direta —
   evento é ferramenta de desacoplamento de *vários* consumidores.
3. Escreva o payload candidato. Para cada campo pergunta:
   descreve o fato (estado do mundo) ou prescreve ação ("com
   tom alegre", "em português", "por Slack")? Remova prescrições
   — são decisões do consumidor.
4. Teste o evento com o exercício "adicionar consumidor novo":
   um time pede para reagir ao evento de forma diferente.
   Preciso mudar código do produtor? Se sim, o payload está
   prescritivo ou pobre demais.
5. Teste com o exercício "remover consumidor": removo um
   consumidor. O produtor nota? Se sim, há acoplamento vazando —
   produtor não deveria saber quem escuta.
6. Se o produtor "espera" resposta via polling depois de
   publicar, você construiu RPC síncrono com 3 pulos de rede.
   Volta pra chamada direta.

## Rules: Do

- Nomeie eventos como fatos no passado: `ContentProcessed`,
  `OrderPlaced`, `UserRegistered`, `EmailRequested`. Pretérito +
  particípio (ou gerúndio em inglês: `Processing` → não, é ainda
  em progresso; melhor `Processed`).
- Payload descreve estado: o que aconteceu, qual entidade,
  quando, com que atributos. Quem vai fazer o quê é
  responsabilidade do consumidor.
- Inclua metadados estáveis que consumidores provavelmente
  usarão para filtrar: categoria, domínio, severidade,
  timestamp. Filtros granulares desacoplam mais.
- Publique uma única vez, num bus dedicado (ver
  `swarch-event-gateway-boundary`). Publicar o mesmo evento em
  N buses para N consumidores reintroduz acoplamento.
- Trate o evento como contrato público: uma vez em produção,
  evolui por adição de campo opcional — nunca renomeação nem
  remoção.

## Rules: Don't

- Não use verbo imperativo em nome de evento. `SendEmail`,
  `ProcessOrder`, `NotifyUser` são comandos, não fatos.
- Não coloque instruções no payload ("com tom alegre", "via
  Slack canal #alerts", "em português"). Isso é decisão do
  consumidor, não estado do mundo.
- Não publique evento para fazer RPC com passo a mais.
  1 produtor + 1 consumidor + polling pela resposta = chamada
  direta vestida.
- Não deixe produtor saber quem são os consumidores. Se
  adicionar consumidor novo exige mexer no produtor, a
  fronteira está vazando.
- Não mude semântica de campo mantendo o nome ("agora
  `classification` pode ser lista em vez de string"). Consumidor
  quebra em produção sem aviso.

## Expected Behavior

Após aplicar a skill, o evento fica nomeado como fato no passado,
o payload descreve o mundo sem prescrever ação, e o produtor é
imperativamente ignorante sobre consumidores. Adicionar um
consumidor novo é regra de subscrição — nunca mudança no
produtor. Remover um consumidor é silencioso — produtor não
nota.

No longo prazo: o time que produz um evento pode evoluir seu
sistema sem coordenar com os times que consomem, e vice-versa —
desde que o contrato (shape do payload) seja respeitado. Essa é
a propriedade que justifica investir em event-driven em vez de
RPC.

## Quality Gates

- Nome do evento é particípio passado / fato. Verificável em
  code review: `grep -E "(Send|Process|Notify|Trigger)"` em
  detail-types retorna vazio.
- Payload contém só estado (IDs, atributos, timestamps). Nenhum
  campo é instrução para consumidor.
- Teste "adicionar consumidor novo" passa sem mudar código do
  produtor — só regra de subscrição.
- Teste "remover consumidor" passa sem produtor notar.
- Nenhum polling do produtor sobre resultado do evento. Se
  precisa de resposta, é RPC — use chamada direta.

## Companion Integration

**Pareia com `swarch-event-gateway-boundary`** (esta skill é o
contrato semântico do evento; aquela desenha a topologia de
fan-out quando você tem N consumidores heterogêneos). E com
`swarch-ordering-decision` (fatos no passado com metadados
ricos permitem que consumidores decidam se precisam de ordem ou
não — comandos imperativos tendem a forçar ordem).

Pareia também com `matilha-sysdesign-pack:sysdesign-event-streaming-kafka`
(sysdesign decide Kafka vs alternativas a partir de NFRs;
esta skill define a semântica de qualquer evento —
independente do transporte). Um evento mal nomeado vira RPC em
qualquer substrato.

## Output Artifacts

- Nome do evento no particípio passado, com critério nomeado
  (ex.: `ContentProcessed`).
- Payload schema com campos de estado (sem prescrições).
- Lista de consumidores conhecidos — e verificação de que o
  produtor não os referencia em código.
- Regra de filtragem (se em EventBridge: pattern por consumidor)
  usando os metadados do payload.

## Example Constraint Language

- Use "must" para: exigir nome de evento no particípio passado;
  proibir campos de instrução no payload; exigir que adicionar
  consumidor não mude o produtor.
- Use "should" para: preferir metadados granulares
  (classification, domain, category) que permitam filtros
  sofisticados por regra EventBridge; preferir timestamp ISO-8601
  UTC.
- Use "may" para: incluir versão do schema (`schema_version: 1`)
  no payload; como serializar tipos complexos (datetime, Decimal).

## Troubleshooting

- **"Mas o consumidor quer saber se é urgente ou normal"**:
  inclua `severity: CRITICAL | NORMAL` como *fato* (estado do
  item) — não como instrução. O consumidor lê e decide.
- **"O produtor precisa saber se o consumidor recebeu"**:
  confirmação é responsabilidade do substrato (SQS ack, Kafka
  commit). Se o produtor quer saber semanticamente "foi
  processado", o consumidor emite *outro* evento de fato
  (`NotificationSent`) — não retorna resposta.
- **"Meu evento `EmailRequested` virou `SendEmail` porque só
  tem um consumidor"**: aí não é evento — é chamada direta. Faça
  `notifier.send_email(...)` explicitamente. Fingir que é
  event-driven quando é 1↔1 adiciona latência sem desacoplar.
- **"Produtor precisa do ID gerado pelo consumidor (ex.:
  `message_id` do serviço de email)"**: é RPC, aceite. Use
  chamada síncrona, ou faça o consumidor emitir outro fato
  (`EmailSent { email_id, request_id }`) que o produtor consome
  se quiser correlacionar.

## Concrete Example

Evento mal nomeado (visto em projetos reais):

```json
{
  "detail-type": "NotifyUser",
  "detail": {
    "user_id": "123",
    "channel": "slack",
    "message": "Seu conteúdo foi processado com sucesso!",
    "tone": "friendly"
  }
}
```

Problemas: nome é imperativo (`NotifyUser`); payload tem
instruções (`channel`, `message`, `tone`). Se quero adicionar
notificação por email, preciso mudar o produtor. Se quero
notificação em inglês, preciso mudar o produtor. Falso event-driven.

Reescrito como fato:

```json
{
  "detail-type": "ContentProcessed",
  "detail": {
    "content_id": "abc",
    "user_id": "123",
    "classification": "CRITICO",
    "category": "edital",
    "processed_at": "2026-04-21T14:22:01Z"
  }
}
```

Agora: o Slack Notifier consome filtrando por `classification`; o
Email Notifier consome filtrando por `user_id`; o tradutor
consome e decide idioma no próprio consumidor. Produtor não
conhece nenhum deles. Adicionar WhatsApp Notifier é regra de
subscrição nova, zero linha no produtor.

## Sources

- [[docs/rules/Event-Driven Decoupling]]
- Síntese do princípio raiz ("eventos não são RPC disfarçado —
  são fatos publicados"), da regra "publique fatos, não
  comandos", do sinal de alerta "nome de evento é verbo
  imperativo", e do teste de fumaça (adicionar/remover
  consumidor sem mexer no produtor).

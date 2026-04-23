---
name: swarch-event-schema-contract
description: "Use when alguém pergunta se pode mudar/remover/renomear um campo de um evento em produção, ou como versionar o contrato do evento — schema publicado é contrato público imutável."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando alguém no time pergunta "posso mudar esse campo?", "dá pra
renomear esse atributo?", "como versiono esse evento?", ou quando um PR
altera o shape de um evento que já está publicado (EventBridge, SNS, Kafka,
Kinesis). Também fira quando aparece a ideia de "ah, só vou adicionar esse
campo obrigatório, é rápido" — qualquer mexida em evento vivo ativa o
skill. Não confundir com `swarch-fact-vs-command-events` (nomear o evento) nem
com `swarch-event-gateway-boundary` (decidir para quem publicar): este é
especificamente sobre **estabilidade do schema depois de publicado**.

## Preconditions

- O evento em questão **já está em produção** (ou prestes a entrar). Se
  ainda está em desenvolvimento local, mude à vontade.
- Existe pelo menos um consumidor — mesmo que seja interno ao time. Se
  ninguém consome ainda, é RPC com passo a mais; resolva isso antes.
- O time tem como listar os consumidores (regras EventBridge, subscribers
  SNS, consumer groups Kafka). Se não sabe quem consome, você já quebrou
  o contrato — `swarch-projection-rebuild-discipline` ajuda a reencontrar.
- Há um local para registrar versão do contrato (detail-type, header
  Kafka, campo `schema_version` no payload, ou registro num schema
  registry tipo EventBridge Schemas / Confluent SR).

## Execution Workflow

1. **Identifique o evento e seus consumidores.** Liste a source +
   detail-type (ou equivalente) e todo mundo que está ouvindo. Se o
   time não consegue listar, a primeira ação é mapear — mudar schema
   sem saber quem depende é soltar granada.
2. **Classifique a mudança proposta** em uma de três categorias:
   - **Aditiva** (campo novo **opcional**, enum com valor novo que
     consumidores podem ignorar): seguro, publique.
   - **Destrutiva** (renomear, remover, mudar tipo, tornar obrigatório um
     campo que era opcional, mudar semântica mantendo o nome): exige
     versionamento.
   - **Ambígua** ("eu juro que ninguém usa esse campo"): trate como
     destrutiva até provar o contrário com log/metric.
3. **Para mudança aditiva**, documente o novo campo no schema registry
   (ou no doc do evento), marque como opcional com default claro, e
   suba. Consumidores antigos ignoram, novos consomem.
4. **Para mudança destrutiva**, abra uma **nova versão** do evento:
   - Novo `detail-type` (ex.: `ContentProcessed.v2`) ou novo header
     `schema_version: 2`.
   - Produtor publica **os dois** durante a janela de migração.
   - Cada consumidor migra no seu tempo.
   - Quando todos migraram (confirmado por métrica, não por "eu acho"),
     produtor para de publicar v1 e os recursos são removidos.
5. **Comunique a mudança.** Changelog do contrato + aviso aos times
   consumidores. Se o evento é crítico, defina uma **deadline de
   sunset** da versão antiga com data real.
6. **Registre o contrato.** O schema vai para o registry / repositório
   compartilhado. A regra: se não está no registry, não existe.

## Rules: Do

- Trate o evento publicado como API pública externa. Mesma disciplina
  que você aplicaria num endpoint REST exposto a clientes — porque é
  exatamente isso.
- Adicione campos como **opcionais com default sensato**. Consumidores
  antigos ignoram, consumidores novos usam.
- Versione explicitamente (`.v2`, `schema_version: 2`) quando a mudança
  for destrutiva. Versão implícita ("agora o campo tem outro significado")
  é a raiz de bug em produção às 3h da manhã.
- Mantenha as duas versões vivas durante a migração, com janela
  definida em dias/semanas — não "até alguém lembrar de desligar".
- Registre o schema num lugar compartilhado (EventBridge Schemas,
  Confluent SR, repo `events/` do time). Se não está registrado, não
  existe contrato — só boa-fé.

## Rules: Don't

- Não renomeie campo em evento já em produção. Não importa que o nome
  atual seja feio. Deprecate + adicione o novo + versione.
- Não adicione campo obrigatório novo na mesma versão. Obrigatório
  sempre exige v2.
- Não mude semântica mantendo o nome (campo `status` que antes era
  `PENDING|DONE` e agora também pode ser `RETRYING`): consumidores
  validam contra enum conhecido e quebram silenciosamente.
- Não descubra consumidores "orgânicamente" na hora da mudança. Mapeie
  antes. Quem não sabe quem consome, não tem contrato.
- Não publique a mesma detail-type em buses diferentes para
  consumidores diferentes. Agora você tem dois contratos públicos pra
  manter sincronizados e vai esquecer de um.

## Expected Behavior

Depois do skill aplicado, mudanças em eventos passam por um fluxo
explícito: classificar (aditiva vs destrutiva), versionar se
necessário, migrar consumidores com janela definida, desligar a versão
antiga com evidência de que ninguém depende mais. A conversa no PR
muda de "posso mudar esse campo?" para "essa mudança é aditiva ou
exige v2?" — e a resposta vem do checklist, não da intuição.

O evento vira ativo versionado e documentado, não texto vivo no código.
Quando um consumidor novo entra, ele lê o schema no registry, não a
tabela interna do produtor.

## Quality Gates

- Todo evento publicado tem **schema registrado** num lugar versionado
  (registry externo ou arquivo no repo), acessível a consumidores.
- Lista de consumidores por evento existe e é atualizada.
- Mudanças destrutivas **só entram em produção** via nova versão com
  janela de coexistência definida.
- Sunset de versão antiga tem critério explícito (ex.: "zero
  invocações na v1 por 7 dias consecutivos"), não palpite.
- Novos campos obrigatórios **sempre** ganham versão nova — nunca
  entram no shape existente.

## Companion Integration

- **Complementa** `swarch-fact-vs-command-events` — aquele skill decide
  o nome/semântica do evento; este decide como o shape evolui depois.
- **Usa** `swarch-projection-rebuild-discipline` quando uma mudança
  exige replay de eventos passados para reconstruir projeção.
- **Dialoga com** `matilha-sysdesign-pack:sysdesign-event-streaming-kafka`:
  aquele escolhe a plataforma e estratégia de ordenação; este foca na
  disciplina de schema, que é ortogonal à escolha Kafka vs EventBridge
  vs SNS.
- Metodologia: fase 30 (arquitetura) na criação do evento; fase 50
  (review) em toda mudança de shape.

## Output Artifacts

- Schema do evento no registry (EventBridge Schemas, Confluent SR, ou
  `events/<name>.v1.json` no repo).
- Lista de consumidores por evento (planilha, documento, ou saída de
  `aws events list-rules --event-bus-name X`).
- ADR curto quando promover v2: motivo da mudança, janela de
  coexistência, critério de sunset.
- Changelog do evento (arquivo `events/CHANGELOG.md` ou equivalente).

## Example Constraint Language

- Use "deve" para: versionar mudança destrutiva, manter janela de
  coexistência, registrar schema, listar consumidores antes de mudar.
- Use "pode" para: escolher entre `detail-type.v2` vs header
  `schema_version`, usar schema registry externo vs arquivo no repo,
  escolher janela de 2 semanas vs 2 meses conforme criticidade.
- Use "nunca" para: renomear campo sem versionar, tornar campo
  opcional obrigatório sem versão, mudar semântica mantendo nome.

## Troubleshooting

- **"Não sei quem consome esse evento"**: mapeie antes de mudar. No
  EventBridge, `list-rules` + `list-targets-by-rule` dá um ponto de
  partida. Qualquer mudança sem a lista é risco cego.
- **"Preciso remover campo porque vaza dado sensível"**: caso legítimo
  de urgência. Solução: v2 sem o campo + sunset acelerado de v1 (dias,
  não semanas), com comunicação direta aos consumidores. Não dá pra
  remover in-place sem quebrar todo mundo.
- **"Adicionar campo opcional está quebrando consumer"**: consumer
  validando schema com `additionalProperties: false` ou similar. Não
  é culpa do produtor — o consumidor viola o princípio de leitor
  tolerante (robustness principle). Ajuste o consumer, mas registre
  o aprendizado: publique a expectativa no doc do evento.
- **"Versão antiga nunca morre, ficamos com 3 versões vivas"**: sunset
  sem critério. Defina métrica (zero invocações por N dias) e hard
  deadline. Aceitar "sempre tem alguém usando" é aceitar dívida
  infinita.

## Concrete Example

Argos publica `ContentProcessed` no bus `argos.platform`. Payload v1
tem `classification: "CRITICO" | "INFORMATIVO" | "IRRELEVANTE"`.
Produto pede nova categoria `URGENTE` para editais de fase única.

Mudança aditiva? **Não**: consumidores que validam enum quebram ao
receber valor desconhecido. Solução correta: `ContentProcessed.v2`
com enum estendido. Produtor publica ambas as versões por 3 semanas.
Plataforma pedagógica (único consumer que filtra por classification)
migra na primeira semana. Slack notifier, auditoria e admin dashboard
não filtram por classification — indiferentes, migram quando puderem.
Métrica confirma zero consumers na v1 após 3 semanas. Produtor
desliga v1. ADR de 1 página no repo registra a decisão. Custo real:
~2 dias de trabalho. Custo alternativo (mudar in-place): plataforma
pedagógica silenciosamente ignora editais `URGENTE` e descobre uma
semana depois por reclamação de cliente.

## Sources

- `[[docs/rules/Event-Driven Decoupling.md]]` — seção "Schema do evento
  é contrato público" + "Você quebrou o contrato público se".
- Destilado das regras de Danilo a partir do Argos (EventBridge bus
  `argos.platform`) e do Gravicode (eventos de orquestração do
  LangGraph).

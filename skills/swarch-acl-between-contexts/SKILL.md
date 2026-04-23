---
name: swarch-acl-between-contexts
description: "Use when consumer tá lendo tabela interna do producer, evento tá vazando modelo interno, ou alguém pergunta 'posso importar o model do outro contexto direto?' — garante que fronteira entre bounded contexts passa por anti-corruption layer."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando comunicação entre bounded contexts (módulos ou serviços
distintos) está prestes a vazar modelo interno. Sinais típicos:

- "Posso importar o ORM model de Target Management em Notifications?"
- "Esse consumer lê direto a tabela interna do producer — tá certo?"
- "O evento tá carregando `internal_target_state_hash`, é ok?"
- "Consumer faz lookup num segundo endpoint pra enriquecer o evento."
- Duas equipes discutindo o "nome certo" de um campo que aparece em
  dois contextos.
- Schema change em um contexto exige coordenação com outros times antes
  do deploy.
- Agente propõe passar "o objeto inteiro" num evento entre contextos.

A regra de fundo: **ENTRE CONTEXTOS, O CONTRATO É O EVENTO (FATO
PUBLICADO) PASSADO POR ANTI-CORRUPTION LAYER.** Schema interno nunca
cruza fronteira. Se o consumer precisa de 15 campos internos do
producer, ou a fronteira está errada ou o evento é pobre.

## Preconditions

- Dois bounded contexts já estão identificados (vocabulário próprio,
  invariantes próprias). Se não estão, aplique
  `swarch-context-by-vocabulary` primeiro.
- Existe comunicação real ou proposta entre eles: evento, API, leitura
  de banco, import de módulo. A questão é *como* essa comunicação
  atravessa a fronteira.
- Existe alguém (você, agente, PR) defendendo atalho que vaza modelo
  interno, ou houve dúvida legítima sobre como desenhar a tradução.

## Execution Workflow

1. Nomeie os dois contextos e a direção da comunicação. "Changes
   publica, Notifications consome." "Target Management expõe API,
   Crawl consome." Deixe explícito quem é producer, quem é consumer.
2. Identifique o contrato público proposto. Se é evento, qual o
   schema? Se é API, quais campos? Se é leitura de banco compartilhado,
   **pare** — isso já é violação da fronteira.
3. Aplique o teste de vazamento: o contrato carrega algum dos abaixo?
   - Nomes internos do producer (`internal_*`, `_db_*`, `row_version`).
   - Estado que só faz sentido dentro do producer.
   - Campos que forçam o consumer a conhecer a implementação do producer
     (IDs técnicos, flags de migração, versões de schema interno).
   Se sim, você vazou o modelo interno.
4. Aplique o teste de pobreza: o consumer precisa fazer lookup
   adicional num endpoint/banco do producer pra entender o evento?
   Se sim, o evento é pobre — enriqueça-o com o que o consumer precisa,
   ou repense se os contextos estão certos (talvez o consumer devesse
   estar dentro do producer).
5. Desenhe o ACL explicitamente no consumer. Código de tradução:
   recebe o evento (ou resposta API) no formato do producer, devolve o
   modelo interno do consumer. Isso é uma camada real, não uma
   convenção mental — fica em `.../adapters/<producer>_translator.py`
   ou equivalente. **A tradução é assimétrica**: nomes, unidades,
   invariantes do consumer, não do producer.
6. Se o contrato é evento, verifique que é **fato no passado**
   (`ContentProcessed`, `TargetActivated`) e não comando imperativo
   (`NotifyUser`, `ProcessContent`) — comando vazando forma de chamada
   RPC mascarada de evento é outro sintoma do modelo cruzando fronteira.
7. Documente o contrato público (evento schema, API pública) separado
   do modelo interno do producer. São dois artefatos diferentes;
   mudanças no modelo interno não tocam o contrato.

## Rules: Do

- Todo evento entre contextos é fato no passado, com schema explícito,
  versionado, separado do modelo interno do producer.
- Consumer sempre traduz o formato externo pro seu modelo interno
  via camada ACL explícita. Código de tradução isolado em módulo
  próprio.
- Enriqueça o evento com o que o consumer precisa — o evento é o
  contrato, não um ponteiro pra buscar dados no producer.
- Documente o contrato público em lugar separado (schema registry,
  `contracts/`, README do producer). Esse é o artefato que consumidores
  podem ler sem ler o código do producer.
- Versione o evento quando precisar mudar. Adicione campos opcionais
  livremente; remova/renomeie só com versão nova.

## Rules: Don't

- Nunca deixe consumer ler tabela interna do producer. É compartilhar
  banco, não evento. Fronteira deixa de existir.
- Não exponha ORM models diretamente como contrato. ORM model é
  implementação interna; o evento/API é contrato público, escrito à
  mão, estável.
- Não publique eventos com campos internos (`internal_*`, `db_*`,
  estados de migração). Mesmo que "ninguém lê", alguém vai ler e
  acoplar.
- Não permita comando imperativo entre contextos via evento
  (`NotifyUser`, `SendEmail`). Ou é RPC e deve ser API explícita, ou
  é fato e deve ser reescrito como `UserRegistered`.
- Não faça o consumer depender de lookup síncrono no producer pra
  interpretar o evento. Se precisa do campo, bota no evento.
- Não deixe dois contextos "discutirem o nome certo" de um conceito.
  Se ambos têm argumentos válidos, são dois conceitos diferentes — use
  nomes diferentes em cada lado e um ACL traduzindo.

## Expected Behavior

Após aplicar a skill, a fronteira entre contextos fica explícita e
auditável. Contrato público (evento, API) é artefato separado do
modelo interno do producer. Consumer tem camada de tradução isolada:
mudança no modelo interno dele não quebra contratos, e vice-versa.

Resultado de longo prazo: mudanças internas de um contexto (adicionar
tabela, refatorar schema, trocar tecnologia) deixam de cascatear nos
outros. Deploy dos contextos fica independente (ou pelo menos
preparado pra ficar, quando a hora chegar). A discussão "mas se eu
mudar esse campo, quem quebra?" tem resposta clara: quem consome o
contrato público; modelo interno é livre.

## Quality Gates

- Todo evento entre contextos foi nomeado como fato no passado
  (`*Processed`, `*Activated`, `*Registered`) — não comando.
- Schema do evento está documentado em local separado do modelo
  interno do producer (contrato em `contracts/`, schema registry,
  README, ADR).
- Consumer tem camada de tradução (ACL) explícita, em módulo próprio,
  que converte formato externo em modelo interno.
- Nenhum campo do evento tem prefixo `internal_`, `_db_`, ou carrega
  estado que só faz sentido no producer.
- Consumer não faz lookup síncrono no banco do producer. Se precisa
  de dados extras, o evento carrega; se não, repense a fronteira.

## Companion Integration

**Pareia com `swarch-context-by-vocabulary`** e
`swarch-context-without-microservice` — essas três formam o núcleo de
Family 4 (bounded contexts). Vocabulário descobre a fronteira,
microservice-vs-module decide deploy, ACL garante a integridade da
comunicação.

**Pareia com `swarch-fact-vs-command-events`** (Family 2). A disciplina
de "evento é fato no passado" é exatamente o que impede comando mascarado
de atravessar a fronteira. ACL + fact-events = comunicação entre contextos
que não vaza forma de RPC.

**Pareia com `swarch-event-schema-contract`** (Family 2). O contrato
público do evento precisa ser versionado e evoluído com disciplina:
só adiciona opcional, nunca remove sem versão nova. Essa skill garante
que o contrato *existe*; a outra garante que evolui sem quebrar
consumidores.

## Output Artifacts

- Schema do evento documentado (JSON Schema, Avro, TypedDict, pydantic
  model) em arquivo separado do código do producer.
- Código ACL no consumer: função ou classe de tradução isolada,
  testável offline, que recebe o evento externo e devolve modelo
  interno.
- Nota no README do producer: "este contexto publica os seguintes
  eventos públicos. Modelo interno é livre para mudar; contratos
  seguem versionamento."
- Opcional: ADR descrevendo quando escolheu ACL forte vs tradução
  inline, e por quê.

## Example Constraint Language

- Use "must" para: todo contrato entre contextos ser fato no passado
  com schema explícito; consumer ter ACL dedicado (não tradução
  espalhada).
- Use "should" para: schema viver em lugar separado do código do
  producer; evento carregar dados suficientes para consumer não
  precisar de lookup síncrono.
- Use "may" para: formato exato do schema (JSON Schema, Avro, pydantic);
  localização exata do ACL (`adapters/`, `translators/`, `ports/`).

## Troubleshooting

- **"Mas dá trabalho manter o contrato e o model separados"**: dá
  trabalho no primeiro dia e economiza nos próximos cem. Quando o
  modelo interno vira contrato público, qualquer refactor vira
  breaking change coordenado com N times.
- **"O ORM model já tem tudo que o consumer precisa"**: aparentemente.
  Na primeira refatoração interna, você vai querer renomear um campo
  e descobrir que três contextos dependem daquele nome exato. Então
  escolhe entre pagar agora (contrato explícito) ou pagar depois
  (migração coordenada).
- **"O evento ficou gigante"**: dois casos. (a) O consumer realmente
  precisa de muitos campos — então ok, enriquece o evento (melhor que
  lookup). (b) Você tá mandando coisa que só faz sentido no producer
  — pare, revise o schema, tire o que não deveria estar lá.
- **"Posso consultar o producer via API em vez de só evento?"**: pode,
  mas a API é outro contrato público (mesma disciplina se aplica:
  nomes, schema, versionamento). Eventos são assíncronos/desacoplados;
  API é acoplamento síncrono. Escolha consciente.
- **"Tá vazando `internal_state` no evento mas ninguém usa"**: vai
  começar a usar no dia que você menos quer. Retire já; mantém a
  fronteira limpa.

## Concrete Example

No Argos, o contexto **Changes** detecta mudanças em URLs monitoradas
e publica `ContentProcessed`. O contexto **Notifications** consome e
dispara alertas.

Contrato público do evento (schema versionado):

```
ContentProcessed v1 {
  target_id: string
  detected_at: timestamp
  classification: "CRITICO" | "INFORMATIVO" | "IRRELEVANTE"
  summary: string              # resumo curto humano-legível
  before_url: string           # S3 URL do conteúdo anterior
  after_url: string            # S3 URL do conteúdo novo
  extracted_entities: [...]    # entidades do domínio (concurso, banca)
}
```

O que **não** entra: `hash_algorithm`, `db_row_version`,
`internal_classification_score`, `scraper_engine_used`. Esses são
modelo interno de Changes; mudam sem cascatear.

ACL em Notifications (`adapters/changes_translator.py`):

```python
def translate_content_processed(evt: ContentProcessedV1) -> DispatchIntent:
    # DispatchIntent é modelo interno de Notifications
    return DispatchIntent(
        trigger_event_id=evt.target_id + ":" + str(evt.detected_at),
        severity=_map_classification(evt.classification),  # tradução explícita
        channels=_route_by_classification(evt.classification),
        payload=_build_slack_payload(evt.summary, evt.extracted_entities),
    )
```

Feature que mudou a regra de roteamento por classificação (CRITICO →
#editais, INFORMATIVO → #geral): 100% dentro de Notifications, no
`_route_by_classification`. Zero toque no schema do evento, zero toque
em Changes. O ACL foi exatamente a fronteira que permitiu a feature
não vazar.

Caso anti-padrão evitado: havia proposta de Notifications ler direto a
tabela `changes.diffs` pra enriquecer a notificação com o diff
visual. Recusado: diff visual foi adicionado ao evento (`diff_summary_url`
em v2), não via leitura direta. Resultado: Changes podia refatorar a
tabela interna livremente.

## Sources

- [[docs/rules/Bounded Contexts na Prática]]
- Síntese direta da regra do Danilo. Preserva "Fronteira explícita =
  tradução explícita" (ACL), "Contextos compartilham eventos, não
  schemas", os testes de vazamento (`internal_*` fields, lookup
  síncrono em segundo endpoint) e o caso Changes → Notifications do
  Argos.

---
name: swarch-handler-as-adapter
description: "Use when wanting to test Lambda/handler logic without LocalStack, when handler tem if aninhado demais, or when 'preciso subir Docker pra rodar esse teste' — refatora handler como adapter burro e move decisão pra função de domínio pura testável offline."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando a testabilidade de um handler (Lambda, FastAPI,
consumer SQS) está difícil, ou quando decisões condicionais estão
se acumulando dentro dele. Sinais típicos:

- "Preciso testar essa lógica sem subir LocalStack."
- "Handler tem if aninhado demais — vira bagunça."
- "Pra rodar o teste unitário preciso de Docker + mock de boto3 +
  fixture de evento gigante."
- "O handler tá crescendo — já passou de 80 linhas."
- "Essa decisão condicional depende do shape do evento OU da
  regra de negócio? Não sei mais."

Regra raiz: **handlers são burros, domínio é inteligente.** O
handler faz parse, validação leve, delegação, serialização de
saída. Qualquer decisão condicional que não seja sobre *shape do
evento* pertence ao domínio — e o domínio é pura, testável offline.

## Preconditions

- Existe um handler concreto em cima da mesa (arquivo, função) que
  está crescendo ou com teste difícil.
- O projeto tem (ou pode ter) uma camada de domínio separada. Se
  não tem, esta skill é o momento de criar — extraindo a primeira
  função pura do handler.
- Você já aplicou (ou vai aplicar) `swarch-dependency-direction` — o
  domínio extraído não importa SDKs.

## Execution Workflow

1. Abra o handler. Conte linhas úteis (ignore docstrings, imports,
   logging estrutural). Se passa de ~30 linhas, há domínio
   escondido ali.
2. Classifique cada bloco em uma das quatro categorias: (a) parse
   do evento, (b) validação leve (campo obrigatório, tipo básico),
   (c) decisão de negócio, (d) side-effect (escrever no Dynamo,
   publicar evento, notificar Slack).
3. Extraia todos os blocos de categoria (c) para uma função pura
   em `domain/`. Ela recebe dados parseados como argumentos (não
   o evento bruto) e retorna uma decisão/resultado — nunca escreve
   em lugar nenhum.
4. Se a função de domínio precisa de I/O (ex.: consultar cache,
   chamar modelo AI), injete como protocolo no parâmetro. Nunca
   construa cliente dentro do domínio.
5. Reescreva o handler como quatro linhas lógicas: parse, chamada
   de domínio (recebendo injetadores), side-effect, serialização
   de saída.
6. Escreva (ou atualize) o teste de domínio — deve rodar sem
   Docker, sem rede, sem LocalStack, usando só fixtures de dados
   puros. Se ainda precisa de infra, volta ao passo 3 e
   identifica o acoplamento residual.

## Rules: Do

- Handler é adapter, não decisor. Ele traduz evento externo em
  chamada de domínio e escreve o resultado — nada além.
- Decisão condicional sobre *shape do evento* (ex.: `if "s3" in
  event: ... elif "sqs" in event: ...`) fica no handler. Decisão
  sobre *regra de negócio* (ex.: "é crítico?", "já foi
  processado?") vai pro domínio.
- Funções de domínio recebem inputs simples (strings, dicts,
  dataclasses) e retornam saídas simples. Dependências de I/O
  entram por parâmetro (protocolo) — nunca por `boto3.client(...)`
  dentro.
- Teste de domínio é unitário puro: sem Docker, sem LocalStack,
  sem rede. Fixtures são dados, não infra.
- Teste de handler é pequeno: verifica parse e orquestração, com
  função de domínio mockada ou stub — não reexecuta a lógica de
  negócio.

## Rules: Don't

- Não escreva `if` de regra de negócio dentro do handler.
  `if classification == CRITICO: notify_slack()` no handler
  esconde domínio ali dentro.
- Não monte cliente AWS dentro de função de domínio ("só dessa
  vez, é rápido"). É sempre o começo de teste que precisa de
  LocalStack.
- Não aceite "pra testar, sobe o Docker" como custo normal. É
  sinal de acoplamento — refatore antes de normalizar.
- Não faça mock heavy de `boto3` para testar lógica de negócio.
  Se o teste está mockando 5 clientes AWS, a lógica não está no
  lugar certo.
- Não deixe o handler inflar silenciosamente. Revise a contagem
  de linhas úteis em toda PR que toca handler — ~30 é o teto
  prático.

## Expected Behavior

Após aplicar a skill, o handler fica curto (≤ ~30 linhas úteis),
legível numa leitura, e com teste que só verifica orquestração.
A função de domínio extraída tem teste puro — roda em segundos,
sem infra, e cobre o comportamento real do negócio.

No longo prazo: substituir a fonte do evento (S3 → SQS → HTTP)
vira reescrever o handler e manter o domínio. Adicionar caso
novo de negócio vira adicionar teste puro e implementar na função
de domínio — sem tocar adapter. É a separação que permite a
equipe mover rápido sem quebrar coisas.

## Quality Gates

- Handler com ≤ ~30 linhas úteis. Qualquer decisão condicional
  sobre negócio foi extraída.
- Teste de função de domínio roda sem Docker, sem rede, sem
  LocalStack. Tempo de execução em milissegundos.
- Teste de handler usa stub/mock só na borda (boto3, clientes
  externos) — a lógica de domínio é chamada real.
- Nenhum `if classification ==` ou equivalente de regra de
  negócio vive no handler.
- Nenhum `boto3.client(...)` dentro de função importada do
  domínio.

## Companion Integration

**Pareia com `swarch-dependency-direction`** (a regra macro — esta
skill é a aplicação concreta da regra ao nível do handler
individual). E com `swarch-lambda-chain-shape` (forma da
pipeline — handlers finos são pré-requisito para que a cadeia
funcione).

Pareia também com `matilha-software-eng-pack:sweng-kiss-antidote-overengineering`
(o handler "só com um if extra" é cheiro de over-engineering por
descuido — a regra de simplicidade casa com a disciplina de
extrair domínio). E com
`matilha-software-eng-pack:sweng-responsabilidade-tecnica`
(handler gordo é dívida técnica explícita — rastreável).

## Output Artifacts

- Handler refatorado: ~30 linhas, quatro blocos nomeáveis (parse,
  validar, delegar, escrever).
- Função de domínio extraída em `domain/`: pura, com protocolo
  para I/O se necessário.
- Teste unitário de domínio rodando em segundos, sem Docker.
- Teste de handler pequeno, verificando só a orquestração.

## Example Constraint Language

- Use "must" para: exigir teste de domínio sem Docker/LocalStack;
  limitar handler a ~30 linhas úteis; proibir `if` de regra de
  negócio dentro do handler.
- Use "should" para: preferir dataclass como input da função de
  domínio em vez de dict cru; preferir `Protocol` do `typing` para
  injeção de I/O em Python.
- Use "may" para: usar `pytest.fixture` vs `unittest.mock` no
  teste de domínio; organização de sub-pastas dentro de
  `domain/`.

## Troubleshooting

- **"Mas o if é sobre se o campo vem ou não no evento, é shape"**:
  shape-check é validação leve (categoria b) — pode ficar no
  handler. Decisão sobre o *valor* do campo (regra de negócio) é
  domínio.
- **"Extrair 5 linhas pra outro arquivo parece exagero"**: se
  essas 5 linhas são a regra de negócio, sim — elas mudam por
  razão diferente das outras 25 linhas do handler. Separação de
  razões de mudança é o ganho.
- **"Preciso de dados do Dynamo pra decidir"**: injete um
  `repository: ContentRepository` no parâmetro da função de
  domínio. No teste, passa um fake; em produção, passa o real.
- **"O teste do handler agora é quase vazio"**: é a ideia. Se o
  handler é um adapter, o teste dele só precisa verificar que ele
  chama o domínio com os argumentos certos e escreve o resultado
  certo. A lógica real está coberta no teste de domínio.

## Concrete Example

Handler original (52 linhas úteis — já é cheiro):

```python
def lambda_handler(event, context):
    record = event["Records"][0]
    content_id = record["dynamodb"]["Keys"]["id"]["S"]
    text = get_text_from_s3(content_id)
    # lógica de classificação embutida
    if re.search(r"GABARITO|RESPOSTA", text):
        classification = "CRITICO"
    elif len(text) < 100:
        classification = "IRRELEVANTE"
    else:
        classification = "INFORMATIVO"
    ddb.update_item(..., UpdateExpression="SET classification=:c",
                    ExpressionAttributeValues={":c": classification})
    if classification == "CRITICO":
        sns.publish(TopicArn=..., Message=...)
```

Diagnóstico: o `if/elif/else` é regra de negócio — define o que
"crítico" significa. Está escondido no handler. O teste unitário
precisaria mockar boto3 só pra verificar regex.

Refatorado:

```python
# domain/classification.py  (puro — teste sem Docker)
def classify_content(text: str) -> Classification:
    if _looks_like_gabarito(text):
        return Classification.CRITICO
    if len(text) < 100:
        return Classification.IRRELEVANTE
    return Classification.INFORMATIVO

# adapters/lambdas/classifier_handler.py  (~15 linhas úteis)
def lambda_handler(event, context):
    content_id = parse_content_id(event)
    text = content_repo.get_text(content_id)
    classification = classify_content(text)
    content_repo.persist_classification(content_id, classification)
    if classification == Classification.CRITICO:
        publisher.publish_critical(content_id)
```

Teste de `classify_content` roda em 5ms, sem rede, sem Docker.
Teste de handler é 10 linhas verificando orquestração com
`content_repo` e `publisher` fakes.

## Sources

- [[docs/rules/Layering e Dependency Direction]]
- Síntese da seção "Handlers são burros, domínio é inteligente" e
  do sinal de alerta "teste de domínio que precisa de
  Docker/LocalStack". Preserva o teto de ~30 linhas, a
  classificação em quatro categorias de código no handler, e a
  propriedade de testabilidade offline como prova de que a
  separação foi bem feita.

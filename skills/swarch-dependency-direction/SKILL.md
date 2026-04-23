---
name: swarch-dependency-direction
description: "Use when questioning an import ('pode domínio importar boto3?'), when a domain module has SDK references, or when regra de negócio parece saber que Lambda/DynamoDB/Slack existem — recentra a direção da seta: infra sempre depende do domínio, nunca o contrário."
category: swarch
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando a direção da dependência entre camadas está em jogo.
Sinais típicos de gatilho:

- "Esse import tá certo? `domain/classification.py` pode fazer
  `import boto3`?"
- "A regra de negócio depende de Lambda/DynamoDB/Slack?"
- "Por que esse teste de domínio precisa subir LocalStack?"
- "Onde é que eu coloco isso — em `domain/` ou em `infrastructure/`?"
- Você abre um arquivo de `domain/` e encontra `from boto3 import ...`.
- `utils.py` cresceu e ninguém sabe mais o que pertence a ele.

Regra raiz: **dependências apontam sempre para o domínio, nunca em
sentido contrário.** O núcleo (regras de negócio, máquina de estados,
invariantes) não pode saber que AWS existe. Quem sabe disso são as
bordas.

## Preconditions

- Você está prestes a escrever um `import`, revisar um PR, ou aceitar
  uma sugestão do agente que conecta domínio a infraestrutura.
- O projeto tem (ou está se aproximando de ter) uma separação entre
  núcleo e bordas — mesmo que informal (pastas `domain/`,
  `infrastructure/`, `adapters/`, `application/`).
- Existe um caso concreto em cima da mesa (um arquivo, um import, uma
  função) — não é debate abstrato sobre "arquitetura limpa".

## Execution Workflow

1. Localize o arquivo em questão e nomeie a camada a que ele pertence.
   Se está em `domain/`, `application/`, `adapters/` ou
   `infrastructure/` — ou onde seria o análogo no projeto.
2. Liste os imports. Procure por SDKs (`boto3`, `psycopg`, clientes
   HTTP), ORMs, variáveis de ambiente, `datetime.now()` sem injeção.
3. Aplique a regra da seta invertida: *se este arquivo está em
   `domain/` e importa algo de `infrastructure/`, a seta está errada.*
   A dependência deveria ser injetada via protocolo/parâmetro, ou o
   código pertence a outra camada.
4. Decida: (a) mover para a borda, (b) inverter a dependência via
   injeção de protocolo, ou (c) extrair a parte pura para o domínio e
   deixar a parte suja no adapter.
5. Rode o teste de fumaça: abra o arquivo de domínio revisado. Ele
   importa SDK? Chama `time.sleep`/`datetime.now()` sem injeção? Lê
   `os.environ`? Se sim, ainda está acoplado.
6. Documente a decisão em commit message ou comentário curto ("movi
   X pra `infrastructure/` porque importava SDK"), para o próximo
   reviewer (humano ou agente) não tentar desfazer.

## Rules: Do

- Mantenha `domain/` puro: sem `boto3`, sem SDK AWS, sem ORM, sem
  cliente HTTP. Funções recebem strings, dicts, dataclasses — nunca
  objetos de infraestrutura.
- Injete dependências via protocolo (interface/ABC) ou parâmetro. Se
  o domínio precisa de I/O, recebe-o por argumento — nunca importa
  diretamente.
- Coloque `adapters/` e `infrastructure/` como bordas: ambas podem
  importar `domain/`. Domínio nunca importa delas.
- Extraia lógica condicional de handlers para funções de domínio
  puras. Handler só traduz evento → chamada de domínio → side-effect.
- Nomeie `utils.py` com crítica: funções de negócio (ex.:
  `detect_gabarito_pattern`) moram em `domain/`; formatação de data,
  serialização, cliente HTTP moram em `infrastructure/`.

## Rules: Don't

- Não importe SDKs dentro de `domain/`. `from boto3` ou
  `import psycopg` em qualquer arquivo de domínio é red flag. Sempre.
- Não escreva teste de regra de negócio que exige Docker/LocalStack
  para rodar. Se precisa, a regra está acoplada — separe.
- Não crie `utils.py` gigante sem direção — geralmente é domínio mal
  nomeado disfarçado.
- Não aceite "vou colocar esse if aqui no handler porque é mais
  rápido". Nunca é mais rápido — é dívida imediata. O próximo que
  ler vai procurar lógica no handler em vez de no domínio.
- Não faça `lambda A` invocar `lambda B` via `boto3.client("lambda")`.
  Isso reconstrói orquestração síncrona que a event-driven veio
  resolver.

## Expected Behavior

Após aplicar a skill, a camada do arquivo em questão fica explícita.
Se estava em `domain/` e tinha imports de infra, foi movido ou
refatorado. O teste de domínio associado roda sem Docker e sem rede.
Se um SDK precisa ser trocado (ex.: Phi-3-mini → Gemini Flash), o
diff fica restrito a `infrastructure/` — domínio não sofre uma linha
de alteração.

No longo prazo: o núcleo do sistema é testável offline, substituições
de infraestrutura ficam baratas, e novas categorias de regra entram
sem reescrever Lambdas. É o mesmo payload arquitetural observado no
Argos quando a camada AI client foi trocada sem tocar
`domain/classification.py`.

## Quality Gates

- Nenhum arquivo de `domain/` importa SDK (`boto3`, `psycopg`,
  cliente HTTP, ORM). Verificável por grep.
- Teste de domínio roda sem Docker/LocalStack, sem rede, sem mock de
  AWS. Só fixtures de dados puros.
- Handler em `adapters/`/`infrastructure/` tem no máximo ~30 linhas
  úteis — qualquer lógica de negócio foi extraída para `domain/`.
- Funções de `domain/` que precisam de I/O recebem o cliente como
  parâmetro ou protocolo — nunca constroem internamente.
- `utils.py` contém apenas infra-utils (serialização, formatação) —
  nada de lógica de negócio.

## Companion Integration

**Pareia com `swarch-lambda-chain-shape`** (pipeline composition —
como os handlers finos se conectam em cadeia) e com
`swarch-handler-as-adapter` (testabilidade individual do handler).
Esta skill é a regra de direção; aquelas duas são aplicações da regra.

Pareia também com `matilha-software-eng-pack:sweng-responsabilidade-tecnica`
(dívida técnica explícita — violações de dependency direction são
dívida arquitetural que deveria ser rastreada) e
`matilha-software-eng-pack:sweng-nomenclatura-clareza` (nomes claros
ajudam a detectar quando algo está na camada errada — `utils.py` é
o caso canônico).

## Output Artifacts

- Patch/PR que move arquivo para a camada correta, OU inverte
  dependência via injeção de protocolo.
- Commit message curto explicando direção da seta restaurada.
- Opcional: teste de domínio novo (ou existente atualizado) rodando
  sem LocalStack.

## Example Constraint Language

- Use "must" para: proibir imports de SDK dentro de `domain/`;
  exigir injeção de I/O via protocolo/parâmetro; exigir testes de
  domínio sem Docker.
- Use "should" para: preferir extrair lógica de handler para
  `domain/` quando passar de ~30 linhas; preferir nomes explícitos
  em vez de `utils.py` genérico.
- Use "may" para: qual abstração usar para injeção (ABC, Protocol do
  `typing`, dataclass com callable); como organizar sub-pastas
  dentro de `domain/`.

## Troubleshooting

- **"Mas o SDK é só pra ler variável de ambiente rápido"**: ler env
  var é infra. Cria `config.py` em `infrastructure/` e injeta o
  valor já resolvido no domínio.
- **"Preciso de `datetime.now()` no domínio"**: injeta um `clock:
  Callable[[], datetime]` ou `now: datetime` — o domínio não decide
  quando é agora, recebe.
- **"Se eu extrair, vai ficar 3 arquivos pra 10 linhas de lógica"**:
  sim, e vale. O ganho é testabilidade offline + possibilidade de
  trocar infra sem tocar domínio. É o investimento que paga no
  segundo caso de uso.
- **"`utils.py` tem 40 funções misturadas, por onde começo?"**:
  divida em duas listas — negócio vs. infra. Funções de negócio
  (nomes do domínio, decisões de classificação) vão pra `domain/`;
  o resto fica em `infrastructure/utils.py` ou é absorvido por
  bibliotecas.

## Concrete Example

Arquivo `domain/change_detection.py` contém:

```python
import boto3
from datetime import datetime

def classify_change(diff: str) -> str:
    # ... lógica pura de classificação ...
    summary = boto3.client("bedrock-runtime").invoke_model(...)
    return summary["label"]
```

Diagnóstico: duas violações. `boto3` não pertence a `domain/`; e a
chamada ao Bedrock é I/O, portanto infra. A lógica de classificação
está misturada com a chamada de modelo.

Refatoração aplicando a regra:

```python
# domain/change_detection.py  (puro)
from typing import Protocol

class AIClient(Protocol):
    def summarize(self, text: str) -> str: ...

def classify_change(diff: str, ai: AIClient) -> Classification:
    summary = ai.summarize(diff)
    return _classify_from_summary(summary)
```

```python
# infrastructure/ai_client.py  (borda)
class BedrockAIClient:
    def __init__(self) -> None:
        self._client = boto3.client("bedrock-runtime")
    def summarize(self, text: str) -> str:
        resp = self._client.invoke_model(...)
        return resp["label"]
```

Agora `classify_change` é testável offline com um fake AIClient, e
trocar Bedrock por OpenAI é um diff em um único arquivo de infra.

## Sources

- [[docs/rules/Layering e Dependency Direction]]
- Síntese direta da regra do Danilo. Preserva o princípio raiz
  ("dependências apontam sempre para o domínio"), as quatro Regras
  Fundamentais, a regra da seta invertida, e o teste de fumaça do
  documento original.

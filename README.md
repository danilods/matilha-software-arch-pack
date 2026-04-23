# matilha-software-arch-pack

> **You lead. Agents hunt.**
> Matilha companion pack — software architecture, Caminho C (opinions-from-practice).

17 skills distilled from Danilo-experience on the **Argos** (content-intelligence platform for Brazilian public-exam prep) and **Gravicode** (AI mentor ecosystem) projects. Not a literature summary — patterns that survived contact with production constraints, AWS bills, failure modes, and maintenance over multiple quarters.

## What makes this pack different

Unlike literature-based packs (ux/growth/harness/sysdesign), this pack is opinions-from-practice — same Caminho C discipline as `matilha-software-eng-pack`. The source is 5 curated architecture rules authored from real project decisions, not from Uncle Bob / Evans / Cockburn (though those books inspire the vocabulary).

Where `matilha-sysdesign-pack` answers *"how does a distributed system work?"* (Tan's NFR + case-study framework), this pack answers *"how do I actually organize my code when I'm building a system like Argos?"* — concrete layering, concrete event contracts, concrete dual-store decisions.

## What this pack covers

- **Dependency & layering** (3 skills): import direction (domain never depends on infra), Lambda chain shape (thin handlers, pure domain core), handler as adapter.
- **Event-driven patterns** (4 skills): fact vs command events, Event Gateway boundary (EventBridge bus for fan-out), ordered vs unordered decision, event schema as public contract.
- **Data architecture** (4 skills): dual-store source-of-truth (Postgres control + Dynamo hot-state), CDC over dual-write, hot state via status machine, projection rebuild discipline.
- **Bounded contexts** (3 skills): context by vocabulary, context without microservice-spaghetti, anti-corruption layer between contexts.
- **Escalabilidade disciplinada** (3 skills): Ticker vs rule-per-entity, pull over push orchestration, measure before scale.

## Install

### Claude Code (recommended: user scope)

Install at user scope so the pack is available in every workspace:

```
/plugin marketplace add danilods/matilha-software-arch-pack
/plugin install matilha-software-arch-pack@matilha-software-arch-pack --user
```

When [matilha core](https://github.com/danilods/matilha-skills) is also installed, its `matilha-compose` gateway auto-detects this pack via plugin-namespace inspection (`matilha-*-pack`) and injects pack-aware preamble into brainstorming sessions whenever user intent touches architecture decisions. The pack also works standalone — skills activate via their own descriptions when intent matches.

Or locally during development:

```
/plugin install /path/to/matilha-software-arch-pack
```

## Relationship to other matilha packs

- **`matilha-sysdesign-pack`** — overlapping territory (distributed systems NFRs + scaling). sysdesign is the Tan/industry lens; swarch is the Argos/Gravicode practitioner lens. 4 skills in this pack have `Complementa matilha-sysdesign-pack:*` disclosures at specific angles.
- **`matilha-software-eng-pack`** — sibling Caminho C pack. sweng covers day-to-day coding discipline (KISS, commits, docs); swarch covers structural architecture (layering, events, data). Same authoring style, different scope.
- **`matilha-harness-pack`** — 1 cross-reference. `swarch-lambda-chain-shape` complements `harness-orchestrator-workers` — same abstract shape (orchestrator + workers) on different substrates (AWS Lambda chain vs LLM agent architecture).

## Source attribution

- **Danilo de Sousa** — curated architecture rules derived from Argos + Gravicode practice. Source docs shipped with this pack under `docs/rules/`:
  - `Layering e Dependency Direction.md`
  - `Event-Driven Decoupling.md`
  - `Dual-Store Architecture.md`
  - `Bounded Contexts na Prática.md`
  - `Escalabilidade sem Prematuridade.md`

## Skills inventory

See [docs/skills-inventory.md](docs/skills-inventory.md).

## Contributing

Follows the matilha [skill-authoring-guide.md](https://github.com/danilods/matilha-skills/blob/main/docs/matilha/skill-authoring-guide.md) + [pack-authors.md](https://github.com/danilods/matilha-skills/blob/main/docs/matilha/pack-authors.md).

## License

MIT — see [LICENSE](LICENSE).

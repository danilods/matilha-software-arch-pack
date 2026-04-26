# Changelog

## [0.3.0] — 2026-04-26 — Wave 5h-trigger: deterministic trigger skill

### Added

- **`matilha-software-arch-trigger` skill** — independent activation surface for software-architecture domain. Keyword-rich description (hexagonal, DDD, bounded context, event sourcing, CQRS, outbox, saga, clean architecture, handler as adapter, dependency direction, domain model, aggregate, CDC, pipeline, ports and adapters, etc.) ensures pack skills enter the conversation whenever the domain appears in user prompts.
- Complements `matilha-skills`'s routing table (`skills/matilha-compose/routing-table.md`); together they form Wave 5h's Maximum Deterministic Activation surface.

### Notes

- Fully additive: existing 17 skills untouched. Pack continues to work standalone or with `matilha-skills` (Matilha core).
- Trigger is a routing surface, not a craft skill — emits a compact domain acknowledgment and hands off to the most relevant pack skill via the Skill tool.
- No behavior change when pack is uninstalled: trigger emits a `/matilha-install` nudge and yields to default flow.

## [0.1.0] — 2026-04-23 — Wave 5h: initial release

### Added

- 17 skills distilled from Danilo-experience on Argos + Gravicode:
  - **Dependency & layering** (3): `swarch-dependency-direction`, `swarch-lambda-chain-shape`, `swarch-handler-as-adapter`.
  - **Event-driven patterns** (4): `swarch-fact-vs-command-events`, `swarch-event-gateway-boundary`, `swarch-ordering-decision`, `swarch-event-schema-contract`.
  - **Data architecture** (4): `swarch-dual-store-source-of-truth`, `swarch-cdc-over-dual-write`, `swarch-hot-state-via-status-machine`, `swarch-projection-rebuild-discipline`.
  - **Bounded contexts** (3): `swarch-context-by-vocabulary`, `swarch-context-without-microservice`, `swarch-acl-between-contexts`.
  - **Escalabilidade disciplinada** (3): `swarch-ticker-vs-rule-per-entity`, `swarch-pull-over-push-orchestration`, `swarch-measure-before-scale`.

- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` — canonical owner + metadata + plugins[] schema (aligned with Wave 5d.1 marketplace discipline).
- `README.md` — pack overview with install, companion-pack relationships, overlap disclosures, source attribution.
- `docs/rules/` — 5 architecture rules (Caminho C source, 981 lines total) shipped with the pack for transparency.
- `docs/skills-inventory.md` — full inventory with triggering intents + key principles + source-rule mapping.

### Overlap disclosures

- `swarch-dual-store-source-of-truth` **Complementa** `matilha-sysdesign-pack:sysdesign-dual-write-event-sourcing` — sysdesign treats dual-write as anti-pattern + 3 alternatives; swarch focuses on Postgres-SoT + Dynamo-hot-state case via CDC.
- `swarch-event-gateway-boundary` **Complementa** `matilha-sysdesign-pack:sysdesign-event-streaming-kafka` — sysdesign decides Kafka vs alternatives; swarch designs the Event Gateway (EventBridge bus + rules) for fan-out between bounded contexts.
- `swarch-ticker-vs-rule-per-entity` + `swarch-pull-over-push-orchestration` **Complementam** `matilha-sysdesign-pack:sysdesign-scalability-horizontal-vs-vertical` — sysdesign is structural; swarch brings concrete Argos case (Ticker GSI + ecs:RunTask failure + golden signals ritual).
- `swarch-lambda-chain-shape` **Complementa** `matilha-harness-pack:harness-orchestrator-workers` — same abstract shape, different substrates (AWS Lambda vs LLM agents).

### Notes

- First Wave 5h release. Caminho C discipline: 2-layer distillation (rule → skill), preserves Danilo's voice (same as matilha-software-eng-pack precedent).
- Rules docs shipped with the pack so readers can see the source behind the skills — full transparency on opinions-from-practice.

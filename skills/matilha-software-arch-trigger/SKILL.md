---
name: matilha-software-arch-trigger
description: "Use when the user mentions hexagonal architecture, DDD, domain-driven design, bounded context, event sourcing, CQRS, CDC, outbox pattern, saga, clean architecture, handler as adapter, dependency direction, domain model, aggregate, ports and adapters, or any topic related to software architecture patterns and long-term structural decisions. Fires independently of compose to ensure matilha-software-arch-pack skills activate whenever software architecture domain appears."
category: software-arch-pack
version: "1.0.0"
---

## When this fires

User prompt mentions any keyword from matilha-software-arch-pack's domain: hexagonal architecture, DDD, domain-driven design, bounded context, event sourcing, CQRS, CDC, outbox pattern, saga, clean architecture, handler as adapter, dependency direction, domain model, aggregate, ports and adapters.

This skill fires independently of `matilha:matilha-compose` and `CLAUDE.md` activation rules — its keyword-rich description is the activation surface for the Skill tool's matcher. It guarantees that whenever the user prompt touches the software-architecture domain, at least one matilha-software-arch-pack skill enters the conversation.

## Execution Workflow

1. **Pack presence check.** Inspect the ambient skill list for skills in the `matilha-software-arch-pack:*` namespace. Examples: `matilha-software-arch-pack:swarch-context-by-vocabulary`, `matilha-software-arch-pack:swarch-handler-as-adapter`, `matilha-software-arch-pack:swarch-cdc-over-dual-write`.

2. **Pack installed path.** If ≥1 skill from `matilha-software-arch-pack` is visible:
   - Identify the user's sub-intent within the architecture domain (drawing bounded contexts, deciding fact-vs-command events, choosing CDC over dual-write, structuring lambda chains, framing dependency direction, picking pull vs push orchestration, etc.).
   - Emit a compact domain acknowledgment (≤2 lines — no full sigil, that belongs to compose). Example: `Software architecture domain detected. Pulling matilha-software-arch-pack guidance for <sub-intent>.`
   - Invoke the most relevant pack skill via the Skill tool. Prefer one specific skill over listing many.

3. **Pack not installed path.** If no `matilha-software-arch-pack` skills are visible:
   - Emit: "Software Architecture skills not installed. Run `/matilha-install` and select `software-arch` to add 17 architecture skills."
   - Proceed with default flow (`matilha:matilha-compose` if visible, otherwise `superpowers:brainstorming`).

## Rules

- Do NOT emit the full compose sigil. The sigil belongs to compose; this skill emits a compact pack-specific acknowledgment.
- Do NOT block on pack absence — emit the nudge and continue with the default flow.
- Prefer invoking a specific pack skill over listing all skills.
- This trigger is a routing surface, not a craft skill — hand off to the chosen pack skill quickly.

## Why this exists

Wave 5h adds deterministic activation surfaces to compensate for probabilistic Skill-tool matching. Compose's routing table covers the central path; this trigger covers prompts that bypass compose (e.g., when CLAUDE.md is absent, or when a domain keyword appears mid-conversation without a compose-classifiable phase). Together they form the Maximum Activation guarantee for matilha-software-arch-pack.

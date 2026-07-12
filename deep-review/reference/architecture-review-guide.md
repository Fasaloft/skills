# Architecture Review Guide

Reminders for reviewing structure and design: SOLID, coupling/cohesion, layering, module boundaries, and over-engineering. The reviewer knows these concepts — this file lists the signals to match against a diff and the questions to ask.

## Contents

- [SOLID](#solid)
- [Anti-Patterns](#anti-patterns)
- [Coupling & Cohesion](#coupling--cohesion)
- [Layering](#layering)
- [Module Boundaries (Modulith)](#module-boundaries-modulith)
- [Patterns & Over-Engineering](#patterns--over-engineering)
- [Structure & Size](#structure--size)
- [Review Checklist](#review-checklist)

---

## SOLID

- **SRP** — one reason to change; job describable without "and". Signals: `...Manager`/`...Handler`/`...Processor` names, class >300 lines / component >200, grab-bag hooks, methods on unrelated data. Frontend split: data in hooks, logic in pure functions, presentation via props, thin container. Backend: repository / domain / thin service. **Testability is the litmus test** — hard to test usually *is* the SRP violation.
- **OCP** — new variants via composition/polymorphism/config, not by editing existing code. Signals: growing type-switches, `instanceof` checks scattered, a component gaining a boolean/`renderX` prop per case. Ask: "adding a new X type — which files change?"
- **LSP** — subtypes honor the base contract. Signals: empty/`NotImplemented` overrides, callers checking concrete types, downcasts.
- **ISP** — no client forced to depend on methods it doesn't use. Signals: >5–7 method interfaces, `IManager`/`IService` names, implementers with dead methods.
- **DIP** — high-level code depends on abstractions it owns. Signals: business logic `new`-ing concrete infrastructure, hardcoded config/connection strings in domain code, dependencies that can't be substituted in tests.

## Anti-Patterns

Flag on sight: **God Object** (one class knows/does too much), **Big Ball of Mud** (no boundaries, anything calls anything), **Copy-Paste Programming**, **Boat Anchor** (unused "we might need it" code — delete, YAGNI), **Golden Hammer** (same pattern for every problem), **Lava Flow** (untested untouchable code being extended instead of addressed).

## Coupling & Cohesion

- Prefer **data coupling** (pass only what's needed) and **message/event coupling**; flag **control coupling** (behavior flags like `isAdmin=true`), **common coupling** (shared mutable globals), and **content coupling** (reaching into another module's internals — worst).
- Stamp coupling smell: passing a whole object where two fields would do.
- Cohesion: all members of a unit should serve one task; a class whose methods form disconnected groups (LCOM ≥ 2) is several classes in a trench coat.
- Ask: "How many modules does this depend on — reducible?" · "How many places does changing this affect?"

## Layering

**Dependencies point inward only**: frameworks/drivers → adapters → application → domain. The domain layer imports no database, HTTP, or framework code — it defines interfaces; infrastructure implements them.

Flag: domain entities importing infrastructure; controllers containing business logic; UI calling repositories directly; business logic and data access mixed in one unit; config/secrets hardcoded instead of centralized.

## Module Boundaries (Modulith)

For modular monoliths and module-per-feature monorepos. The architecture erodes one convenient import at a time — every violation makes the next look normal.

**Core rule: modules communicate only through each other's public API.** Everything else is private, even if the language allows the import. If module B needs module A's internals, either A promotes it to its API deliberately, or the code moves to a shared module — both are explicit design decisions, never a deep import.

Check in a PR:

1. **New cross-module imports** target the other module's public surface — no `internal`/`impl`/deep paths. Fast scan: `git diff origin/main...HEAD | grep '^+' | grep -E '\.(internal|impl)\.|modules/\w+/(?!index)'`
2. **Dependency direction & cycles** — new edges follow the documented direction (feature → shared, never shared → feature); a new cycle is 🔴 blocking, even a "temporary" one.
3. **Data ownership** — no module reads/writes another module's tables directly (SQL, repository, or entity reuse); cross-module data goes through the owner's API or events.
4. **Events vs calls** — deliberate choice; flag synchronous work hidden in listener chains and events published inside another module's transaction.
5. **Transaction scope** — a transaction spanning two modules has silently fused them; require eventual consistency or an explicit decision.
6. **Shared module hygiene** — moving code to `shared`/`common` to dodge a boundary check is the same violation; shared holds genuinely cross-cutting, dependency-free code only.
7. **Enforcement moves with the code** — new modules/edges registered in ArchUnit/Konsist/Spring Modulith/ESLint-boundaries/Nx config in the same PR. If enforcement exists and CI is green, review only what it can't see (data ownership, transaction scope, event semantics).
8. **New module hygiene** — own docs, deliberately small public API, enforcement registration, own integration tests.

Severity: internals import, shared-table access, new cycle → 🔴 blocking. Cross-module transaction, ad-hoc API growth, business logic drifting into shared, missing docs/registration → 🟡 important.

## Patterns & Over-Engineering

- A pattern must solve a **present** extensibility problem. "Patternitis" signals: Strategy+Factory+Registry replacing a simple if/else; interfaces with one implementation; abstraction layers "for later"; newcomers needing a map to find the logic.
- Ask: "What problem does this pattern solve here? What would the code look like without it — and what breaks?"
- Extension points (hooks, events, config) beat hardcoded call chains **when variation is actually expected** — otherwise they're Speculative Generality.
- Singleton → prefer an injected instance. Observer → only when one-to-many is real; a direct call is simpler.

## Structure & Size

- Organize by **feature/domain**, not by technical layer (`controllers/`, `services/` mixing every domain).
- Rules of thumb (split signals, not laws): file <300 lines, function <50, params <4, nesting <4 levels.
- Names: conventional, behavior-reflecting; rename when behavior changes.

## Review Checklist

- [ ] Dependencies point the right way (inward; feature → shared); no new cycles
- [ ] Domain/business logic free of framework, DB, and HTTP imports
- [ ] SOLID signals checked on new/changed units; every unit testable in isolation
- [ ] Cross-module imports hit public APIs only; no shared tables; no cross-module transactions
- [ ] Boundary-enforcement config updated with new modules/edges
- [ ] No god objects, dead "for later" code, or one-implementation interfaces
- [ ] Abstractions justified by a present need, not speculation
- [ ] Organized by feature/domain; size rules of thumb respected

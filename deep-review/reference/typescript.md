# TypeScript/JavaScript Code Review Guide

Reviewer reminder sheet: signals to flag in a TS/JS diff and the expected fix. Assumes fluent TypeScript — no tutorials.

## Contents

- [Type Safety](#type-safety)
- [Generics](#generics)
- [Advanced Types](#advanced-types)
- [Strict Mode / tsconfig](#strict-mode--tsconfig)
- [Async](#async)
- [Immutability](#immutability)
- [ESLint](#eslint)
- [Review Checklist](#review-checklist)

## Type Safety

- `any` in a diff (param, return, cast, generic arg) → replace with a proper interface, or `unknown` + a real type guard. `as` is not a guard.
- `value as X` on a union or external data → demand narrowing instead: `Array.isArray`, `typeof`, `instanceof`, `in`, or a user-defined `x is T` predicate.
- Note: since TS 4.9, `in` checks narrow `unknown` — a guard like this needs no assertion:

  ```typescript
  typeof data === 'object' && data !== null && 'value' in data && typeof data.value === 'string'
  ```

- Object literals whose fields should be literal types (config, method names, action types) widened to `string` → add `as const`.
- `as SomeType` used to make a literal "fit" a type → prefer `satisfies SomeType` (keeps the narrower inferred type, still checks shape); `as` silently permits missing/wrong members:

  ```typescript
  const cfg = { method: 'GET' } satisfies RequestConfig; // cfg.method stays 'GET'
  const bad = {} as RequestConfig;                        // compiles, lies
  ```

- `@ts-ignore` → must be `@ts-expect-error` (with a reason), or better, fixed.
- Non-null `!` assertions → acceptable only with an obvious local invariant; otherwise flag.

## Generics

- Duplicated functions differing only in element type → one generic.
- Unconstrained `<T>` that indexes/accesses properties → add a constraint (`K extends keyof T`, `T extends {...}`).
- Generic type params that callers will usually omit → give a default (`<T = unknown>`).
- Hand-rolled shapes that mirror built-ins → use `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `ReturnType`, `keyof`.
- Over-genericized code (type gymnastics for one call site) → flag; KISS.

## Advanced Types

- Repetitive derived types → conditional types (`T extends ... ? ... : ...`, `infer`) or mapped types (`[K in keyof T]`, key remapping with `as \`get${Capitalize<...>}\``).
- Stringly-typed event names / routes / keys → template literal types (`` `on${Capitalize<EventName>}` ``, `` `/api/${string}` ``).
- Unions of object shapes handled by unsafe casts or optional-everything → discriminated union with a discriminant field (`type`/`success`/`kind`) and exhaustive `switch` (add a `never` default check when a case might be missed).
- Result/error flows modeled as `{ data?, error? }` → prefer `{ success: true; data } | { success: false; error }`.

## Strict Mode / tsconfig

- New or edited tsconfig without `"strict": true` → flag. `strict` already implies noImplicitAny, strictNullChecks, strictFunctionTypes, strictBindCallApply, strictPropertyInitialization, noImplicitThis, useUnknownInCatchVariables — don't ask for those individually.
- Recommended extras (not implied by strict): `noUncheckedIndexedAccess`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `noPropertyAccessFromIndexSignature`.
- `exactOptionalPropertyTypes` distinguishes a missing property from one set to `undefined` — nice-to-have, but expect third-party type friction; not a must-enable.
- With `noUncheckedIndexedAccess`, `arr[0]` / `obj[key]` is `T | undefined` — code that uses an indexed value without an `undefined` check (or a justified `!`) is a bug waiting.

## Async

- `await fetch(...)` / `response.json()` without checking `response.ok` or wrapping errors → flag; a non-2xx is not a rejection.
- `catch (error)` that assumes `Error` → narrow with `instanceof Error` before `.message` (`useUnknownInCatchVariables` makes it `unknown`).
- Rethrown/wrapped errors that drop the original context → include the underlying message or `cause`.
- `Promise.all` where partial failure should be tolerated (batch fetches) → `Promise.allSettled` and split fulfilled/rejected.
- Effect/handler firing a fetch per input change without cancellation → race condition; require `AbortController` (`signal` + cleanup `abort()`, swallow `AbortError` only).
- Floating promises: a returned Promise ignored at a call site → `await`, `.catch(...)`, or explicit `void`.
- Async callback passed where a void callback is expected — classic:

  ```typescript
  items.forEach(async (item) => { await process(item); }); // fire-and-forget, errors lost
  ```

  → sequential `for...of` with `await`, or `await Promise.all(items.map(process))`.

## Immutability

- In-place mutation of a parameter (`arr.sort(...)`, `arr.push`, property writes) → copy first (`[...arr].sort(...)`) or return new objects via spread.
- Params that should never be mutated → type as `readonly T[]` / `Readonly<T>`; deep structures may warrant a `DeepReadonly` mapped type.
- Generic functions meant to preserve tuple/literal types → `T extends readonly string[]` + caller `as const`.

## ESLint

- Config should extend `tseslint.configs.recommendedTypeChecked` (or `strictTypeChecked`) with `projectService: true` — flat config in typescript-eslint v8.
- Key rules to expect enabled: `no-explicit-any`, the `no-unsafe-*` family, `no-floating-promises`, `no-misused-promises`, `await-thenable`, `explicit-function-return-type` (warn), `consistent-type-imports`, `prefer-nullish-coalescing`, `prefer-optional-chain`.
- Diffs that disable these rules inline without justification → flag.

## Review Checklist

### Type System
- [ ] No `any` (use `unknown` + type guards); no unsafe `as` on unions/external data
- [ ] Interfaces/types complete and meaningfully named
- [ ] Literal types via `as const` where widening loses information; `satisfies` over `as` for shape-checking literals
- [ ] Unions narrowed correctly; discriminated unions for variant shapes
- [ ] Built-in utility types used instead of hand-rolled equivalents

### Generics
- [ ] Appropriate constraints (`extends`, `keyof`); sensible defaults
- [ ] Not over-genericized

### Strict Mode
- [ ] `strict: true`; `noUncheckedIndexedAccess` enabled and indexed access checked
- [ ] No `@ts-ignore` (use `@ts-expect-error`)

### Async
- [ ] Errors handled (`response.ok`, `instanceof Error` narrowing)
- [ ] No floating promises; no `async` callbacks into void-expecting APIs (`forEach`)
- [ ] `Promise.allSettled` when partial failure is acceptable
- [ ] Race conditions cancelled via `AbortController`

### Immutability
- [ ] No parameter mutation; spread/copy before sort/mutate
- [ ] `readonly` modifiers on params that must not change

### ESLint
- [ ] `recommendedTypeChecked`/`strictTypeChecked` base; no unjustified inline disables
- [ ] `consistent-type-imports` respected

# React & Frontend Review Guide

The single frontend reference: hooks/Effects, memoization, composition, component-library reuse, React 19 / RSC, TanStack Query v5, integration testing, and the reviewer patterns that come up most in real PRs. Reminder-style — signals to match against the diff, plus the version-specific gotchas worth spelling out.

## Contents

- [Discover the Project's Platform First](#discover-the-projects-platform-first)
- [Reuse the Platform](#reuse-the-platform)
- [Hooks & Effects](#hooks--effects)
- [Memoization](#memoization)
- [Component Design & Composition](#component-design--composition)
- [Question Necessity](#question-necessity)
- [Server Components & React 19 Forms](#server-components--react-19-forms)
- [Suspense & Streaming](#suspense--streaming)
- [TanStack Query v5](#tanstack-query-v5)
- [Integration Testing](#integration-testing)
- [UX, Naming, Docs](#ux-naming-docs)
- [Review Checklist](#review-checklist)

---

## Discover the Project's Platform First

Every check below runs against *this repo's* platform — identify it before judging the diff: the component library (internal package, shadcn/ui, MUI…), the data-access layer (generated clients, TanStack Query, SWR, RTK Query…), form/state/utility conventions, and where integration tests live. Fastest way: open two or three existing screens similar to the one being changed.

## Reuse the Platform

The single most common review point: the change re-implements something the platform provides. Three layers, in order:

1. **Component library.** A raw element styled to look like a library component is a defect even when it renders identically — duplicated tokens, drift on restyle, re-implemented a11y. Signals: raw `<button>`/`<input>`/`<label>` with design-system-mimicking styles; inline hex/px values duplicating tokens; hand-rolled modal/tooltip/menu/tabs (overlays carry focus traps, ESC, scroll lock, ARIA — hand-rolled versions get some of it wrong); forms re-deriving the library's Field/Label/Input composition with manual `htmlFor` wiring and forgotten `aria-invalid`.
2. **Data-access layer.** Data goes through the repo's standard clients/hooks and their built-in state — no raw `fetch` + hand-rolled `loading` flags (`isPending` exists), no reimplemented pagination (infinite query exists), no parallel state library.
3. **Existing hooks and utilities** — `useFieldArray` for form arrays, shared parsers, test utils. Before approving new logic, be able to say: *"I checked the library / the data layer / existing hooks — genuinely nothing to reuse."*

Also respect **design-system semantics**: the right component for the meaning (a backdrop-less dialog is a popover; a single-select of options is a dropdown, not hacked radios), tokens over ad-hoc literals, the component's own states/variants over overrides (no manual `opacity` on disabled, no `size`/casing overrides without reason). Verify against Storybook/workbench if the repo has one, including edge cases (long text, two-line cells, empty states).

**Import hygiene:** import from the library's public entry point (barrel) — entry points can carry side-effect registrations (module augmentation, CSS) a deep import silently skips. No re-export chains; fix the import path instead. Genuinely new UI belongs in the library/shared layer as a documented component, not inline in a feature.

## Hooks & Effects

- Hooks at top level, never conditional (exception: React 19 `use()`); complete dependency arrays; cleanup for subscriptions/timers/requests (fetch-in-Effect also needs race handling — a `cancelled` flag or AbortController).
- **An Effect exists to synchronize with a system outside React.** The tell for a wrong Effect: all deps are React values and the body calls `setState`. Route each case to its real tool:
  - Derived/transformed data → compute during render (or `useMemo` if measurably expensive) — never Effect + state (extra render, stale window)
  - Resetting state on a prop change → `key` remount, not an Effect
  - Reacting to a user action → the event handler that caused it, not an Effect watching state
  - Effect chains (one Effect's `setState` triggering another) → derive during render
  - Subscribing to an external store → `useSyncExternalStore`
  - Data fetching → the data library / route loader (caching + race handling included)
  - App-once initialization → module scope, not a mount Effect (runs per instance, twice in StrictMode)
- A component with many Effects is a design smell, not a style issue.

## Memoization

If the React Compiler is enabled (stable since Oct 2025), manual `useMemo`/`useCallback`/`React.memo` is mostly redundant — flag *newly added* manual memoization. Otherwise: memoize deliberately — `useMemo` on constants is noise, `useCallback` is useless unless the function reaches a memoized child or a dependency array, and it memoizes nothing if a dependency changes every render. Memoization pays when paired: `React.memo` child + stable props.

## Component Design & Composition

- No components defined inside components (new identity every render → remount/state loss); no inline object/function props to memoized children.
- React 19: `ref` is a regular prop — no new `forwardRef`; `<Context value>` replaces `<Context.Provider>`.
- **Prop explosion → composition.** Signals: accumulating boolean flags (`showX`/`hideY`/`withZ`), multiple `renderX`/slot props, a config object gaining keys, every new variant editing the component. Fix: compound components (`Dialog.Header`/`Dialog.Body`/`Dialog.Footer`) sharing state via context — consumers compose new arrangements without the component changing (open/closed).
- Rule of thumb: **props for data and simple config; composition for structure and content.** Slots are fine for small fixed-shape leaves; pages and large reusable parts need real composition.
- Single responsibility: data in hooks, logic in pure functions, presentation via props, thin containers — see [Architecture Guide](architecture-review-guide.md#solid).
- State: keep it close to use; derive > store; `useReducer` for complex transitions.

## Question Necessity

Default reviewer stance is Socratic: every new prop, re-export, wrapper, abstraction, or side effect must answer "why do we need this?". Weigh sustainability ("does this hold at 10+ variants?"), prefer the smallest change that solves the problem, and flag unrelated churn.

## Server Components & React 19 Forms

- No hooks/event handlers in Server Components; `await` data there directly. `'use client'` only where interactivity lives — on a layout it drags the whole tree client-side; keep client components at the leaves.
- `useActionState` replaces hand-rolled `isPending`/`error`/`data` triplets.
- `useFormStatus` reads the **parent** form — must be called in a child component inside the `<form>`, never a sibling.
- `useOptimistic`: setter must run inside a transition/Action (React 19 warns otherwise); the UI snaps back to the source-of-truth when the action ends, so success must update the real state too. Not for critical operations (payments).
- Server Functions (`'use server'`) + `useActionState` replace client-side `fetch('/api/…')` in RSC apps.

## Suspense & Streaming

- Every `<Suspense>` needs an ErrorBoundary above it. Split boundaries by UX — one giant boundary makes fast content wait for slow. Skeletons > spinners.
- `use(promise)`: the Promise must come from a Suspense-compatible source (Server Component, framework, cache); a fresh Promise created during a Client Component render → "uncached promise" warning + refetch every render. `use(Context)` may be called conditionally, unlike `useContext`.

## TanStack Query v5

- `queryOptions()` — define key+fn once, reuse type-safely across `useQuery`/`prefetchQuery`/`getQueryData`.
- **queryKey must include every parameter the queryFn uses** — otherwise no refetch on change.
- `staleTime` defaults to 0 (refetch every mount) — set deliberately. `cacheTime` → `gcTime`.
- v5 semantics: `isPending` = no cached data; `isFetching` = request in flight; `isLoading` = both. Verbatim v4 `isLoading` migrations can change behavior.
- Mutations: invalidate related queries on success; use built-in `isPending`; `variables` gives simple optimistic UI without manual cache juggling (manual optimistic updates need `onError` rollback).
- `useSuspenseQuery`: no `enabled`/`placeholderData`; `data` is `T` (guaranteed); initial-load errors throw to the ErrorBoundary but **background-refetch errors land silently in `error`** — surface with `if (error && !isFetching) throw error;`. Conditional fetching → composition (parent early-returns, child queries unconditionally). Parallel queries → `useSuspenseQueries`.

## Integration Testing

New functionality needs an **integration test in its module**: render the feature wired-up (real children and hooks), mock only the network boundary (MSW or the repo's equivalent), assert user-visible behavior. A unit test on the hook alone doesn't cover the wiring.

- Mock the **network boundary**, not the module's own hooks — stubbing internals tests the mock.
- Behavior over implementation: `screen` + `*ByRole`/`*ByLabelText`, `userEvent` over `fireEvent`, `findBy*` for async content; assert visible outcomes, never internal state, classes, or call counts.
- **Assertions must be able to catch a regression** — pin concrete expected values; never re-derive the implementation's own formula (test and impl then change together and catch nothing).
- Cover happy path + API failure → visible error + empty/loading states + the validation branches the change introduces.
- Share mocks/test utils instead of redefining per `describe`; update visual tests when UI changes; co-locate tests with the module. E2E complements, never replaces, module integration tests.

## UX, Naming, Docs

- **Notifications:** follow the repo's policy — no per-call error toasts if errors surface automatically; no toasts for routine saves.
- **In-flight operations:** block or disable the trigger while an action runs (e.g. keep the dialog blocked until deletion completes).
- **No uncommunicated side effects:** a control does what its label says — if it now also mutates something else, narrow it or rename and communicate.
- **Names** conventional and behavior-reflecting; rename when behavior changes. Update the docs the repo maintains (module docs, README, ADRs); document shared components per repo convention (TSDoc, story, docs page).

## Review Checklist

**Platform & reuse**
- [ ] Platform identified (library, data layer, test conventions); nothing re-implements it
- [ ] Data via standard clients/hooks with built-in pending state; no raw fetch + hand-rolled loading
- [ ] Semantically correct components, tokens not literals, barrel imports, no re-export chains
- [ ] New UI added to the shared layer and documented — not a one-off

**Hooks & rendering**
- [ ] Effects only synchronize with external systems; derived state computed, `key` for resets, events in handlers
- [ ] Complete deps + cleanup on remaining Effects; no Effect chains
- [ ] Memoization deliberate (or left to the Compiler); no components-in-components; `ref` as prop
- [ ] Composition over prop bags; props for data, composition for structure

**RSC & data**
- [ ] `'use client'` at the leaves only; React 19 form hooks used per their placement/transition rules
- [ ] Suspense paired with ErrorBoundary, boundaries split by UX
- [ ] queryKey complete; deliberate staleTime; v5 `isPending` semantics; mutations invalidate; `useSuspenseQuery` quirks handled

**Tests & UX**
- [ ] Wired-up integration test in the module, mocked at the network boundary
- [ ] Happy + failure + empty covered; assertions pin concrete values, behavior not implementation
- [ ] Notification policy respected; in-flight handled; no uncommunicated side effects; names + module docs updated

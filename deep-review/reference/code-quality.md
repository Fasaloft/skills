# Code Quality Review Guide

Language-agnostic quality review: a baseline of classic code smells to match against any diff, plus expanded anti-patterns that come up constantly in PR review (reuse, leaky abstractions, parameter bloat, stringly-typed values, TOCTOU, no-op updates).

Two rules bind everything here:

- **The repo overrides.** A documented repo standard always wins; where it endorses something this guide would flag, suppress the finding.
- **Always a judgement call.** Each item is a labelled heuristic ("possible Feature Envy"), never a hard violation — and skip anything tooling (linter, typechecker, formatter) already enforces.

## Contents

- [Universal Bug Checks](#universal-bug-checks)
- [The Smell Baseline (Fowler)](#the-smell-baseline-fowler)
- [Code Reuse Review](#code-reuse-review)
- [Parameter Bloat](#parameter-bloat)
- [Leaky Abstractions](#leaky-abstractions)
- [Stringly-Typed Values](#stringly-typed-values)
- [Nested Conditionals](#nested-conditionals)
- [No-Op Updates](#no-op-updates)
- [TOCTOU Race Conditions](#toctou-race-conditions)
- [Overly Broad Operations](#overly-broad-operations)
- [Redundant State](#redundant-state)
- [Review Checklist](#review-checklist)

---

## Universal Bug Checks

Language-independent correctness checks for any diff (language-specific lists: [Common Bugs Checklist](common-bugs-checklist.md)):

- **Logic** — off-by-one in loops/slices, inverted boolean logic (De Morgan), missing null/undefined checks, wrong comparison operators, integer overflow, float equality comparisons, race conditions in concurrent paths
- **Resources** — unclosed connections/file handles, listeners and timers never removed, missing cleanup on the error path
- **Error handling** — swallowed exceptions (empty catch), overly generic catches hiding specific failures, missing propagation, missing finally/cleanup

---

## The Smell Baseline (Fowler)

A fixed set of code smells (*Refactoring*, ch. 3) that applies even when a repo documents nothing. Each reads *what it is* → *how to fix*; match it against the diff:

- **Mysterious Name** — a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code** — the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy** — a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps** — the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession** — a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches** — the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery** — one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change** — one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality** — abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains** — long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man** — a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest** — a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

The sections below expand the smells and anti-patterns that appear most often in practice, with concrete review points.

---

## Code Reuse Review

Before accepting new code, search the existing codebase (neighboring files, shared/utils modules) for utilities that already do the job.

```javascript
// ❌ Hand-rolled debounce — the project already has lodash or utils/debounce.ts
function debounce(fn, ms) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}

// ✅ Use the existing utility
import { debounce } from "@/utils/debounce";
```

**Review points:** Does the new function overlap in name or functionality with an existing utility? Can inline logic become a call to an existing module? (See also *Duplicated Code* in the baseline.)

---

## Parameter Bloat

```typescript
// ❌ 6+ positional parameters, growing with every requirement
function renderWidget(
  title: string, width: number, height: number,
  theme: string, collapsible: boolean, icon: string
) { ... }

// ✅ Options object with types and defaults
interface WidgetOptions {
  title: string;
  width?: number;
  height?: number;
  theme?: "light" | "dark";
  collapsible?: boolean;
  icon?: string;
}
function renderWidget(options: WidgetOptions) { ... }
```

**Review points:** ≥4 parameters → options object/dataclass. A new boolean flag → consider an enum or strategy. Mutually exclusive parameters (`enable_x`, `disable_y`) are a design smell. (See also *Data Clumps*.)

---

## Leaky Abstractions

```python
# ❌ Returning an internal ORM object — callers are forced to know about SQLAlchemy
def get_users():
    return session.query(User).filter(User.active == True).all()

# ✅ Return a domain object, hiding the persistence layer
def get_active_users() -> list[UserDTO]:
    return [UserDTO.from_row(r) for r in user_repo.find_active()]
```

**Review points:** Does the return type leak the underlying implementation (ORM, HTTP client, file format)? Does a component/function depend on an external system's raw data structures instead of a domain type behind an adapter?

---

## Stringly-Typed Values

```typescript
// ❌ Raw string event names — typos won't raise errors
emitter.emit("userCreated", data);
emitter.on("usercreated", handler); // bug: typo

// ✅ Constants, enum, or union type
const Events = { USER_CREATED: "userCreated" } as const;
emitter.emit(Events.USER_CREATED, data);
```

**Review points:** Strings used where an enum/union/constant exists? Event names, action types, status values scattered across files? (See also *Primitive Obsession*.)

---

## Nested Conditionals

```python
# ❌ Nested if, 3+ levels deep
def process(order):
    if order is not None:
        if order.items:
            for item in order.items:
                if item.price > 0:
                    ...

# ✅ Early return + guard clauses
def process(order):
    if not order or not order.items:
        return
    for item in order.items:
        if item.price <= 0:
            continue
        ...
```

**Review points:** Ternaries nested ≥2 levels or if/else ≥3 levels deep → lookup table, early return, or match. Ternary chains mapping value→label → a lookup table. (See also *Repeated Switches*.)

---

## No-Op Updates

```python
# ❌ Every loop iteration writes to the DB — even when the value hasn't changed
for item in items:
    item.status = compute_status(item)
    session.commit()

# ✅ Write only when it changes
for item in items:
    new_status = compute_status(item)
    if item.status != new_status:
        item.status = new_status
        session.commit()
```

**Review points:** Do polling/interval/event handlers update state unconditionally (causing renders/writes with identical data)? Do DB writes check for actual changes?

---

## TOCTOU Race Conditions

```python
# ❌ Check-then-act — the file may be deleted/created in between
if os.path.exists(path):
    with open(path) as f:
        data = f.read()

# ✅ Operate directly + handle the exception
try:
    with open(path) as f:
        data = f.read()
except FileNotFoundError:
    data = None
```

**Review points:** Replace `if exists → operate` with `try operate → catch`. Multi-step state changes (check balance → deduct) belong in a transaction/lock. In async code, any `await` between check and act is a race window.

---

## Overly Broad Operations

```typescript
// ❌ Loading all rows then filtering in memory
const allItems = await db.query("SELECT * FROM orders");
const pending = allItems.filter(o => o.status === "pending");

// ✅ Filter at the database layer
const pending = await db.query(
  "SELECT * FROM orders WHERE status = ?", ["pending"]
);
```

**Review points:** Entire collection/file read to use a small part? Push filtering down to the storage layer; use pagination/limits; read the first line, not the whole file.

---

## Redundant State

```typescript
// ❌ Storing fullName alongside firstName + lastName — can go stale
interface User { firstName: string; lastName: string; fullName: string; }

// ✅ Derive it
const fullName = `${user.firstName} ${user.lastName}`;
```

**Review points:** Fields derivable from other fields? Cached values without an invalidation mechanism? Derive > store.

---

## Review Checklist

- [ ] **Smell baseline**: diff matched against the Fowler smells; findings labelled as judgement calls, repo standards respected
- [ ] **Reuse**: searched for existing utilities/helpers before accepting new ones
- [ ] **Parameters**: ≤3 params, options object beyond that; no boolean-flag creep
- [ ] **Abstraction boundaries**: return types don't expose internals (ORM, HTTP client, file format)
- [ ] **Type safety**: no magic strings where an enum/constant/union exists
- [ ] **Conditional depth**: ternary nesting ≤1, if/else nesting ≤2; guard clauses used
- [ ] **No-op guard**: polling/interval/event handlers and DB writes check for actual change
- [ ] **TOCTOU**: `try operate → catch` instead of `if exists → operate`; atomic multi-step changes
- [ ] **Data precision**: not reading a whole collection/file for a subset
- [ ] **Redundant state**: nothing stored that can be derived

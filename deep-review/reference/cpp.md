# C++ Code Review Guide

> Reviewer reminder for C++ diffs (C++17/20/23): ownership/RAII, lifetime and dangling, move semantics, undefined behavior,
> exceptions/noexcept, concurrency, API shape, integer conversions, performance, headers/ODR. Terse by design — signal in the
> diff → what to flag. Qt-specific concerns (QObject, signals/slots, QML) → [Qt & QML Guide](qt-qml.md).
> Distilled from the C++ Core Guidelines, clang-tidy `bugprone-*` checks, Abseil Tips of the Week, and PVS-Studio's UB guide.

## Contents

- [Ownership and RAII](#ownership-and-raii)
- [Lifetime and Dangling](#lifetime-and-dangling)
- [Move Semantics](#move-semantics)
- [Undefined Behavior Hotspots](#undefined-behavior-hotspots)
- [Exceptions and noexcept](#exceptions-and-noexcept)
- [Concurrency](#concurrency)
- [API Shape and Const-Correctness](#api-shape-and-const-correctness)
- [Integers and Conversions](#integers-and-conversions)
- [Performance](#performance)
- [Headers, Linkage, ODR](#headers-linkage-odr)
- [Review Checklist](#review-checklist)

## Ownership and RAII

- Raw owning `new`/`delete` in application code → flag. Owners are `std::unique_ptr` (default), `std::shared_ptr` (only for genuinely shared lifetime), or containers. Raw pointers/references = non-owning observers only.
- Manual acquire/release pairs (`open`/`close`, `lock`/`unlock`, `malloc`/`free`) → wrap in a RAII type; an early return or exception between the pair leaks.
- **Rule of zero first**: a class managing no resource should declare none of the five special members. If it declares *any* of destructor/copy/move, review all five — a user-declared destructor suppresses implicit moves, silently degrading every "move" into a copy.
- Polymorphic base without `virtual ~Base()` → `delete` through base pointer is UB. Either virtual destructor or `protected` non-virtual one.
- `std::make_unique`/`std::make_shared` over naked `new` (exception safety, single allocation for `make_shared`).
- `shared_ptr` cycles (parent↔child both `shared_ptr`) → leak; back-references are `weak_ptr` or raw.
- `.get()` handed to something that stores it → the smart pointer's lifetime no longer protects the user; flag stored raw pointers from `get()`.
- Signatures: take `unique_ptr<T>` by value = "I take ownership"; `shared_ptr<T>` by value = "I share ownership"; plain `T&`/`T*` = "I just use it". A function taking `const shared_ptr<T>&` that never copies it should take `T&`/`T*`.

## Lifetime and Dangling

- `std::string_view`/`std::span` **stored** (member, container, captured beyond the call) → flag; they are non-owning views, the buffer must provably outlive them. As parameters they're fine.

```cpp
std::string_view sv = name + "!";   // ❌ temporary string dies at end of statement — sv dangles
std::string_view first(const std::string& s);
auto v = first(get_name());          // ❌ view into a temporary return value
```

- Function returning reference/pointer to a local, or to a parameter taken by value → dangling.
- Lambda capturing by reference (`[&]`, `[&x]`) or `[this]` stored in a callback/queue/thread that outlives the scope → use-after-free. Capture copies, `shared_ptr`, or `weak_ptr` + lock in body. `[=]` still captures `this` implicitly (deprecated in C++20) — members are *not* copied.
- Iterator/reference invalidation: `vector`/`string` — any reallocation (push_back over capacity) and everything at/after an erase point; `unordered_*` — rehash invalidates iterators; `map`/`list` — only the erased element. Mutating a container inside a range-for over it → flag; use `std::erase_if` (C++20) or the erase-remove idiom.

```cpp
for (auto& x : vec)
    if (stale(x)) vec.erase(...);    // ❌ invalidates the range-for's iterators
std::erase_if(vec, stale);           // ✅
```

- Range-for over a temporary's member: `for (auto& x : make_config().items)` — the temporary `make_config()` dies before the loop body on pre-C++23 (fixed by P2718); flag unless the toolchain is C++23.
- `std::optional`: `*opt`/`opt->` without a prior check is UB, not a throw — `value()` throws, `operator*` doesn't.

## Move Semantics

- Use-after-move: reading a variable after `std::move(x)` was consumed → moved-from is valid-but-unspecified; only assign-over or destroy. (clang-tidy: `bugprone-use-after-move`.)
- `return std::move(local);` → pessimization: blocks NRVO; plain `return local;` elides or implicitly moves.
- `std::move` on a `const` object → overload resolution silently picks the **copy** constructor; nothing moves.
- Forwarding: `T&&` where `T` is a deduced template parameter is a forwarding reference → `std::forward<T>`, not `std::move`. `std::move` there wrecks lvalue callers.
- A moved-into sink parameter that's used but never moved on → the by-value copy was pointless; conversely a by-value sink param stored with `member = param;` instead of `member = std::move(param);` copies twice.

## Undefined Behavior Hotspots

- Signed integer overflow, shift ≥ bit-width, shift of negative values → UB, and optimizers exploit it (e.g. `if (x + 1 < x)` folded to `false`).
- Out-of-bounds indexing (`operator[]` on vector/array does not check), `front()`/`back()` on empty containers.
- Reading uninitialized variables — flag members without default member initializers and locals declared before their first assignment. Member initialization runs in **declaration order**, not initializer-list order; an initializer reading a later-declared member reads garbage (`-Wreorder`).
- Type punning via `reinterpret_cast`/union → strict-aliasing UB; use `std::memcpy` or C++20 `std::bit_cast`.
- `const_cast` used to actually write a const object → UB, not a style issue.
- Virtual calls in constructors/destructors dispatch statically (the derived part doesn't exist yet) — flag `virtual` calls in ctor/dtor expecting derived behavior.
- Default arguments on virtual functions bind statically to the pointer's type, not the object's → base and override must agree.
- Slicing: passing/catching/storing a polymorphic type **by value** (`void f(Shape s)`, `catch (std::exception e)`) chops off the derived part → pass by reference, `catch (const std::exception& e)`.

## Exceptions and noexcept

- Move constructor/assignment without `noexcept` → `vector` reallocation falls back to **copies** (`move_if_noexcept`); flag on any type stored in containers.
- `noexcept` on a function with a visible throwing path (allocation, `.at()`, user callbacks) → `std::terminate` at runtime, not a caught exception.
- Throwing destructors → `terminate` during unwinding; destructors are implicitly `noexcept` — cleanup that can fail needs an explicit `close()`-style member the caller can invoke and check.
- Multi-step acquisition without RAII in between → the steps before the throw leak. Every `new`/acquire followed by fallible code before ownership lands in a RAII object is a leak path.
- Error-code returns that callers can silently drop → `[[nodiscard]]` on the function or the error type.

## Concurrency

- Shared mutable data (even a lone `bool` flag) accessed from two threads without `std::atomic`/mutex → data race = UB, regardless of "it's just a flag". Torn/never-observed writes are real.
- Manual `mutex.lock()`/`unlock()` → flag; `std::lock_guard`/`std::scoped_lock` (multiple mutexes: `scoped_lock` locks deadlock-free). Two mutexes locked in different orders on different paths → deadlock.
- `condition_variable::wait` without a predicate → spurious wakeups and lost notifications:

```cpp
cv.wait(lock);                              // ❌ wakes spuriously; misses notify sent before wait
cv.wait(lock, [&] { return ready; });       // ✅ predicate re-checked under the lock
```

- `std::thread` never joined → `terminate` in its destructor; `.detach()` → flag: the thread outlives every lifetime guarantee, captures dangle at shutdown. Prefer C++20 `std::jthread` (auto-join, `stop_token` cooperative cancellation).
- Discarded `std::async` result → the temporary future's destructor **blocks**, making the call synchronous: `std::async(std::launch::async, work);` runs serially.
- Thread/task capturing locals or `this` by reference while outliving the scope → same dangling rules as lambdas above, now with a data race attached.
- Holding a lock while invoking user callbacks or emitting notifications → re-entrancy deadlock; call out after unlocking.
- `shared_ptr`: the control block is thread-safe, the pointee is **not**; concurrent reads/writes of `*ptr` still need synchronization.

## API Shape and Const-Correctness

- Parameter passing: cheap-to-copy (few words) → by value; otherwise `const T&`; sinks (stored) → by value + `std::move`; read-only string/array views → `std::string_view`/`std::span<const T>`.
- Return by value, trust RVO — out-parameters for "returning" values → flag; return a struct or `std::optional<T>` for maybe-values.
- Member functions that don't mutate → `const`; `mutable` only for caches/mutexes with unchanged logical state.
- Single-argument constructors without `explicit` → accidental implicit conversions in overload resolution; flag unless conversion is the intent.
- Plain `enum` (unscoped) in new code → `enum class`: no implicit int conversion, no namespace pollution. Switch over an enum: enumerate all cases, no `default:` — keeps `-Wswitch` alive when a value is added.
- Boolean parameters at call sites reading `f(true, false)` → suggest enum class or designated initializers on an options struct.

## Integers and Conversions

- Unsigned reverse loops: `for (size_t i = n - 1; i >= 0; --i)` → `i` wraps, infinite loop; `n - 1` with `n == 0` wraps to huge. Iterate down with signed index, reverse iterators, or `while (i-- > 0)`.
- Signed/unsigned comparison: `int i` vs `v.size()` promotes `i` to unsigned — `-1 < size()` is false. C++20 `std::cmp_less` family or consistent signed types (`ssize`).
- Narrowing in arithmetic used for allocation/indexing: `int` products overflowing before assignment to `size_t` (`bytes = w * h * 4` with `int` w/h).
- Implicit float↔int truncations and `double`→`float` narrowing in data paths → require explicit `static_cast` showing intent.

## Performance

- `auto` copies: `auto x = expensive()` and `for (auto item : container)` copy — `const auto&` (or `auto&&`) unless a copy is wanted.
- `map`/`unordered_map` `operator[]` on a **read path** default-inserts the key → `find()`/`contains()`/`at()`:

```cpp
if (counts["key"] > 0) …        // ❌ silently inserts {"key", 0}
if (auto it = counts.find("key"); it != counts.end() && it->second > 0) …  // ✅
```

- Repeated `push_back` with a known size → missing `reserve()`; string concatenation in loops → same.
- `emplace_back`/`try_emplace` over `push_back(T(...))` when constructing in place saves a move/copy — a nit, not a blocker.
- `std::regex` construction inside loops → recompiles per iteration; hoist it. (`std::regex` is slow generally — repeated hot use deserves a question.)
- Copying `shared_ptr` in hot paths/loop headers → atomic refcount traffic; pass `const shared_ptr&` or the raw `T&` downward.
- `std::endl` in loops → flushes every line; `'\n'`.
- Passing `std::function` by value in hot code / storing it when a template or `auto` callable would do → type-erasure + possible allocation.

## Headers, Linkage, ODR

- Non-inline global/static definitions in headers → ODR violations or per-TU duplicates; C++17 `inline` variables or define in one TU.
- Initialization order of globals across TUs is unspecified (static init order fiasco) → a global whose initializer reads another TU's global → flag; function-local `static` (initialized on first use, thread-safe) or `constinit`.
- File-local helpers in `.cpp` without `static`/anonymous namespace → external linkage, ODR collision risk with same-named helpers elsewhere.
- `using namespace` in headers → flag, always; it injects into every includer.
- Headers relying on transitive includes → include what you use; the transitive include disappearing breaks distant TUs.

## Review Checklist

### Ownership and RAII

- [ ] No raw owning `new`/`delete`; `unique_ptr` default, `shared_ptr` only for shared lifetime
- [ ] Every acquire/release pair wrapped in RAII (exception/early-return safe)
- [ ] Rule of zero; if any of dtor/copy/move declared, all five reviewed (dtor suppresses moves)
- [ ] Polymorphic bases have virtual (or protected non-virtual) destructors
- [ ] No `shared_ptr` cycles; back-references are `weak_ptr`
- [ ] Signatures encode ownership (smart pointer by value = transfer; `T&`/`T*` = use only)

### Lifetime

- [ ] No stored `string_view`/`span`/reference to a temporary or shorter-lived object
- [ ] Escaping lambdas capture by value / `shared_ptr` / `weak_ptr`, never `[&]` or bare `[this]`
- [ ] No container mutation while iterating; invalidation rules respected (`erase_if`)
- [ ] `optional` checked before `*`/`->`

### Moves

- [ ] No use-after-move; no `std::move` on `const` or on return values
- [ ] `std::forward` (not `move`) for forwarding references
- [ ] Sink params passed by value are moved into place (`member = std::move(param)`)

### UB

- [ ] No signed overflow/shift games; no OOB indexing; empty-container `front`/`back` guarded
- [ ] All members have initializers; init order = declaration order
- [ ] No `reinterpret_cast` type punning (→ `memcpy`/`bit_cast`); no `const_cast` writes
- [ ] No virtual dispatch expectations in ctors/dtors; no by-value slicing (params, `catch`)

### Exceptions

- [ ] Move ops `noexcept` (container reallocation); `noexcept` claims verified against throwing paths
- [ ] Destructors can't throw; fallible cleanup exposed as explicit member
- [ ] Droppable error returns marked `[[nodiscard]]`

### Concurrency

- [ ] All cross-thread shared state atomic or mutex-protected — including "simple" flags
- [ ] `lock_guard`/`scoped_lock`, consistent lock order, no manual `unlock`
- [ ] `condition_variable::wait` always takes a predicate
- [ ] No detached threads; `jthread`/join; no discarded `std::async` future (dtor blocks)
- [ ] No locks held across callbacks/notifications

### API and Integers

- [ ] Params: value for cheap/sink, `const&` otherwise, views for read-only buffers
- [ ] Return by value; `optional` over out-params; `explicit` single-arg ctors; `enum class`
- [ ] No unsigned wraparound loops; signed/unsigned comparisons via `cmp_*` or consistent types

### Performance

- [ ] `const auto&` in range-for; no `operator[]` map reads; `reserve` before bulk append
- [ ] No regex/`shared_ptr` copies/allocations in hot loops

### Headers

- [ ] No non-inline definitions or `using namespace` in headers; IWYU; anonymous namespace for file-local helpers
- [ ] No cross-TU global init dependencies (SIOF)

# Common Bugs Checklist

Per-language bug patterns for stacks **without a dedicated guide**. TypeScript/JavaScript → [TypeScript Guide](typescript.md); React → [React Guide](react.md); Kotlin → [Kotlin Guide](kotlin.md); Python → [Python Guide](python.md); AWS Lambda/serverless → [AWS Lambda Guide](aws-lambda.md); C++ → [C++ Guide](cpp.md); Qt/QML → [Qt & QML Guide](qt-qml.md); universal logic/resource/error checks → [Code Quality Guide](code-quality.md).

## Vue 3

- [ ] Destructuring `reactive()` object loses reactivity (use `toRefs`)
- [ ] Passing `props.x` to composable instead of `() => props.x` or `toRef(props, 'x')`
- [ ] `watch` with async callback missing `onCleanup` (race condition)
- [ ] `computed` with side effects (mutations, API calls)
- [ ] `v-for` using index as `:key` when list can reorder
- [ ] `v-if` and `v-for` on the same element
- [ ] `defineProps` without TypeScript type declaration
- [ ] `withDefaults` object default values not using factory functions
- [ ] Directly mutating props instead of emitting events
- [ ] `watchEffect` with unclear dependencies causing over-triggering

## Rust

**Ownership & Borrowing:**
- [ ] Unnecessary `clone()` to work around borrow checker
- [ ] `Arc<Mutex<T>>` when single-owner would suffice
- [ ] Storing borrows in structs when owned data is simpler
- [ ] Unnecessary `RefCell` (runtime checks vs compile-time)

**Unsafe Code:**
- [ ] `unsafe` block without `SAFETY:` comment explaining invariants
- [ ] `unsafe fn` without `# Safety` doc section
- [ ] Unsafe invariants split across modules

**Async & Concurrency:**
- [ ] Blocking in async context (`std::fs`, `std::thread::sleep`)
- [ ] Holding `std::sync::Mutex` across `.await`
- [ ] Spawned task missing `'static` lifetime bound
- [ ] Dropping a Future without awaiting (forgotten work)

**Error Handling:**
- [ ] `unwrap()`/`expect()` in production code
- [ ] Library using `anyhow` instead of `thiserror` (callers can't match)
- [ ] Swallowing error context (`map_err(|_| ...)`)
- [ ] Ignoring `must_use` return values

**Performance:**
- [ ] Unnecessary `.collect()` — prefer lazy iterators
- [ ] String concatenation in loops without `with_capacity`
- [ ] `Box<dyn Trait>` when `impl Trait` would work

## Go

- [ ] Ignoring errors (`result, _ := SomeFunction()`)
- [ ] Goroutine with no exit mechanism (leak)
- [ ] Missing or incorrect `context.Context` propagation
- [ ] Loop variable capture issue (Go < 1.22)
- [ ] `defer` in loops (deferred until function, not loop iteration)
- [ ] Variable shadowing
- [ ] Map used before initialization
- [ ] Error wrapping with `%v` instead of `%w` (breaks `errors.Is`/`errors.As`)

## Java / Spring Boot

- [ ] POJO/DTO with manual boilerplate instead of `record`
- [ ] Traditional switch missing `break` (use switch expressions)
- [ ] Field injection instead of constructor injection
- [ ] JPA N+1 query (missing `fetch join` or `@EntityGraph`)
- [ ] Incorrect `equals`/`hashCode` on JPA entities (use business key, not ID)
- [ ] `Optional.get()` without `isPresent()` check
- [ ] Stream operations with side effects

## C

- [ ] Pointer/buffer overflow or underflow
- [ ] Undefined behavior (use-after-free, double-free, null deref)
- [ ] Missing error handling after allocation (`malloc` can return `NULL`)
- [ ] Integer overflow in size calculations
- [ ] Resource leaks (missing `free`, `fclose`, etc.)
- [ ] Missing `static` on file-local functions/variables

## SQL

- [ ] String concatenation for queries (SQL injection risk) — use parameterized queries
- [ ] Missing indexes on filtered/joined columns
- [ ] `SELECT *` instead of specific columns
- [ ] N+1 query patterns
- [ ] Missing `LIMIT` on large tables
- [ ] Not handling `NULL` comparisons correctly (`IS NULL` vs `= NULL`)
- [ ] Missing transactions for related operations
- [ ] Incorrect JOIN types
- [ ] Collation / case sensitivity surprises across databases (MySQL vs Postgres defaults)
- [ ] Date and timezone handling errors (naive timestamps, server-local `NOW()`, DST)

**See also:** [Security Review Guide](security-review-guide.md) for SQL injection prevention

## API Design

- [ ] Inconsistent resource naming
- [ ] Wrong HTTP methods (POST for idempotent operations)
- [ ] Missing pagination for list endpoints
- [ ] Incorrect status codes
- [ ] Missing rate limiting
- [ ] Missing input validation and sanitization
- [ ] Trusting client-side validation only

## Testing

- [ ] Testing implementation details instead of behavior
- [ ] Missing edge case tests
- [ ] Flaky tests (non-deterministic)
- [ ] Tests with external dependencies (no mocks)
- [ ] Missing negative tests (error cases)
- [ ] Overly complex test setup

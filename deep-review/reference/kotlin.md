# Kotlin Code Review Guide

> Reviewer reminder for Kotlin diffs (Android + backend): coroutine scope/cancellation, Flow, Compose recomposition, null safety, memory leaks, architecture layering, sealed-class state, and server-side coroutine concerns. Terse by design — signal in the diff → what to flag.

## Contents

- [Coroutines: Scope and Cancellation](#coroutines-scope-and-cancellation)
- [Flow](#flow)
- [Jetpack Compose](#jetpack-compose)
- [Null Safety](#null-safety)
- [Memory Leaks](#memory-leaks)
- [Architecture: ViewModel and Repository](#architecture-viewmodel-and-repository)
- [Sealed Classes and State](#sealed-classes-and-state)
- [Server-Side Kotlin (Backend)](#server-side-kotlin-backend)
- [Review Checklist](#review-checklist)

## Coroutines: Scope and Cancellation

- `GlobalScope.launch` → flag, always. Use `viewModelScope` / `lifecycleScope` (Android) or an injected application scope (backend). Uncontrolled lifetime = leaks and post-destroy crashes.
- Broad catch in suspend code → must not swallow the cancellation signal:

```kotlin
try { repo.fetchData() }
catch (e: Exception) { showError(e) }          // ❌ swallows CancellationException — coroutine becomes uncancellable
// ✅ catch (e: CancellationException) { throw e } first, or call ensureActive() inside the catch
// ✅ runCatching { ... }.onFailure { if (it is CancellationException) throw it }
```
- CPU-bound loop in a coroutine → needs periodic `ensureActive()` / `yield()` or it ignores cancellation.
- Dispatcher choice: blocking I/O → `Dispatchers.IO`; CPU-bound → `Dispatchers.Default`. Flag decode/parse work on IO and blocking calls (JDBC, `execute()`) on Default or unqualified.
- `runInterruptible` only when the blocking API responds to `Thread.interrupt()` and prompt abort matters — not a default wrapper.
- Un-awaited `async { }` → flag: its exception surfaces only at `await()`, i.e. possibly never. Fire-and-forget = `launch`; parallel results = `async` + `await`, or `map { async { } }.awaitAll()` for homogeneous lists.
- `launch(Job())` → severs cancellation from the parent scope. And `Job(parent)` does NOT create an independent lifecycle — it's a child job, cancelled with the parent. Independent lifetime = own `CoroutineScope(SupervisorJob() + ...)` with an explicit `shutdown()` that cancels it, documented.
- `withContext(NonCancellable)` around whole operations → flag; legitimate only for cleanup in `finally`.
- `launch(SupervisorJob())` / `async(SupervisorJob())` → no-op mistake:

```kotlin
scope.launch(SupervisorJob()) {   // ❌ SupervisorJob becomes the PARENT; children still cancel each other,
    launch { taskA() }            //    and the coroutine is severed from scope's lifecycle
    launch { taskB() }
}
// ✅ scope.launch { supervisorScope { launch { taskA() }; launch { taskB() } } }
```

- `CoroutineExceptionHandler` on a child coroutine or on `async` → never invoked. Install it on the scope's context or the root `launch`; handle `async` failures at `await()`.

## Flow

- `flow { emit(api.fetch()) }` shared by multiple collectors → each collect re-runs the block (cold). Shared data belongs in `StateFlow`/`SharedFlow` (or `shareIn`/`stateIn`).
- `withContext` inside a `flow {}` builder → `IllegalStateException`. Use `.flowOn(dispatcher)` for upstream, or `channelFlow`/`callbackFlow` if context switching is genuinely needed.
- `try/catch` wrapped around `collect { }` → catches downstream exceptions too. Use the `.catch { }` operator (upstream only) and let collector exceptions propagate.
- UI state: `StateFlow` (has a value, replays latest). `SharedFlow(replay = 0)` for one-shot events → flag: events emitted with no active collector (e.g. during config change) are dropped; `extraBufferCapacity` doesn't fix that. Prefer a `Channel(BUFFERED).receiveAsFlow()` or model the event as state the UI acknowledges.
- Collecting in Activity/Fragment without lifecycle gating → flag:

```kotlin
// ✅ Fragment: viewLifecycleOwner.lifecycleScope + repeatOnLifecycle(STARTED) around collect
// ✅ Compose:  val state by vm.uiState.collectAsStateWithLifecycle()
// ❌ lifecycleScope.launch { vm.uiState.collect { binding.x = it } }  // keeps collecting when stopped; view refs after destroy
```

## Jetpack Compose

- Strong Skipping (default since Kotlin 2.0.20): unstable params compare by `===`, stable by `equals()`. A data class holding `List`/other unstable types passed as new-but-equal instances → child recomposes. Fix: `@Immutable`, `kotlinx.collections.immutable`, or pass primitive fields.
- Manual `remember { { ... } }` around lambdas → redundant noise under Strong Skipping (compiler memoizes); only correct on pre-2.0.20 compilers.
- Reading high-frequency state (scroll position etc.) to compute a boolean/derived value directly in composition → wrap in `remember { derivedStateOf { ... } }`.
- Side effects (VM calls, logging, navigation) in the composable body → run on every recomposition:

```kotlin
@Composable
fun MyScreen(userId: String, vm: MyViewModel) {
    vm.loadUser(userId)                          // ❌ fires on every recomposition
    LaunchedEffect(userId) { vm.loadUser(userId) } // ✅ only when userId changes
    val once = remember { vm.initialData() }       // ✅ one-time init
}
```
- Internal `mutableStateOf` for state the caller needs to control/test → hoist it: `value` + `onValueChange` params, `Modifier` parameter with default, caller owns state (`rememberSaveable`).
- Flow collection in composition without lifecycle awareness → `collectAsStateWithLifecycle()`, not `collectAsState()`, for Android UI state.

## Null Safety

- `!!` → flag; prefer `?.`, `?: default`, early return `?: return`, or `requireNotNull(x) { "why" }` with a message.
- Choosing the wrong init strategy → flag against intent: `lateinit` only when the lifecycle guarantees init before use (e.g. `binding` in `onCreate`); nullable when init is genuinely uncertain (null check required); `by lazy` for expensive thread-safe first-access init. `lateinit` on a value that may legitimately never be set is semantically wrong — use nullable.
- Java interop: platform types assigned to non-null Kotlin types → flag. Receive as nullable (`val u: User? = javaSvc.getUser()`) or wrap the Java API with explicit nullability.

## Memory Leaks

- Long-lived coroutine capturing `Context`/`View`/`binding` (esp. from GlobalScope or app-scoped work) → Activity leaked. Move to ViewModel + StateFlow.
- Listener registered (`registerListener`, callbacks, broadcast receivers) with no matching unregister in `onPause`/`onDestroyView` → flag.
- Custom `CoroutineScope` created but never cancelled → flag; require `shutdown()`/`cancel()`, or in a ViewModel use `addCloseable { ... }` (lifecycle 2.8+; there's also a `ViewModel(viewModelScope:)` overload for injecting a test scope).
- Singletons/companions holding `Activity`/`Fragment`/`View` references → flag.

## Architecture: ViewModel and Repository

- Public `MutableStateFlow`/`MutableLiveData` on a ViewModel → expose read-only (`asStateFlow()`), keep the mutable `_field` private.
- Filtering/sorting/mapping/business rules in the ViewModel → push into Repository/UseCase; ViewModel manages state only.
- ViewModel hitting the network directly with no cache → prefer single source of truth: Repository exposes `Flow` from the local store, `refresh()` writes network → DB; ViewModel derives state via `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), initial)`.
- Repository methods with verb-phrase names orchestrating multiple concerns (`validateAndSubmitOrder`) → extract a UseCase (`operator fun invoke`); Repository stays pure data access.

## Sealed Classes and State

- UI state as a bag of nullables (`isLoading` + `data?` + `error?`) → invalid combinations representable. Model as `sealed interface UiState { Loading; Success(data); Error(msg) }`.
- Navigation/one-shot events as enums or strings → can't carry params; use sealed classes with typed payloads.
- Network results collapsed to `null` on failure → error info lost; wrap in a sealed result type (Success / Error(code, message) / Exception(cause)) and map each branch to UI state in the ViewModel.
- `when` over sealed types: must be exhaustive, and capture the subject (`when (val s = state)`) so smart casts work — flag `as` casts inside branches.

## Server-Side Kotlin (Backend)

- `withContext(Dispatchers.IO)` hardcoded in services → inject `CoroutineDispatcher` (constructor param, production default `Dispatchers.IO`) so tests can pass a `TestDispatcher`.
- Blocking JDBC/HTTP calls directly in suspend request handlers → blocks server event-loop/worker threads; wrap in `withContext(ioDispatcher)`.
- ThreadLocal-bound transactions (`@Transactional`, JPA) crossing a dispatcher switch → queries silently run outside the transaction:

```kotlin
@Transactional
suspend fun transfer(...) {
    withContext(Dispatchers.IO) { dao.debit(...); dao.credit(...) }  // ❌ may execute OUTSIDE the tx
}
// ✅ whole blocking tx inside ONE withContext + transactionTemplate.execute { ... },
//    or coroutine-aware tx (R2DBC TransactionalOperator, Exposed newSuspendedTransaction)
```

- Fire-and-forget work (audit, analytics, notifications) launched in the request coroutine → dies with the request/client disconnect; `GlobalScope` → no shutdown, no error handling, invisible dependency:

```kotlin
suspend fun placeOrder(order: Order) = coroutineScope {
    launch { auditLog.write(order) }   // ❌ cancelled when the request ends
    ...
}
// ✅ inject applicationScope: CoroutineScope(SupervisorJob() + CoroutineExceptionHandler),
//    cancelled on shutdown; applicationScope.launch { auditLog.write(order) }
```
- MDC/correlation IDs are ThreadLocal → lost after suspension/dispatcher switch. Require `withContext(MDCContext())` (kotlinx-coroutines-slf4j) after `MDC.put`.
- Main-safety convention: a suspend function that blocks internally must wrap itself in `withContext(ioDispatcher)` — flag suspend functions that rely on every caller remembering the dispatcher.

## Review Checklist

### Coroutines

- [ ] No `GlobalScope`; correct lifecycle-bound or injected scope
- [ ] `CancellationException` rethrown everywhere (incl. `runCatching` in suspend code)
- [ ] CPU work on `Default`, blocking I/O on `IO`; long CPU loops call `ensureActive()`/`yield()`
- [ ] `runInterruptible` only for interrupt-responsive blocking APIs that must abort promptly
- [ ] No un-awaited `async`; `launch` for fire-and-forget, `await`/`awaitAll` for parallel results
- [ ] No `launch(Job())` / `launch(SupervisorJob())`; `supervisorScope { }` for independent children; `Job(parent)` is a child, not independent
- [ ] `NonCancellable` only for `finally` cleanup
- [ ] `CoroutineExceptionHandler` on scope/root only; `async` failures handled at `await()`

### Flow

- [ ] Cold vs hot understood: shared data via `StateFlow`/`SharedFlow`, not re-executing `flow {}`
- [ ] No `withContext` inside `flow {}`; use `flowOn` (or `channelFlow`)
- [ ] Collection is lifecycle-aware (`repeatOnLifecycle` / `collectAsStateWithLifecycle`)
- [ ] Upstream errors via `.catch`, not `try` around `collect`
- [ ] `StateFlow` for UI state; one-shot events via `Channel` or acknowledged state, not `SharedFlow(replay=0)`

### Compose

- [ ] Parameters stable (`@Immutable`, immutable collections) where new-but-equal instances are passed; under Strong Skipping, unstable params compare by `===`
- [ ] No redundant `remember` around lambdas under Strong Skipping (default since Kotlin 2.0.20)
- [ ] High-frequency derived reads wrapped in `derivedStateOf`
- [ ] Side effects in `LaunchedEffect`/`SideEffect`, never the composable body; one-time init via `remember`
- [ ] State hoisted; composables stateless and reusable
- [ ] `collectAsStateWithLifecycle()` for flows in composition

### Null Safety

- [ ] No `!!`; `?.` / `?:` / early return / `requireNotNull` with message
- [ ] `lateinit` only with lifecycle-guaranteed init; nullable when init is uncertain; `lazy` for expensive first-access
- [ ] Java interop values received as nullable (no platform-type leakage into non-null types)

### Memory Leaks

- [ ] No `Context`/`View` captured by long-lived coroutines
- [ ] Listeners unregistered in `onPause`/`onDestroyView`
- [ ] Custom scopes have a cancel path (`shutdown()` / `addCloseable`)
- [ ] No `Activity`/`Fragment` refs in singletons

### Architecture

- [ ] ViewModel exposes immutable state only
- [ ] Business logic in Repository/UseCase, not ViewModel
- [ ] Repository as single source of truth (offline-first where applicable)
- [ ] Complex orchestration extracted into UseCases

### Sealed Classes and State

- [ ] UI state modeled with sealed types; impossible states unrepresentable
- [ ] Events and nav carry typed payloads via sealed classes
- [ ] Network results wrapped without losing error info
- [ ] `when` exhaustive with captured subject; no `as` casts

### Server-Side (Backend)

- [ ] Dispatchers injected (`CoroutineDispatcher` constructor param, default `Dispatchers.IO`), never hardcoded
- [ ] No blocking calls (JDBC, blocking HTTP) directly in request coroutines; `withContext(ioDispatcher)`
- [ ] No dispatcher switch inside ThreadLocal-bound transactions; coroutine-aware tx APIs (`TransactionalOperator`, `newSuspendedTransaction`) where needed
- [ ] Fire-and-forget in an injected application scope (`SupervisorJob` + handler), not `GlobalScope`/request scope
- [ ] MDC / correlation IDs propagated with `MDCContext()` (kotlinx-coroutines-slf4j)
- [ ] Suspend functions are main-safe: internal `withContext`, callable from any dispatcher

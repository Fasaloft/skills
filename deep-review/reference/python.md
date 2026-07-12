# Python Code Review Guide

Reviewer reminder sheet: signals to flag in a Python diff and the expected fix. Assumes fluent Python 3.11+ — no tutorials.
Universal logic/resource/error checks live in the [Code Quality Guide](code-quality.md); Lambda-specific concerns in the [AWS Lambda Guide](aws-lambda.md).

## Contents

- [Typing](#typing)
- [Mutable State and Defaults](#mutable-state-and-defaults)
- [Exceptions and Error Handling](#exceptions-and-error-handling)
- [Resources and Context Managers](#resources-and-context-managers)
- [Data Modeling](#data-modeling)
- [Idioms and Correctness Traps](#idioms-and-correctness-traps)
- [Async](#async)
- [Logging](#logging)
- [Testing (pytest)](#testing-pytest)
- [Tooling and Packaging](#tooling-and-packaging)
- [Review Checklist](#review-checklist)

## Typing

- New/changed public functions without annotations in a typed repo → add them. New code uses modern syntax: `list[str]`, `dict[str, int]`, `X | None` — not `typing.List`/`Optional`.
- `Any` as a param/return type, or `cast()` used to silence the checker → proper type, `object` + `isinstance` narrowing, or a `Protocol`. `cast` asserts without checking — it is Python's `as`.
- `# type: ignore` without an error code and reason → flag; prefer fixing, else `# type: ignore[code]  # why`.
- Implicit optional (`def f(x: str = None)`) → `x: str | None = None`.
- Ad-hoc dicts passed around as records (`user["emial"]` won't fail until runtime) → `TypedDict`, `@dataclass`, or a Pydantic model.
- Dependencies typed as concrete classes when only a couple of methods are used → `Protocol` (structural typing keeps call sites testable without the real class).
- Functions that only iterate a param typed as `list[X]` → accept `Iterable[X]`/`Sequence[X]`; return concrete types.
- Type gymnastics (nested `TypeVar`s, overload towers) serving one call site → flag; KISS.

## Mutable State and Defaults

- Mutable default argument (`def f(x=[])`, `={}`, `=set()`) → `None` sentinel, or in dataclasses `field(default_factory=list)`. The default is created once and shared across calls.
- Class attribute holding a mutable container (`class C: items = []`) → shared across all instances; move to `__init__` or `default_factory`.
- Mutating a list/dict while iterating it → iterate a copy, or build a new collection.
- Function stores a caller's mutable argument and later mutates it (aliasing) → copy at the boundary; know when `deepcopy` is needed for nested structures.

## Exceptions and Error Handling

- Bare `except:` → also catches `KeyboardInterrupt`/`SystemExit`; at minimum `except Exception`.
- `except Exception: pass` → swallowed failure. Narrow the type, handle it, or make the intent explicit with `contextlib.suppress(SpecificError)`.
- Wrapping and re-raising without the cause → `raise NewError(...) from e` (or explicit `from None`); a bare `raise NewError(...)` inside `except` buries the original traceback in noise.
- `try` block spanning many statements → shrink it to the operation that can fail; a broad block makes the `except` catch unrelated bugs.
- Failure collapsed to `return None` → callers can't distinguish "absent" from "broken"; raise a typed exception or return a result object.
- Raw `raise Exception("msg")` / `ValueError` for domain failures → a small module exception hierarchy so callers can catch selectively (and error-to-status mapping stays in one place).
- `assert` used for runtime validation of inputs/invariants that must hold in production → stripped under `python -O`; use an explicit `raise`.
- `if os.path.exists(...)` / `if key in d` followed by the operation → EAFP: do the operation, catch the specific error (see [TOCTOU](code-quality.md#toctou-race-conditions)).

## Resources and Context Managers

- `open()`, locks, DB connections/cursors without `with` → context manager; several at once → one `with a, b:` or `contextlib.ExitStack`.
- `requests` / `httpx` call without `timeout=` → `requests` waits forever by default; always set a timeout. Client/`Session` created per call inside a loop → create once, reuse (connection pooling).
- Manual temp-file bookkeeping → `tempfile.NamedTemporaryFile` / `TemporaryDirectory`.
- A generator that holds a resource open (yield inside `with`) → cleanup runs only when the generator is exhausted or GC'd; callers that stop early leak it. Prefer `contextlib.contextmanager` handed to the caller, or close explicitly.

## Data Modeling

- Tuples with positional meaning or bag-of-fields dicts crossing function boundaries → `@dataclass` (`frozen=True` for value types, `slots=True` when instances are numerous) or `NamedTuple`.
- String literals for states/kinds/actions compared around the codebase → `enum.Enum`/`StrEnum`; exhaustive `match` over the enum where branching.
- Pydantic v2: `model_validate` / `model_dump` — flag v1 API (`parse_obj`, `.dict()`) in new code. Models parsing external input need `model_config = {"extra": "forbid"}` (unknown fields silently dropped otherwise).
- Validate at the boundary, trust inside: a Pydantic model re-validated in inner helpers → the trust boundary is blurred; parse once at the entry point, pass typed objects inward.
- Mutable dataclass used as a dict key / element of a set; `__eq__` defined without `__hash__` → flag.

## Idioms and Correctness Traps

- `is` for value comparison (`x is "done"`, `x is 1000`) → `==`; `is` only for `None` and sentinels. Interning makes this pass in tests and fail in prod.
- Truthiness where falsy values are legitimate (`if not count:` when `0` is valid; `if not s:` when `""` is valid) → explicit `is None` / `== 0` check.
- Late-binding closure in a loop (`callbacks.append(lambda: process(i))`) → all closures see the final `i`; bind with a default (`lambda i=i:`) or `functools.partial`.
- `datetime.utcnow()` / naive datetimes in new code → `datetime.now(timezone.utc)`; `utcnow` is deprecated (3.12) and naive-vs-aware comparison raises. Money/precise arithmetic → `Decimal`, never `float`.
- `zip()` where length mismatch means a bug → `zip(strict=True)` (silent truncation otherwise).
- `functools.lru_cache` on an instance method → the cache keeps `self` alive forever and is shared across instances; use `functools.cached_property` or a module-level function.
- `os.path` string mangling in new code → `pathlib.Path`.
- `subprocess` with `shell=True` and interpolated input → list args (see [Security Guide](security-review-guide.md)); `subprocess.run` without `check=True` → non-zero exit ignored.
- Import-time side effects (network, file I/O, env validation crashing on import) → move into functions; modules must be importable in tests.
- Shadowing builtins (`list`, `id`, `type`, `input`) as variable names → rename.

## Async

- Blocking calls inside `async def` (`time.sleep`, `requests`, sync DB drivers, heavy CPU) → block the entire event loop; `asyncio.sleep`, an async client, or `asyncio.to_thread`.
- Coroutine called but not awaited → nothing runs (only a `RuntimeWarning`); flag any bare `some_coro()` statement.
- `asyncio.create_task(...)` result discarded → the task can be garbage-collected mid-flight and its exception is lost; keep a reference and handle completion, or use structured concurrency:

  ```python
  async with asyncio.TaskGroup() as tg:   # 3.11+; failure cancels siblings, exceptions propagate
      tg.create_task(fetch_a())
      tg.create_task(fetch_b())
  ```

- `asyncio.gather(...)` on failure leaves the other coroutines running detached → prefer `TaskGroup`; `gather(return_exceptions=True)` requires the caller to actually inspect the results for exceptions.
- `except Exception` in async code swallows `CancelledError` on 3.7 semantics — on modern Python it's a `BaseException`, but a broad `except BaseException` or bare `except` still eats cancellation → re-raise it.
- Check-then-act across an `await` → race window; shared mutable state touched from multiple tasks needs an `asyncio.Lock` or a redesign.

## Logging

- `print()` in service/library code → module logger (`logging.getLogger(__name__)` or the repo's structured logger).
- `logger.error(str(e))` or `logger.error(f"failed: {e}")` in an `except` block → `logger.exception("failed")` (keeps the traceback); f-string interpolation also evaluates even when the level is filtered — use lazy `%s` args or structured `extra=`.
- Library/shared modules calling `basicConfig` or adding handlers → only the application configures logging.
- User input or secrets interpolated into log lines → structured fields, never raw (log injection + leakage; see [Security Guide](security-review-guide.md#logging--monitoring)).

## Testing (pytest)

- `unittest.TestCase` boilerplate in a pytest repo → plain functions, fixtures, plain `assert`.
- Copy-pasted near-identical test bodies → `@pytest.mark.parametrize`.
- `pytest.raises(Exception)` → precise exception type plus `match=` on the message.
- `mock.patch` targeting where the object is *defined* instead of where it is *looked up* → patch `module_under_test.dep`; the test passes/fails for the wrong reason otherwise.
- `time.sleep` to "wait for" async/threaded effects → flaky; use events, polling with deadline, or a fake clock (`time-machine`/`freezegun`).
- Test asserts a call sequence on mocks while the observable behavior goes unchecked → test behavior, not implementation; if everything is mocked, the test tests the mocks.
- Session/module-scoped fixture exposing mutable state that tests mutate → cross-test contamination; function scope or return copies.
- Tests reaching real network/AWS/DB → fakes at the boundary (`moto`, botocore `Stubber`, injected clients); env via `monkeypatch.setenv`, not `os.environ` writes.

## Tooling and Packaging

- `# noqa` / `# type: ignore` without a rule code and reason → flag. Skip anything ruff/mypy already enforces in CI — don't re-litigate the linter.
- Dependency added without lockfile update (`uv.lock`/`poetry.lock` and any exported `requirements.txt` must move together) → flag; new heavy dependency duplicating stdlib or an existing dep → question it.
- Version pins: applications pin (lockfile), libraries use ranges; a library pinning exact versions → flag.
- Import inside a function to break a cycle → works, but it's a design smell; note it and suggest `if TYPE_CHECKING:` for type-only imports or restructuring.
- Logic in `__init__.py` beyond re-exports → import-time surprises; keep it declarative.

## Review Checklist

### Typing
- [ ] Public functions annotated; modern syntax (`X | None`, `list[str]`)
- [ ] No `Any`/`cast` escapes; `# type: ignore` has code + reason
- [ ] Records are TypedDict/dataclass/Pydantic, not ad-hoc dicts; Protocols at test-relevant boundaries

### State & Errors
- [ ] No mutable defaults (args, class attrs, dataclass fields)
- [ ] No bare `except` / silent swallow; narrow `try` blocks; `raise ... from e`
- [ ] Typed exception hierarchy; no `assert` for production validation
- [ ] EAFP over check-then-act

### Resources
- [ ] `with` for files/locks/connections; `ExitStack` for many
- [ ] HTTP calls have timeouts; clients/sessions reused, not per-call

### Data & Idioms
- [ ] Enums over string constants; frozen dataclasses for value types
- [ ] Pydantic v2 API, `extra="forbid"` on input models, validation at the boundary only
- [ ] `is` only for `None`/sentinels; explicit checks where falsy is valid
- [ ] Timezone-aware datetimes; `Decimal` for money; `zip(strict=True)`
- [ ] No `lru_cache` on methods; no import-time side effects; no shadowed builtins

### Async
- [ ] No blocking calls in coroutines; no un-awaited coroutines
- [ ] Tasks referenced or in a `TaskGroup`; cancellation not swallowed
- [ ] No check-then-act across `await` on shared state

### Logging & Tests
- [ ] Logger not `print`; `logger.exception` in handlers; no secrets/PII in logs
- [ ] Parametrized tests, precise `raises(match=)`, patch-where-looked-up
- [ ] No sleeps, no real network; behavior asserted, not mock choreography

### Packaging
- [ ] Lockfile moves with manifest; pins appropriate to app vs library
- [ ] `noqa`/ignores justified; no logic in `__init__.py`

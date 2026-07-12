# AWS Lambda Review Guide

Reviewer reminder sheet for serverless diffs: Lambda handlers, event source wiring, and the SAM/CloudFormation templates around them.
Runtime-agnostic principles with Python (powertools/Pydantic) examples — the house default. Language-level issues → [Python Guide](python.md);
injection/secrets fundamentals → [Security Guide](security-review-guide.md). Review the template with the same rigor as the handler —
half the Lambda bugs live in the event source mapping, not the code.

## Contents

- [Handler and Execution Lifecycle](#handler-and-execution-lifecycle)
- [Event Parsing and Validation](#event-parsing-and-validation)
- [Idempotency and Retry Semantics](#idempotency-and-retry-semantics)
- [Batch Processing (SQS / Streams)](#batch-processing-sqs--streams)
- [Timeouts, Limits, Concurrency](#timeouts-limits-concurrency)
- [Error Handling and DLQs](#error-handling-and-dlqs)
- [IAM and Secrets](#iam-and-secrets)
- [Observability](#observability)
- [Infrastructure (SAM / CloudFormation)](#infrastructure-sam--cloudformation)
- [Performance and Cost](#performance-and-cost)
- [Testing](#testing)
- [Review Checklist](#review-checklist)

## Handler and Execution Lifecycle

- boto3 clients, DB connections, config/secret fetches, Pydantic model setup created **inside** the handler → move to module level; they are reused across warm invocations. Per-invocation clients = latency + connection churn on every call.
- The inverse trap: module-level **mutable** state written per invocation (request-keyed caches, appended lists, logger context via `append_keys`) → leaks into the next warm invocation, potentially across tenants. State must be invocation-scoped or explicitly reset (powertools `@logger.inject_lambda_context(clear_state=True)`).
- Handler containing the business logic inline → thin handler that parses/validates, then delegates to plain functions — those are testable without a Lambda runtime.
- Work still in flight at `return` (background threads, fire-and-forget async, buffered telemetry) → the sandbox freezes immediately after return; the work resumes on the *next* invocation or never. Everything must complete before return; buffered exporters flush in `finally` (see Observability).
- `/tmp` or in-memory caches treated as durable → they are best-effort per-sandbox only; `/tmp` is 512 MB by default and shared across warm invocations — clean up or bound it.
- Cold start: heavyweight imports (pandas, numpy) used only on a rare path → import lazily inside that branch; everything used per-invocation initializes once at module level.

## Event Parsing and Validation

- Raw `event["field"]` access in a handler → typed parsing at the boundary, e.g. powertools `@event_parser` with a Pydantic model (`extra="forbid"`), or an envelope + schema validator. Every entry point — including "internal" direct invokes — is a trust boundary.
- Hand-rolled envelope unwrapping → bug magnet; flag mistakes with the real shapes: SQS `record.body` is a JSON *string*; SNS payload sits under `Sns.Message`; SNS→SQS wraps twice; API Gateway `body` is a string and may be base64 (`isBase64Encoded`). Powertools envelopes/data classes handle these.
- Tenant/user identity read from the request body → must come from the verified principal context (`requestContext`, authorizer claims), never the payload. The body says who the caller *claims* to be.
- URL, S3 key, or filesystem path built from event input → SSRF / path traversal review ([Security Guide](security-review-guide.md#input-validation--injection)).

## Idempotency and Retry Semantics

- **Every async source (SQS, SNS, EventBridge, S3 events, streams) is at-least-once** — the whole handler run gets repeated on retry or duplicate delivery. Non-idempotent side effects (send email, charge, unconditional `put_item`, increment) without protection → flag; require conditional writes, an idempotency key, or powertools `@idempotent`.
- Side effect A succeeds, then B throws → the retry repeats A. Order steps so replays are safe, or make each step individually idempotent.
- DynamoDB `ConditionalCheckFailedException` handled as a plain error → distinguish "already processed" (treat as success, skip) from a genuine conflict; log it either way — repeated conditional failures can indicate replay or tampering.
- Handler writing to a source that triggers itself (S3 event handler writing to the same bucket/prefix, SQS handler sending to its own queue) → recursive invocation loop; require prefix/suffix filters or a different target.

## Batch Processing (SQS / Streams)

- Plain `for record in event["Records"]` where one bad record raises → the **entire batch** returns to the queue; good messages get reprocessed (duplicates) and one poison message blocks everything. → partial batch response:

  ```python
  processor = BatchProcessor(EventType.SQS)          # module level

  def record_handler(record: SQSRecord) -> None:
      payload = Input.model_validate_json(record.body)   # per-record validation
      handle(payload)

  def lambda_handler(event, context):
      return process_partial_response(event, record_handler=record_handler,
                                      processor=processor, context=context)
  ```

- Partial batch response is a **two-sided contract**: the code returning `batchItemFailures` does nothing unless the event source mapping declares `FunctionResponseTypes: [ReportBatchItemFailures]` — and the mapping setting without the response shape fails the whole batch as before. A diff changing one side only → flag; check the template.
- `try/except` around the record loop that logs and continues → the message is deleted as "succeeded"; the data is silently gone. Errors must surface through the partial-failure mechanism so retry/DLQ machinery works.
- FIFO queues: skipping a failed record and processing the next one in the same group breaks ordering → stop the group on first failure (powertools `SqsFifoPartialProcessor`).
- Kinesis / DynamoDB streams: a poison record blocks the **shard** until it expires. Mapping needs `MaximumRetryAttempts`, `BisectBatchOnFunctionError`, and an `OnFailure` destination — a stream mapping without them → flag.

## Timeouts, Limits, Concurrency

- `Timeout` left at the 3 s default, or set to 900 "to be safe" → set deliberately from expected work; an oversized timeout hides hangs and multiplies retry cost.
- Downstream calls can outlive the function: boto3's default retries/read timeouts can exceed a short function timeout (the SDK retries while Lambda kills the sandbox) → botocore `Config(connect_timeout=…, read_timeout=…, retries={…})` budgeted **below** the function timeout; every HTTP call has an explicit timeout.
- SQS visibility timeout < ~6× function timeout (+ batching window) → in-flight messages reappear and process concurrently with the still-running attempt; check the queue definition when the function timeout changes.
- Payload limits crossed silently: 6 MB sync / 256 KB async invoke; 256 KB SQS/SNS/Step Functions state → large blobs go to S3 with a pointer in the message (claim check), not inline.
- Long-running batch work ignoring `context.get_remaining_time_in_millis()` → killed mid-record with no checkpoint; stop early and return unprocessed items as failures.
- Downstream with a hard capacity limit (DB connections, rate-limited API) fed by an unbounded event source → cap it: reserved concurrency on the function or `ScalingConfig.MaximumConcurrency` on the SQS mapping. Conversely, reserved concurrency on one function starves the account pool for the rest — must be justified.

## Error Handling and DLQs

- Async-invoked function or queue without a DLQ / `OnFailure` destination → failures vanish after retries. Every DLQ needs a CloudWatch **alarm** (>0 messages) and retention long enough to investigate (14 days) — an unmonitored DLQ is a silent black hole.
- Retryable and non-retryable failures treated the same → a validation error will fail identically on every retry; it should go to the DLQ fast (low `maxReceiveCount`) or be routed out, not burn retries. Transient errors (throttle, timeout) are what retries are for.
- Error responses to synchronous callers leaking internals (stack trace, exception repr, table/key names, ARNs) → generic message + status to the caller, full detail to structured logs only.
- Invoking another Lambda and propagating its error payload upstream → strip `stackTrace`/`requestId` first (house util `sanitize_lambda_error_response` where available).
- Catch-all returning HTTP 200 with an error body on a queue/stream path → the message is acknowledged and lost; only API-style handlers translate errors to responses.

## IAM and Secrets

- One execution role **per function**, not a shared service-wide role → shared roles accumulate the union of all permissions (blast radius). Least privilege: specific actions on specific ARNs.
- `Action: "s3:*"` / `dynamodb:*`, or `Resource: "*"` on data actions (read/write, `lambda:InvokeFunction`, `execute-api`) → flag; wildcard kept "temporarily" needs a TODO naming what will scope it.
- Org conventions travel with house rules — check them (e.g. mandated permission boundary, role path/naming); a role diff missing the org's boundary/path fails deploy or, worse, escapes governance.
- Trust and resource policies: `Principal: "*"` on an Allow without an org/source condition (`aws:PrincipalOrgID`, `aws:SourceArn`) → blocker.
- Secrets in CloudFormation env vars (plaintext value, or `{{resolve:secretsmanager:…}}` landing in an env var) → visible in the console, `GetFunctionConfiguration`, and template state. Pass the secret **name**, fetch from Secrets Manager / SSM SecureString at cold start, cache module-level with a TTL so rotation propagates. `{{resolve:ssm:…}}` is fine for non-sensitive config only.
- Required env vars read mid-invocation with bare `os.environ[...]` → validate at cold start and fail fast with a clear error, not a `KeyError` halfway through a batch.

## Observability

- `print` / unstructured logs → structured JSON logger (powertools `Logger`); `@logger.inject_lambda_context` for the request-id correlation, `clear_state=True` when `append_keys` is used.
- Logging the entire `event` → PII, tokens, and headers end up in CloudWatch; log selected fields. Tenant/principal context on every line; `logger.exception` (not `logger.error(str(e))`) in handlers.
- Buffered telemetry (OTel `PeriodicExportingMetricReader`, powertools `Metrics`) without an end-of-invocation flush → the sandbox freezes between invocations and the periodic export may never fire. Outermost `try/finally: flush()` in the handler — powertools decorators stay outside it; extract the body to `_handle()` if needed.
- New user-facing function without tracing (`Tracing: Active` / `@tracer.capture_lambda_handler`) where the repo standard has it → flag; log level hardcoded to DEBUG → env-driven.

## Infrastructure (SAM / CloudFormation)

- New function without explicit `Timeout` and `MemorySize` → defaults (3 s / 128 MB) are almost never a decision; require deliberate values. Runtime pinned and current (`python3.12`+) — a new function on a deprecated runtime → blocker.
- Additions to `Globals` (layers, env vars) → tax every function's cold start and config; per-function unless genuinely universal.
- Event source mapping settings are review targets, not boilerplate: `BatchSize` + `MaximumBatchingWindowInSeconds` (latency vs cost), `FunctionResponseTypes` (see Batch Processing), stream retry/bisect/destination, `FilterCriteria` to avoid invoking on irrelevant events (cheaper and safer than filtering in code).
- `VpcConfig` added without a stated need → only for reaching private resources; a VPC Lambda needs VPC endpoints (or NAT) for every AWS API it calls — a missing endpoint surfaces as a timeout, not an error. Conversely, removing VPC config from a function that reads RDS/ElastiCache breaks it.
- Layers referenced without a pinned version ARN → unpinned layer = unreviewed code changes at deploy time.
- Queue/topic/bucket definitions moving with the function: DLQ redrive, visibility timeout, encryption at rest — check them in the same diff, they are part of the function's contract.

## Performance and Cost

- CPU scales with `MemorySize` — a CPU-bound function at 128 MB is slower *and* often more expensive than at 512 MB. Memory changed or workload character changed → ask for a tuning rationale (power-tuning numbers beat guesses).
- Synchronous Lambda→Lambda invoke chains → paying for both functions while one waits, and timeouts stack; prefer async invoke, a queue, or Step Functions for orchestration.
- Whole S3 object loaded into memory to use a slice → ranged `get_object` or streaming; memory limit and 512 MB `/tmp` are easy to blow on "just download it".
- Per-invocation re-fetch of config/secrets/reference data → module-level cache with TTL.
- One-message-at-a-time processing of a high-volume queue (`BatchSize: 1`) without a reason → batching amortizes cold start and invocation overhead; conversely huge batches push against the 6 MB payload and timeout budget.

## Testing

- Business logic only reachable through the handler → restructure (thin handler) so core logic tests need no Lambda plumbing.
- Handler tests with minimal hand-crafted event dicts → they drift from real shapes; use recorded sample events or powertools data classes/factories (SQS envelope, API GW v1 vs v2 differ materially).
- Tests hitting real AWS → `moto`, botocore `Stubber`, or injected fake clients at the boundary; never live calls in unit tests.
- Module-level init (clients, env validation, telemetry providers) runs at **import time** — tests importing the handler need env vars and no-op exporters set in `conftest.py` *before* import; a fixture is too late. A diff adding module-level init that breaks bare import → flag.
- Retry semantics deserve tests: duplicate delivery (idempotency) and a poison record in a batch (partial failure) are the two cheapest high-value cases.

## Review Checklist

### Lifecycle
- [ ] Clients/config init at module level; no per-invocation client creation
- [ ] No mutable module state leaking across warm invocations (logger state cleared)
- [ ] Thin handler; logic in plain testable functions
- [ ] No in-flight work at return; telemetry flushed in `finally`

### Input
- [ ] Typed validation at entry (`@event_parser`/schema, `extra="forbid"`), incl. internal invokes
- [ ] Correct envelope handling (SQS body string, SNS nesting, API GW base64)
- [ ] Identity from verified context, never the payload

### Retries & Batches
- [ ] Side effects idempotent under at-least-once delivery
- [ ] Partial batch response wired on BOTH sides (code + `ReportBatchItemFailures`)
- [ ] No swallowed record errors; FIFO ordering preserved on failure
- [ ] Stream mappings have retry limits, bisect, `OnFailure` destination
- [ ] No self-triggering recursion (S3 same-bucket, own queue)

### Limits
- [ ] Timeout/memory deliberate; SDK/HTTP timeouts budgeted below function timeout
- [ ] SQS visibility ≥ ~6× function timeout; payload limits respected (claim check)
- [ ] Concurrency capped where downstream capacity is finite

### Errors
- [ ] DLQ/OnFailure + alarm on every async path; retention allows investigation
- [ ] Non-retryable failures don't burn retries; sanitized errors to callers

### IAM & Secrets
- [ ] Per-function role, least privilege, no wildcards on data actions
- [ ] Org boundary/path conventions honored; no `Principal: "*"` without conditions
- [ ] Secrets fetched at runtime by name and cached with TTL — never env-var values
- [ ] Env vars validated at cold start

### Infra & Tests
- [ ] Template settings explicit (runtime, timeout, memory); nothing snuck into `Globals`
- [ ] Event source mapping settings reviewed (batch size, window, filters)
- [ ] VPC config justified; layers version-pinned
- [ ] Tests use real event shapes and fake AWS; import-safe modules; idempotency + poison-record cases covered

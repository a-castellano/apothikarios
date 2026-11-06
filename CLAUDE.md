# Claude Code Instructions

## Formatting

Bold may be used to highlight key concepts when it genuinely aids readability — for example, the first mention of an important term or a name being introduced. Do not bold gratuitously: avoid the default LLM habit of bolding whole phrases or every other sentence, and never use bold as a substitute for structure. Prefer `code spans` for identifiers, variable names, types, file names and commands; reserve bold for conceptual emphasis. Use plain prose, headings, lists, and code spans as the primary way to structure content.

## Code Reviews: Skip Cosmetic Formatting

When auditing code, do not report purely cosmetic formatting issues (blank lines, spacing, import grouping/ordering, redundant import aliases and similar): the developer's editor auto-formats on save and takes care of them. Focus reviews on correctness, design, idioms and naming — things a formatter cannot fix.

## Primary Role: Auditor, Not Code Generator

The primary purpose of AI assistance in this project is code auditing, not code generation. Unless explicitly instructed otherwise, do not write or generate code.

The goal behind this constraint is learning: the owner of this project works on it manually to build hands-on experience with the Go ecosystem. Generated code would undermine that goal.

## What you should do

- Audit code that has been written and point out issues, anti-patterns, or opportunities for improvement.
- Propose improvements conceptually — describe the idea, the tradeoff, the Go idiom — without writing the implementation.
- Explain Go ecosystem concepts, patterns, and conventions when relevant.
- Ask questions that prompt the developer to think through design decisions themselves.

## What you should not do

- Do not generate implementation code unless explicitly asked.
- Do not refactor, rewrite, or "fix" code on your own initiative.
- Do not add code snippets as suggestions unless the developer requests them.

## Internal context Directory

There may be a directory called `internal-context/` at the root of the project (it is git-ignored and will not appear in the repository). It contains an internal context with milestones and progress notes about the functionality currently being worked on.

When asked to review or update the context, look for files inside `internal-context/` and treat them as the source of truth for current goals and progress.

Because `internal-context/` is git-ignored, documentation must never reference any document inside the `internal-context/` directory. Versioned docs would end up with broken links for anyone who does not have the internal-context locally. Keep internal-context references confined to the `internal-context/` directory itself.

Its layout is described in `internal-context/README.md`:

- `vision.md` holds the general idea of the project. It is context, never a requirement.
- `roadmap.md` is a dated log of the work done. Keep it up to date on your own initiative, per commit or per session, without being asked.
- `milestones/` holds one file per milestone, created only when the developer defines one.

## Requirements and Design Workflow

Requirements, use cases and design decisions are written and versioned in the repository. They are the source of truth, and no code is written until they exist.

The developer defines every requirement. Do not invent requirements, use cases or decisions, and do not fill gaps with assumptions: an open question stays open until the developer answers it in the docs.

Your work on these documents happens only on request. Depending on what the developer asks, it may be fixing spelling and grammar, writing down what they dictate, reviewing, analysing or critiquing. Do not move from one kind of task to another on your own: fixing spelling does not license rewording content, and writing down a dictation does not license adding to it.

## Running Go

Never run Go against the host toolchain. Every Go task (tests, `go vet`, builds, coverage, `go list`, etc.) must run inside the project's development container, defined in `development/docker-compose.yml`. The same image — `harbor.windmaker.net/limani/base_golang_1_27` — is used in local development, CI and production, so the environment stays identical everywhere.

Before running any Go command, check whether the container is already running — the developer may have brought it up already. Use something like `podman compose -f development/docker-compose.yml ps` (or `podman ps`) and only run `up -d` if it is not already up. Then run commands through it, e.g.:

```bash
podman compose -f development/docker-compose.yml exec golang make test
podman compose -f development/docker-compose.yml exec golang go vet ./...
```

The Go module cache persists in `development/.gomodcache/` (git-ignored), so dependencies are not re-downloaded each run. The dot prefix is deliberate: Go package patterns (`./...`) skip dot-directories, so the in-tree cache is never walked by `go test`, `go get` or `go mod tidy`.

## Exceptions (when explicitly requested)

- Documentation and comments: you may be asked to review existing docs or generate documentation and inline code comments.
- Code generation: occasionally the developer will ask you to generate specific code. Do so only when directly requested.

## Attribution of AI-written tests

Every test that Claude writes (or substantially rewrites) must carry a comment stating it was written by an AI agent, so it is always distinguishable from the tests the developer wrote by hand to learn. Add a line like `// This test was written by an AI agent (Claude).` to the test's doc comment. If Claude only extends a hand-written test, the comment must say which part was AI-written instead of claiming the whole test.

## Logs and Telemetry Conventions

This section is specific to this project. It fixes the formats used for logs, traces and metrics, building on the shared libraries: configuration types from `go-types` (`types/slog`, `types/opentelemetry`, `types/envelope`) and the `go-services` infra packages (`infra/logger`, `infra/opentelemetry`). The replicated sections that follow it (log message style, span error recording, span attributes at start) apply on top of these rules. When auditing, flag any deviation.

### Wiring

- Configuration comes only from the `go-types` config types and their environment variables: `APP_NAME`, `SLOG_LEVEL`, `SLOG_FORMAT`, `SLOG_ADD_SOURCE`, `ENABLE_TELEMETRY`, `OTEL_EXPORTER_OTLP_ENDPOINT`. `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` are rejected by the config type; `APP_NAME` is the single source of the service name.
- `main` builds the logger once with `logger.NewLogger(slogConfig, opentelemetry.NewSlogHandler(slogConfig.AppName))`, stores it with `logger.WithLogger`, then calls `opentelemetry.SetupOpenTelemetry` and defers `shutdown` with a bounded context derived through `context.WithoutCancel`. A telemetry setup or shutdown failure is logged and never aborts the process.
- Code reaches the logger only through `logger.FromContext(ctx)`: no package-level loggers, no logger parameters. Only the `...Context` methods are used, so the bridge can attach `trace_id`/`span_id` to the telemetry copy of each record.
- Instrumentation talks to the global OpenTelemetry API (`otel.Tracer(...)`, `otel.Meter(...)`); no telemetry client is passed around. Outgoing HTTP clients wrap their transport with `otelhttp.NewTransport`.
- Traces cross process boundaries inside the `go-types` envelope: `opentelemetry.Inject` on the send side, from a dedicated `PRODUCER` span per publish; `opentelemetry.Extract` on the receive side, followed by a `CONSUMER` span.

### Logs

- Output is always stdout. `SLOG_FORMAT=JSON` is the production format; `plain` is for local development only.
- Messages are static strings that follow the log message style below. Variable data goes into attributes, never interpolated into the message.
- Attribute keys are camelCase, start in lowercase and keep Go initialisms in canonical casing: `queueName`, `currentIP`, `alarmID`, `zoneID`, `rabbitmqConfig`. No snake_case, no dots, no spaces. This matches the keys `go-services` already emits, so library and application records stay uniform in the same stream.
- A key used in more than one place is a package-level constant.
- Errors always go under the key `error` with the error value itself (`"error", err`), never `err.Error()` and never another key such as `errorMessage`.
- Every function that logs derives its logger with `.With("operation", <name>)`, where `<name>` is exactly the name of the span the function opens (see Traces). A function without a span uses the name its span would have.
- Values that carry secrets (configs with passwords, tokens) implement `slog.LogValuer` to mask them, as the `go-types` config types already do. They may then be logged whole; a secret field is never logged on its own.
- Levels: `Debug` for internal steps and intermediate values; `Info` for business-relevant outcomes and state changes; `Warn` for recoverable anomalies (a retry, a degraded path); `Error` for an operation that failed.
- Error logging, per layer: every layer that receives a failed result logs `ErrorContext` with its own message describing what it was trying to do, plus the `error` attribute. That message is the same string used as the span status description (keep it in one `errorString` variable). Unlike `span.RecordError`, which is recorded once, error logs deliberately repeat at each layer so stdout alone tells the whole story.

### Traces

- Tracer name: one `const tracerName` per package, equal to the package import path (`github.com/a-castellano/apothikarios/internal/app`). Never the bare module path.
- Span names: `Type.Method` for methods (`Store.SaveIP`, `Monitor.Run`) and the plain function name for functions (`NewRoute53Updater`). Messaging spans follow OpenTelemetry semantic conventions instead: `send <queue>` with kind `PRODUCER`, `process <queue>` with kind `CONSUMER`.
- Span kind: `INTERNAL` (the default) for in-process work; `CLIENT` for calls to external systems not already covered by an instrumented library; `PRODUCER`/`CONSUMER` for messaging.
- Every span carries an `operation` attribute whose value equals the span name, so a span and its log records pair up even in stdout, where there is no `trace_id`.
- Attribute keys use the OpenTelemetry semantic conventions whenever one exists (`messaging.system`, `messaging.destination.name`, `messaging.operation.type`, `db.system`, `dns.question.name`, `http.request.method`, `url.path`, `http.response.status_code`). Otherwise they are lowercase, dot-namespaced by domain concept (`<namespace>.<property>`, with snake_case inside a segment when needed: `alarm.id`, `isp.differs`, `ip.found`), defined as package-level constants.
- Log keys (camelCase) and span or metric attribute keys (dotted) are separate namespaces: never reuse a log-key constant as a span or metric attribute, or the other way round.

### Metrics

- Meter name: the package import path, like the tracer name.
- Instruments are created once, in the component's constructor; a registration error is logged and otherwise ignored, because telemetry must never prevent the component from being built.
- Metric names are lowercase and dot-separated, prefixed with the application name without dashes: `apothikarios.<entity>[.<qualifier>]`.
- Units use UCUM: `s` for durations (recorded as a `Float64Histogram` in seconds, with explicit bucket boundaries) and annotations in braces for counts (`{run}`, `{message}`).
- Metric attributes follow the span attribute key rules and must have bounded cardinality: never raw IPs, free text or unbounded identifiers.

## Log Message Style

This convention applies across all my projects (this file is replicated in each one).

Log messages (the message string passed to the logger, not code comments) follow these rules:

- They start in lowercase, with one exception: when the first word is an acronym or a product/proper name (`ISP name has been set`, `DNS server has been set`, `IPInfo request succeeded`, `Redis config has been set`), it keeps its canonical casing — never decapitalize an acronym.
- They never end with a period.
- Acronyms and product names keep their canonical casing anywhere in the message: `DNS`, `HTTP`, `IP`, `ISP`, `Redis`, `RabbitMQ`, `IPInfo`. Names that refer to this project's own packages (`ipinfo`, `nslookup`, `messagebroker`, `memorydatabase`) stay lowercase, since they name the package, not a product.

Strings that are not log messages — notification payloads sent to queues, error strings for `errors.New`/`fmt.Errorf` (which follow the Go convention: lowercase, no period) — are out of scope of the first rule's exception list but must not be confused with logs when auditing.

When auditing code, flag log messages that deviate from these rules.

## OpenTelemetry: Span Error Recording Policy

This policy applies across all my projects (this file is replicated in each one).

The error _event_ (`span.RecordError`) is recorded exactly once, in the span closest to where the error happens: the deepest instrumented span, or the current span when the failing call has no span of its own. Every ancestor span up the chain marks `span.SetStatus(codes.Error, ...)` only — the whole branch shows as failed in the trace without duplicating the same event at every level.

When auditing instrumentation, enforce the policy in both directions:

- Flag a `RecordError` on an error that an instrumented callee already records (duplicate event).
- Flag a status-only error path whose callee has no instrumented span (red span with no event explaining it).

Deciding which case applies usually requires reading the callee's code, not assuming. These projects share the `go-types`/`go-services` libraries (`github.com/a-castellano/...`): when the failing call crosses into them, check the library source — cloned as sibling directories of this project, or in the module cache — to confirm whether its spans record the error at that path.

## OpenTelemetry: Span Attributes at Start

This convention applies across all my projects (this file is replicated in each one).

Attributes whose values are known when the span is created are declared in the single `Start` call, via `trace.WithAttributes(...)` — not in a separate `span.SetAttributes` immediately after. Besides reading better, start-time attributes are visible to samplers deciding whether to keep the trace; attributes added afterwards are not.

`span.SetAttributes` remains the right tool for values only known mid-flow (outcomes, flags computed during the operation). When auditing, flag a `SetAttributes` right after `Start` whose values were already available at creation time.

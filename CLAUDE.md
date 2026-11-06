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

There may be a directory called `internal-context/` at the root of the project (it is git-ignored and will not appear in the repository). It contains a internal context with milestones and progress notes about the functionality currently being worked on.

When asked to review or update the context, look for files inside `internal-context/` and treat them as the source of truth for current goals and progress.

Because `internal-context/` is git-ignored, documentation must never reference any document inside the `internal-context/` directory. Versioned docs would end up with broken links for anyone who does not have the internal-context locally. Keep internal-context references confined to the `internal-context/` directory itself.

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

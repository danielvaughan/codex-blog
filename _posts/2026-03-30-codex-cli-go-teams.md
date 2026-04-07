---
title: "Codex CLI for Go Teams: Skills, AGENTS.md and Go 1.26 Workflows"
parent: "Articles"
nav_order: 104
---

# Codex CLI for Go Teams: Skills, AGENTS.md and Go 1.26 Workflows


Go's explicit error handling, strict formatting conventions, and idiomatic concurrency model make it both an ideal and a demanding language for agentic workflows. The agent needs to know your module path, your test tag conventions, your linter configuration, and the single-handling rule — none of which it can infer from the code alone. This article covers how to configure Codex CLI precisely for Go development: the `cc-skills-golang` skill library, a production-grade `AGENTS.md` template updated for Go 1.26[^1], and CI/CD patterns that keep generated code clean.

## The cc-skills-golang Skill Library

The `cc-skills-golang` repository by Samuel Berthe provides 20+ production-ready Codex skills for Go development.[^2] Skills load on demand, keeping context windows lean — the lazy-loading model means the full skill body is only injected when the agent invokes the skill by name.

### Available Skills

The core set for most services:

| Skill | What It Enforces |
|---|---|
| `golang-code-style` | `gofmt` conventions, import ordering, comment style |
| `golang-concurrency` | Goroutine ownership, channel rules, leak prevention, `time.NewTimer` vs `time.After` |
| `golang-context` | Context propagation, deadline patterns, cancellation hierarchies |
| `golang-error-handling` | `%w` wrapping, sentinel vs typed errors, single-handling rule, `slog` structured logging |
| `golang-testing` | Table-driven tests, `t.Parallel()`, `goleak`, integration build tags, fuzz targets |
| `golang-security` | `gosec`, `govulncheck`, injection prevention, `crypto/rand` vs `math/rand` |
| `golang-observability` | `log/slog` structured attributes, OpenTelemetry patterns |
| `golang-modernize` | Applies Go 1.26 modernizers via `go fix ./...` |
| `golang-performance` | Profile-first methodology, allocation reduction, `benchstat` |
| `golang-naming` | Package naming, receiver conventions, exported vs unexported identifiers |
| `golang-stretchr-testify` | Testify `assert`/`require`/`mock`/`suite` usage patterns |
| `golang-grpc` | gRPC interceptors, streaming patterns, status codes |

The `golang-security` and `golang-error-handling` skills both expose an **audit mode** that deploys multiple parallel sub-agents, each covering a distinct attack surface or error pattern.[^2]

### Installation

```bash
# Install all skills via npx
npx skills add https://github.com/samber/cc-skills-golang --skill '*'

# Or via Codex plugin command
codex plugin install cc-skills-golang@samber
```

Skills are discovered from `.agents/skills/` in the working directory, parent directories up to the Git root, and `~/.agents/skills/` for user-level installs.[^3]

Disable skills that don't apply to your stack in `.codex/config.toml` at the repo root:

```toml
# .codex/config.toml
[[skills.config]]
path = "/home/user/.agents/skills/cc-skills-golang/skills/golang-grpc/SKILL.md"
enabled = false
```

## AGENTS.md Template for a Go Repository

AGENTS.md is now stewarded by the Agentic AI Foundation under the Linux Foundation as an open standard.[^4] Codex loads it from outermost to innermost (global → container root → repo root → subdirectory), merging with closer files taking precedence. The active prompt always wins.

### Root AGENTS.md

```markdown
# AGENTS.md

## Repository Overview
Go 1.26+ service. Module path: `github.com/your-org/your-service`.
Production code under `internal/`. Public packages under `pkg/`.
`cmd/` holds only `main.go` entrypoints with minimal logic.

## Build Commands
```bash
go build ./...
go fix ./...        # Apply Go 1.26 modernizers before any PR
```

## Test Commands

```bash
# Unit tests (always run with race detector)
go test -race ./...

# With coverage
go test -race -coverprofile=coverage.out -covermode=atomic ./...

# Short mode (skip integration tests)
go test -short ./...

# Integration tests (require //go:build integration tag)
go test -race -tags=integration ./...
```

## Lint Commands

```bash
go vet ./...
golangci-lint run ./...
gosec ./...
govulncheck ./...
```

## Verification Sequence

Before any commit, run in order:

1. `go mod tidy`
2. `go fmt ./...`
3. `go fix ./...`
4. `go vet ./...`
5. `golangci-lint run ./...`
6. `go test -race ./...`

## Code Style

- Error messages: lowercase, no trailing punctuation
- Wrap with context: `fmt.Errorf("connecting to database: %w", err)`
- Use `%w` inside error chains; `%v` only at top-level boundaries
- Never discard errors with `_`
- Use `errors.Is()` / `errors.As()` for inspection
- Use `errors.AsType[T]()` (Go 1.26) for type-safe unwrapping in new code
- Log with `log/slog` using structured attributes, never `fmt.Println`

## Testing Standards

- Table-driven tests with `t.Run` and named `name` fields
- Mark independent tests `t.Parallel()`
- Tag integration tests: `//go:build integration` at the top of the file
- Use `require` (testify) for fatal assertions; `assert` for non-fatal
- Use `goleak.VerifyTestMain` in any package that spawns goroutines

## Concurrency Rules

- Every goroutine must have a defined exit strategy before it is spawned
- Only the sender closes a channel
- Always include `ctx.Done()` in select statements
- Never use `time.After` in loops — use `time.NewTimer` with `Reset`
- Use channels for ownership transfer; `sync.Mutex` for shared struct state

## Module Rules

- Run `go mod tidy` and commit both `go.mod` and `go.sum` after any import change
- Manage tool dependencies with `go get -tool` (Go 1.24+)
- Use `go.work` for local multi-module development; do not commit unless this is a monorepo

## Go 1.26 Specifics

- Use `new(expr)` for pointer initialisation when the value fits a single expression
- Use `errors.AsType[T]()` for type-safe error unwrapping in new code
- Run `go fix ./...` as part of the pre-PR workflow
- `go tool doc` is removed — use `go doc` directly

```

For a high-trust subdirectory (e.g., a payments service), layer a subdirectory `AGENTS.md` override:

```markdown
# services/payments/AGENTS.md

## Security Requirements (override root)
All changes must pass `gosec -severity high ./services/payments/...`.
Treat medium-severity findings as blocking for new code.
Run `govulncheck ./services/payments/...` and fail on any known vulnerability.
```

## Go 1.26 and Its Impact on Generated Code

Go 1.26 shipped on 10 February 2026.[^1] Three language changes directly affect what the agent generates:

**`new()` accepts expressions.** The `golang-modernize` skill will suggest this for pointer fields in JSON/Protobuf structs:

```go
// Before Go 1.26
x := yearsSince(born)
person.Age = &x

// Go 1.26
person.Age = new(yearsSince(born))
```

**`errors.AsType[E]()`** provides compile-time type safety over the reflection-based `errors.As()`:

```go
// Old
var e *MyError
if errors.As(err, &e) { ... }

// Go 1.26
if e, ok := errors.AsType[*MyError](err); ok { ... }
```

The `golang-error-handling` skill references this pattern. Add your team's preference to AGENTS.md so the agent applies it consistently.

**Revamped `go fix`.** `go fix ./...` is now the home of Go's modernizer suite, running on the same analysis framework as `go vet`.[^1] Add it to your verification sequence — and add a `git diff --exit-code` check in CI to catch any uncommitted modernizer output.

The **Green Tea GC** is now the default, delivering 10–40% lower GC overhead without code changes.[^5] The new `goroutineleak` pprof profile type complements `goleak` in tests by detecting permanently blocked goroutines at runtime.

## CI/CD Integration

golangci-lint v2.11.4 is the current stable release (22 March 2026).[^6] Install it as a `go get -tool` dependency to keep versions pinned in `go.mod`:

```bash
go get -tool github.com/golangci/golangci-lint/cmd/golangci-lint@v2.11.4
go tool golangci-lint run ./...
```

A minimal `.golangci.yml` for Go 1.26 projects:

```yaml
version: "2"

run:
  timeout: 5m

linters:
  default: standard
  enable:
    - errcheck
    - errorlint      # error wrapping correctness
    - gosec
    - staticcheck
    - wrapcheck      # errors from external packages must be wrapped
    - noctx          # HTTP requests without context

linters-settings:
  gosec:
    severity: medium
    confidence: medium
  errorlint:
    errorf: true
    asserts: true

issues:
  exclude-rules:
    - path: "_test\\.go"
      linters: [gosec, errcheck]
```

For GitHub Actions, use `golangci/golangci-lint-action@v6` and `govulncheck` in a dedicated security job:

```yaml
- name: govulncheck
  run: |
    go install golang.org/x/vuln/cmd/govulncheck@latest
    govulncheck ./...

- name: golangci-lint
  uses: golangci/golangci-lint-action@v6
  with:
    version: v2.11.4
    args: --timeout=5m
```

For non-interactive Codex tasks in CI, the `ci` profile with `approval_policy = "never"` is the appropriate default:[^7]

```toml
# .codex/config.toml
[profiles.ci]
model = "gpt-5.4"
approval_policy = "never"
sandbox_mode = "workspace-write"
model_reasoning_effort = "medium"
```

```bash
# Run a CI fix pass non-interactively
codex exec --profile ci "Fix all golangci-lint errors in ./internal/payments/..."
```

## Subagent Audit Patterns

When `multi_agent = true` is enabled in `config.toml`,[^8] the `golang-security` and `golang-error-handling` skills can deploy parallel sub-agents covering distinct concerns. Triggering a security audit:

```
# In an interactive Codex session
$golang-security audit ./...
```

This spawns five parallel sub-agents covering injection vulnerabilities, cryptographic operations, web security, authentication flows, and concurrency race conditions — each scoped to a specific attack surface rather than doing a single serial pass.

The `golang-testing` skill provides four modes: **write** (generates table-driven tests via `gotests` then adds edge cases), **review** (checks PR test changes for coverage regressions), **audit** (parallel sub-agents for unit quality, integration isolation, and race conditions), and **debug** (reproduces failing assertions with a minimal reproduction case).[^2]

## The Single-Handling Rule

The most common Go anti-pattern that agents reproduce is logging an error *and* returning it — causing it to be logged twice at the caller. Embed the rule explicitly in AGENTS.md and the `golang-error-handling` skill will reinforce it:

```go
// Wrong: error logged AND propagated
if err := validate(id); err != nil {
    slog.Error("validation failed", "err", err) // logged here
    return err                                   // AND propagated — double log
}

// Correct: propagate with context; log once at the top-level boundary
if err := validate(id); err != nil {
    return fmt.Errorf("processing order %s: %w", id, err)
}
```

At the HTTP handler boundary, log once:

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    if err := h.svc.processOrder(r.Context(), id); err != nil {
        slog.Error("create order failed", "order_id", id, "err", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
    }
}
```

## Summary

The combination of a well-structured `AGENTS.md`, the `cc-skills-golang` skill library, and a `config.toml` `ci` profile gives Go teams a repeatable, tool-aware agentic workflow. AGENTS.md carries the conventions Codex cannot infer; skills provide the idiomatic depth (concurrency, error handling, security) that generic prompts miss; and the `ci` profile makes non-interactive execution safe enough to run in pull-request pipelines.

## Citations

[^1]: Go 1.26 Release Notes — [https://go.dev/doc/go1.26](https://go.dev/doc/go1.26)
[^2]: cc-skills-golang repository — [https://github.com/samber/cc-skills-golang](https://github.com/samber/cc-skills-golang)
[^3]: Codex CLI Skills Documentation — [https://developers.openai.com/codex/skills](https://developers.openai.com/codex/skills)
[^4]: AGENTS.md Open Standard (Agentic AI Foundation / Linux Foundation) — [https://agents.md/](https://agents.md/)
[^5]: Go 1.26 Release Blog — [https://go.dev/blog/go1.26](https://go.dev/blog/go1.26)
[^6]: golangci-lint v2.11.4 Release — [https://github.com/golangci/golangci-lint/releases](https://github.com/golangci/golangci-lint/releases)
[^7]: Codex CLI Non-Interactive Mode Reference — [https://developers.openai.com/codex/noninteractive](https://developers.openai.com/codex/noninteractive)
[^8]: Codex CLI Features Reference — [https://developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features)

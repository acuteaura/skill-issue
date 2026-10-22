---
name: go-conventions
description: Use when writing, reviewing, or scaffolding Go code — choosing an HTTP router or web framework, structuring a new service or package, or deciding whether to add a third-party module.
user-invocable: false
---

# Go Conventions

One blessed choice per concern, stdlib first. Deviating is allowed but must be justified in the PR description, not silently.

## Quick Reference

| Concern | Default | Escape hatch | Detail |
|---|---|---|---|
| HTTP routing | `net/http.ServeMux` | `gofiber/fiber/v3` under a measured throughput requirement | [go-fiber](../go-fiber/SKILL.md) |
| Dependency injection | `go.uber.org/fx` | Plain constructors in `main()` for tools with <5 components | [go-fx](../go-fx/SKILL.md) |
| Logging | `log/slog` | None | [go-slog](../go-slog/SKILL.md) |
| Test assertions, fake time | `testify/assert`, `clockwork` | None — never `time.Sleep` to synchronise | [go-testing](../go-testing/SKILL.md) |
| Anything else | stdlib | A well-maintained library — see below | — |

## HTTP Routing

**Default to `net/http.ServeMux`.** Since Go 1.22 it does method and wildcard matching, which covers most services with zero dependencies.

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /v1/repositories/{id}", h.Get)
mux.HandleFunc("POST /v1/repositories", h.Create)

// in the handler:
id := r.PathValue("id")
```

Middleware is just `func(http.Handler) http.Handler`. Compose by wrapping; no framework needed.

**Fiber is for when high performance is an actual, measured requirement** — not a hunch that a framework will be faster. It is fasthttp-based, so stdlib middleware, `httptest`, and `http.Handler` interop all stop working. Read [go-fiber](../go-fiber/SKILL.md) before choosing it, and never mix both in one binary.

## Dependency Policy

Two failure modes, equally bad:

**Adding a dependency for something stdlib does.** No `github.com/pkg/errors` (use `fmt.Errorf("...: %w", err)` with `errors.Is`/`errors.As`), no assertion DSLs beyond testify, no utility grab-bags, no router for `ServeMux`-shaped problems. `slices`, `maps`, `cmp`, and `errors` cover most of what people reach for libraries for.

**Hand-rolling something genuinely hard.** Do not write your own crypto, JWT/OIDC verification, retry-with-backoff, rate limiter, SQL driver, YAML parser, UUID generator, or connection pool. These have subtle failure modes and you will get them wrong. Take the well-maintained library.

The test: *would a bug here be obvious in review and cheap in production?* Yes → write it. No → import it.

Before adding any module, check that it is actively maintained, has no huge transitive tree, and isn't already solved by something in `go.mod`.

## Package Shape

- One feature per package, exposing a single constructor/provider as its public surface.
- **Depend on interfaces, return concrete types.** Define the interface in the consuming package, not alongside the implementation.
- `cmd/<binary>/main.go` holds the component list and nothing else — no business logic, no `http` handlers.
- Errors wrap with `%w` and context: `fmt.Errorf("fetching repository %s: %w", id, err)`. Sentinel errors over string matching.
- Accept `context.Context` as the first parameter on anything doing I/O, and actually honour it.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Fiber chosen for "it's faster" with no measurement | `ServeMux` unless throughput is a stated requirement |
| `github.com/pkg/errors` in new code | `fmt.Errorf` with `%w` |
| Hand-rolled backoff/rate limiter/JWT verification | Take the library — these fail subtly |
| Interface defined next to its only implementation | Define it where it is consumed |
| Business logic in `main()` | A provider in a feature package |
| Global singletons reached via package vars | Provide it through fx |

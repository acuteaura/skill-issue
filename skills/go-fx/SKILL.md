---
name: go-fx
description: Use when wiring Go components together — adding a constructor or provider, giving a component start/stop behaviour, collecting plural things like routers or workers, or working in a project whose go.mod contains go.uber.org/fx.
user-invocable: false
---

# Dependency Injection with uber fx

`go.uber.org/fx` is the blessed DI container. Components are constructor functions ("providers") returning their type plus an error. `main()` only lists providers and invocations; it never news anything up itself.

```go
func NewRepositoryService(in RepositoryIn) (*RepositoryService, error) { ... }

type RepositoryIn struct {
    fx.In
    Config *Config
    Log    *slog.Logger
    DB     *sql.DB
}
```

## Rules

- **Use `fx.In`/`fx.Out` structs once a constructor has 3+ dependencies.** Positional parameters get unreadable and churn on every change.
- **Collect plural things with group tags**, not with a registry packages write into. Each router package provides one `fx.Out` with `group:"router"`; the server consumes `[]*RouterSpec` with the same tag. Adding a router is then one `fx.Provide` line in `main()`.
- **Group ordering is not deterministic.** Never let one group member depend on being constructed or registered before another.
- **Anything with a start/stop uses `fx.Lifecycle`.** `OnStart` must not block — do the fallible setup (bind the port, open the connection) synchronously so failures abort startup, then spawn the goroutine and return.
- **`fx.Provide` is lazy; `fx.Invoke` is the root.** A provider nobody depends on never runs. That is the design, not a bug.
- **Optional dependencies use `optional:"true"`** on an `fx.In` field; it is the zero value when unprovided.
- **Bridge fx's own logs into slog** with `fx.WithLogger(func(l *slog.Logger) fxevent.Logger { return &fxevent.SlogLogger{Logger: l} })`. Do not leave the default stdout spew.
- **Check `app.Err()` before `app.Run()`** so wiring mistakes fail loudly at startup with a precise message.
- **A background worker that must kill the app on failure takes `fx.Shutdowner`** and calls `Shutdown()`.

## Testing

**Do not boot fx to test a component.** Build the `fx.In` struct by hand with the fakes you want and call the provider directly — that is the whole benefit of constructor injection.

```go
out, err := repository.Provider(repository.RouterIn{
    Log:   slog.New(slog.NewTextHandler(io.Discard, nil)),
    Clock: clockwork.NewFakeClock(),
    Store: repository.NewMemoryStore(),
})
require.NoError(t, err)
```

Use `fxtest.New` / `RequireStart` / `RequireStop` only when the lifecycle itself is under test. See [go-testing](../go-testing/SKILL.md) for assertion and fake-clock conventions.

## Debugging the Graph

- `app.Err()` names the missing or duplicated type — read it, it is usually precise.
- `fx.Populate(&target)` extracts a built value in a test without an `Invoke`.
- Two providers for the same type is an error, not last-write-wins. Use `fx.Annotate` with `fx.ResultTags` to name them apart, or a group.
- "Provider never ran" almost always means nothing in the `Invoke` chain depends on it.

## Full Worked Example

A complete compiling skeleton — `main.go`, config, slog, a group-collected router, HTTP server lifecycle, and a test that never boots fx: [references/fx-wiring.md](references/fx-wiring.md).

## Common Mistakes

| Mistake | Fix |
|---|---|
| Constructor with 7 positional args | `fx.In` struct |
| Central registry file listing every router | `group:"router"` tag |
| Blocking work inside `fx.Hook.OnStart` | Fallible setup synchronously, then a goroutine |
| `fx.New` in a handler unit test | Call the provider directly with fakes |
| Panicking inside a provider on bad config | Return an error; fx reports it with the graph path |
| Reaching for a package-level singleton | Provide it |

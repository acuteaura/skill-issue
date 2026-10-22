---
name: go-slog
description: Use when adding, changing, or reviewing logging in Go — setting up a logger, logging inside a handler or worker.
user-invocable: false
---

# Structured Logging with log/slog

Stdlib `log/slog`, nothing else, unless the codebase already extensively uses it.

## Setup

```go
func NewLogger(cfg *Config) *slog.Logger {
    opts := &slog.HandlerOptions{Level: cfg.LogLevel}

    var h slog.Handler
    if term.IsTerminal(int(os.Stderr.Fd())) {
        h = slog.NewTextHandler(os.Stderr, opts)
    } else {
        h = slog.NewJSONHandler(os.Stderr, opts)
    }

    log := slog.New(h)
    slog.SetDefault(log) // catch libraries that use the package-level funcs
    return log
}
```

Text handler for TTYs, JSON otherwise. `slog.SetDefault` is for third-party code you don't control — your own code takes the logger as a dependency.

## Rules

- **Provide `*slog.Logger` and accept it explicitly.** Package-level `slog.Info` in your own code hides a dependency and makes tests noisy. See [go-fx](../go-fx/SKILL.md).
- **Attributes, not formatting:**

```go
log.Info("registered router", "prefix", prefix, "version", v)   // yes
log.Info(fmt.Sprintf("registered %s at v%d", prefix, v))        // no
```

The whole point is that the fields are machine-queryable. A formatted string is a grep target, not a field.

- **Use the `Context` variants in request paths:** `log.InfoContext(ctx, "repository not found")`. Handlers that carry trace or request IDs in the context get them onto the line.
- **Scope with `With` instead of repeating a field:**

```go
log := r.log.With("repository_id", id)
```

- **Errors go in an attribute, not the message:** `log.Error("encoding response", "error", err)`. The message stays a stable constant you can group by.
- **Message is a constant.** Never interpolate into it — not even an ID. Constant messages are what make aggregation work.
- **Keys are `snake_case` and stable** across a service: `repository_id`, not `repoID` here and `repo` there.
- **Levels:** `Debug` for things useful only while chasing a bug, `Info` for lifecycle and request outcomes, `Warn` for degraded-but-handled, `Error` for things a human should look at. An `Error` that fires routinely is a `Warn` or a bug.
- **Do not log and return the error.** Wrap it (`fmt.Errorf("...: %w", err)`) and let the top of the stack log it once — otherwise one failure produces five lines.
- **Never log credentials, tokens, or full request bodies.** Log the identifier and the shape.

## Testing

Pass `slog.New(slog.NewTextHandler(io.Discard, nil))` so test output stays clean; swap in `os.Stderr` while debugging a single test. If log output itself is the assertion, write to a `bytes.Buffer` with the JSON handler and parse it.

## Common Mistakes

| Mistake | Fix |
|---|---|
| `log.Info(fmt.Sprintf("user %s failed", id))` | `log.Info("user failed", "user_id", id)` |
| Package-level `slog.Info` throughout your own code | Inject `*slog.Logger` |
| Logging the error, then returning it too | Wrap with `%w`, log once at the top |
| `Error` level for an expected 404 | `Info`, or `Debug` |
| Repeating the same field on every line in a handler | `log := log.With(...)` once |
| Adding zerolog/zap to a new service | `log/slog` |

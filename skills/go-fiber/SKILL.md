---
name: go-fiber
description: Use when a Go service already uses gofiber/fiber, when migrating between Fiber v2 and v3, or when deciding whether a measured throughput requirement justifies leaving net/http.
user-invocable: false
---

# Fiber

Fiber is the escape hatch from `net/http.ServeMux`, not the default. Reach for it when **high performance is a stated, measured requirement** — a profiled hot path, or a service whose entire job is shovelling many small requests.

## The Cost You Are Paying

Fiber is built on fasthttp, not `net/http`. That means:

- `fiber.Ctx` is not `http.ResponseWriter` + `*http.Request`. Every stdlib middleware you own stops working.
- `httptest.NewRecorder` does not apply. Testing goes through `app.Test(req)` instead.
- Any library taking an `http.Handler` — OTel instrumentation, oauth middleware, pprof, your existing auth layer — needs a Fiber-specific port or an adaptor.
- fasthttp reuses request objects. **A `[]byte` or string from the context is only valid for that request**; retaining one past the handler gives you another request's data. Use `utils.CopyString` or `c.Bind()` into a struct when holding on to anything.

That last one is a real data-leak class of bug, not a style note. It is the main reason this is not the house default.

Never mix Fiber and `net/http` routing in one binary.

## v3, Not v2

New services use v3. Differences that bite when copying v2 examples or snippets off the internet:

| v2 | v3 |
|---|---|
| `func(c *fiber.Ctx) error` | `func(c fiber.Ctx) error` — an **interface**, no pointer |
| `c.BodyParser(&v)` | `c.Bind().JSON(&v)` / `c.Bind().Body(&v)` |
| `app.Listen(addr)` | `app.Listen(addr, ...ListenConfig)` |
| `c.Params("id")` | unchanged |

## Shape

```go
app := fiber.New(fiber.Config{
    ErrorHandler: func(c fiber.Ctx, err error) error {
        // one place that turns an error into a response
        return c.Status(http.StatusInternalServerError).JSON(errResponse{Error: "internal"})
    },
})

app.Use(func(c fiber.Ctx) error {
    // middleware is a handler that calls Next
    return c.Next()
})

app.Get("/v1/repositories/:id", func(c fiber.Ctx) error {
    id := c.Params("id")
    repo, ok := store.Get(id)
    if !ok {
        return c.SendStatus(http.StatusNotFound)
    }
    return c.JSON(repo)
})
```

Set a central `ErrorHandler` in `fiber.Config` and have handlers `return err` rather than writing error responses inline. That is the one Fiber idiom worth adopting wholesale.

## With fx

`Listen` blocks, so it goes in a goroutine — same split as the stdlib server, and the same rule applies: fallible setup synchronously, blocking work in the goroutine.

```go
func NewFiberServer(lc fx.Lifecycle, shutdown fx.Shutdowner, app *fiber.App, cfg *Config, log *slog.Logger) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            ln, err := net.Listen("tcp", cfg.ListenAddr)
            if err != nil {
                return err // port clash aborts startup
            }
            log.Info("listening", "addr", ln.Addr().String())

            go func() {
                if err := app.Listener(ln, fiber.ListenConfig{DisableStartupMessage: true}); err != nil {
                    log.Error("server stopped", "error", err)
                    _ = shutdown.Shutdown()
                }
            }()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            return app.ShutdownWithContext(ctx)
        },
    })
}
```

`DisableStartupMessage` matters — the ASCII banner is noise in structured logs. See [go-fx](../go-fx/SKILL.md) for the surrounding wiring and [go-slog](../go-slog/SKILL.md) for logging.

## Testing

`httptest` is out; use `app.Test`:

```go
req := httptest.NewRequest(http.MethodGet, "/v1/repositories/1", nil)
resp, err := app.Test(req)
require.NoError(t, err)
assert.Equal(t, http.StatusOK, resp.StatusCode)
```

`app.Test` takes an optional `fiber.TestConfig` for timeouts. Everything else from [go-testing](../go-testing/SKILL.md) applies unchanged.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Choosing Fiber for "it's faster" with no measurement | `net/http.ServeMux` |
| Storing `c.Params(...)` / `c.Body()` past the handler | `utils.CopyString`, or bind into a struct |
| Copying a v2 snippet with `*fiber.Ctx` | v3 uses `fiber.Ctx`, no pointer |
| `c.BodyParser` | `c.Bind().JSON(&v)` |
| Error responses written inline in every handler | Central `ErrorHandler` in `fiber.Config`; `return err` |
| `httptest.NewRecorder` | `app.Test(req)` |
| Blocking `app.Listen` inside `fx.Hook.OnStart` | Bind synchronously, serve in a goroutine |
| Fiber and `net/http` routes in one binary | Pick one |

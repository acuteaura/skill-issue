---
name: go-testing
description: Use when writing, reviewing, or fixing Go tests — choosing assertions, structuring table tests, testing code that depends on time.Now or timers, or chasing a flaky test that passes locally and fails in CI.
user-invocable: false
---

# Go Testing Conventions

testify for assertions, clockwork for time. Nothing else.

## Assertions — testify

**`assert` by default.** It records the failure and lets the test continue, so one run tells you everything that is wrong.

```go
assert.Equal(t, http.StatusOK, w.Code)
assert.Len(t, got.Items, 3)
assert.ErrorIs(t, err, ErrNotFound)
```

**`require` only for preconditions** — when continuing past the failure would panic or make every later assertion meaningless:

```go
svc, err := NewRepositoryService(in)
require.NoError(t, err) // nothing below works if this failed
assert.Equal(t, "ready", svc.Status())
```

The distinction is not stylistic. `assert.NoError` followed by a method call on a nil result turns a clear failure into a nil-pointer panic with a useless stack trace. `require` everywhere is the opposite problem: you fix one assertion, rerun, find the next.

Use the specific matcher over a hand-rolled boolean — `assert.ErrorIs`, `assert.Len`, `assert.Contains`, `assert.ElementsMatch` all print the actual values; `assert.True(t, len(x) == 3)` prints `false`.

## Time — clockwork

**Never `time.Sleep` to wait for something to happen.** It is slow when it works and flaky when it doesn't.

Inject `clockwork.Clock` into anything that reads the time or sets a timer. Production provides `clockwork.NewRealClock()`; tests use a fake.

```go
clock := clockwork.NewFakeClock()
cache := NewCache(clock, time.Minute)

cache.Put("k", "v")
clock.Advance(90 * time.Second)
assert.False(t, cache.Has("k"))
```

The interface covers `Now`, `Since`, `Until`, `Sleep`, `After`, `NewTimer`, `NewTicker`, `AfterFunc` — enough that production code never imports `time` for anything but durations.

**For code that blocks on the clock from another goroutine, `BlockUntil` first:**

```go
go worker.Run(ctx)            // calls clock.Sleep internally
clock.BlockUntil(1)           // wait until the worker is actually waiting
clock.Advance(time.Hour)      // now this cannot fire into the void
```

Without `BlockUntil`, `Advance` can run before the goroutine registers its waiter — the test then hangs or passes by luck. This is the single most common cause of a clockwork test that is flaky in CI only.

`clockwork.NewFakeClockAt(t)` pins a start time when the absolute date matters. `clockwork.WithTimeout(ctx, clock, d)` gives a context deadline driven by the fake clock.

## Shape

- **Table-driven with subtests** is the default:

```go
tests := []struct {
    name     string
    path     string
    wantCode int
}{
    {name: "found", path: "/v1/repositories/1", wantCode: http.StatusOK},
    {name: "missing", path: "/v1/repositories/99", wantCode: http.StatusNotFound},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        ...
    })
}
```

- **Name the case field `name`** and pass it to `t.Run` — failures then say which case, not `#3`.
- **`t.Cleanup` over `defer`** in helpers, so cleanup survives the helper returning.
- **`t.Helper()` at the top of every helper**, so failures point at the caller.
- **Package `foo_test`, not `foo`**, unless you genuinely need unexported access. It keeps you honest about the public surface.
- **Discard logs:** `slog.New(slog.NewTextHandler(io.Discard, nil))`. Swap to `os.Stderr` while debugging one test.
- **Fakes over mocking frameworks.** A ten-line struct implementing the interface beats generated mocks and reads better in review.
- **Test the provider, not the container** — see [go-fx](../go-fx/SKILL.md).

## Common Mistakes

| Mistake | Fix |
|---|---|
| `time.Sleep(100 * time.Millisecond)` to let something settle | Inject `clockwork.Clock`, `Advance` it |
| `Advance` without `BlockUntil` on concurrent code | `clock.BlockUntil(n)` first |
| `require` for every assertion | `assert` by default |
| `assert.NoError` then dereferencing the result | `require.NoError` — it is a precondition |
| `assert.True(t, got == want)` | `assert.Equal` — it prints both values |
| Shared mutable fixture across subtests | Build fresh state inside `t.Run` |
| Real clock plus a generous timeout to "fix" flakiness | Fake clock; the flake is a missing synchronisation point |

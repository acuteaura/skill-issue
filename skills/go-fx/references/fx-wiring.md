# fx Wiring Reference

A complete, compiling service skeleton: config, slog, a group-collected router, HTTP server lifecycle, and a test that never boots fx. Every snippet below is from a module that builds, vets clean, passes `go test`, and serves real requests.

Shape:

```
cmd/main.go            — the only place that knows the full component list
core/config.go         — config provider
core/logger.go         — *slog.Logger provider
core/core.go           — RouterSpec, RouterOut, the mux assembler
core/server.go         — *http.Server with lifecycle hooks
repository/router.go    — one feature package: Provider + handlers
repository/router_test.go
```

## main.go — providers and one invoke

```go
package main

import (
	"log/slog"

	"github.com/jonboulle/clockwork"
	"go.uber.org/fx"
	"go.uber.org/fx/fxevent"

	"example.com/fxdemo/core"
	"example.com/fxdemo/repository"
)

func main() {
	app := fx.New(
		fx.WithLogger(func(log *slog.Logger) fxevent.Logger {
			return &fxevent.SlogLogger{Logger: log}
		}),

		fx.Provide(
			core.NewConfig,
			core.NewLogger,
			core.NewHandler,
			clockwork.NewRealClock,
		),

		// routers — one line per package, no central registry
		fx.Provide(
			repository.Provider,
		),

		fx.Provide(repository.NewMemoryStore),

		// fx.Provide is lazy: without an Invoke nothing is constructed.
		fx.Invoke(core.NewServer),
	)

	if err := app.Err(); err != nil {
		panic(err) // wiring mistakes fail loudly, before anything starts
	}

	app.Run()
}
```

`clockwork.NewRealClock` is provided directly — its signature is already `func() Clock`, so it needs no wrapper. Tests substitute a `*FakeClock` without fx being involved.

## core — the router group

The group tag is what keeps `main()` from growing a registry file. Each feature package emits one `RouterOut`; `NewHandler` receives all of them as a slice.

```go
package core

import (
	"fmt"
	"log/slog"
	"net/http"

	"go.uber.org/fx"
)

// RouterSpec is what every router package contributes to the server.
type RouterSpec struct {
	Prefix   string // "repositories"
	Version  uint   // 1
	Register func(mux *http.ServeMux, prefix string)
}

// RouterOut is embedded by each router package's provider so fx collects them
// into the "router" group without a central registry file.
type RouterOut struct {
	fx.Out

	Router *RouterSpec `group:"router"`
}

type ServerIn struct {
	fx.In

	Config  *Config
	Log     *slog.Logger
	Routers []*RouterSpec `group:"router"`

	// Optional dependencies use the optional tag; nil when nobody provides one.
	Middleware []func(http.Handler) http.Handler `optional:"true"`
}

type Handler struct {
	mux *http.ServeMux
	log *slog.Logger
}

func NewHandler(in ServerIn) (*Handler, error) {
	h := &Handler{mux: http.NewServeMux(), log: in.Log}

	for _, r := range in.Routers {
		prefix := fmt.Sprintf("/v%d/%s", r.Version, r.Prefix)
		in.Log.Info("registering router", "prefix", prefix)
		r.Register(h.mux, prefix)
	}

	return h, nil
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	h.mux.ServeHTTP(w, r)
}
```

Group ordering is **not** deterministic. Never let one router depend on being registered before another.

## core/server.go — lifecycle

The split matters: bind synchronously so a port clash fails startup, then serve in a goroutine so `OnStart` returns promptly.

```go
func NewServer(lc fx.Lifecycle, shutdown fx.Shutdowner, h *Handler, cfg *Config, log *slog.Logger) *http.Server {
	srv := &http.Server{
		Handler:           h,
		Addr:              cfg.ListenAddr,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       time.Minute,
		IdleTimeout:       time.Minute,
	}

	lc.Append(fx.Hook{
		OnStart: func(ctx context.Context) error {
			// Bind synchronously so a port clash fails startup...
			ln, err := net.Listen("tcp", cfg.ListenAddr)
			if err != nil {
				return err
			}
			log.Info("listening", "addr", ln.Addr().String())

			// ...but never block OnStart on Serve.
			go func() {
				if err := srv.Serve(ln); err != nil && !errors.Is(err, http.ErrServerClosed) {
					log.Error("server stopped", "error", err)
					_ = shutdown.Shutdown()
				}
			}()
			return nil
		},
		OnStop: func(ctx context.Context) error {
			return srv.Shutdown(ctx)
		},
	})

	return srv
}
```

A background worker that must stop the whole app on failure takes `fx.Shutdowner` and calls `Shutdown()`, exactly as above.

## core/logger.go

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

## A feature package

`Provider` is the entire public surface `main()` needs. Dependencies arrive in an `fx.In` struct, so adding one is a one-line change here and nowhere else.

```go
package repository

type RouterIn struct {
	fx.In

	Log   *slog.Logger
	Clock clockwork.Clock
	Store Store
}

type Router struct {
	RouterIn // embed so handlers reach deps as r.Log, r.Store, ...
}

// Provider is the only thing main() needs to know about this package.
func Provider(in RouterIn) (core.RouterOut, error) {
	r := &Router{RouterIn: in}

	return core.RouterOut{
		Router: &core.RouterSpec{
			Version:  1,
			Prefix:   "repositories",
			Register: r.Register,
		},
	}, nil
}

func (r *Router) Register(mux *http.ServeMux, prefix string) {
	mux.HandleFunc("GET "+prefix+"/{id}", r.Get)
}

func (r *Router) Get(w http.ResponseWriter, req *http.Request) {
	id := req.PathValue("id")
	log := r.Log.With("repository_id", id)

	repo, ok := r.Store.Get(id)
	if !ok {
		log.InfoContext(req.Context(), "repository not found")
		http.Error(w, "not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	if err := json.NewEncoder(w).Encode(repo); err != nil {
		log.ErrorContext(req.Context(), "encoding response", "error", err)
	}
}
```

Depend on an interface (`Store`), provide the concrete type. The fake in tests then needs no mocking framework.

## Testing — build the fx.In by hand

Do not start an fx app to test a handler. Construct the input struct, call the provider, assert.

```go
package repository_test

// Build the fx.In struct by hand — no fx.New, no lifecycle, no magic.
func setup(t *testing.T) (*http.ServeMux, *clockwork.FakeClock) {
	t.Helper()

	clock := clockwork.NewFakeClock()

	out, err := repository.Provider(repository.RouterIn{
		Log:   slog.New(slog.NewTextHandler(io.Discard, nil)),
		Clock: clock,
		Store: repository.NewMemoryStore(),
	})
	require.NoError(t, err) // nothing below is meaningful if wiring failed

	mux := http.NewServeMux()
	out.Router.Register(mux, "/v1/repositories")

	return mux, clock
}

func TestGet(t *testing.T) {
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
			mux, _ := setup(t)

			w := httptest.NewRecorder()
			mux.ServeHTTP(w, httptest.NewRequest(http.MethodGet, tt.path, nil))

			assert.Equal(t, tt.wantCode, w.Code)
		})
	}
}
```

`slog.New(slog.NewTextHandler(io.Discard, nil))` keeps test output clean. Swap in `os.Stderr` while debugging a single test.

When the lifecycle itself is what you're testing, use `fxtest`:

```go
app := fxtest.New(t, fx.Provide(...), fx.Invoke(core.NewServer))
app.RequireStart()
defer app.RequireStop()
```

## Debugging the graph

- `fx.New` returning an error from `app.Err()` names the missing or duplicated type — read it, it is usually precise.
- `fx.Populate(&target)` extracts a built value in a test without an `Invoke`.
- Two providers for the same type is an error, not a last-write-wins. Use `fx.Annotate` with `fx.ResultTags` to name them apart, or a group.
- A provider that never runs means nothing in the `Invoke` chain depends on it. That is the lazy behaviour working, not a bug.

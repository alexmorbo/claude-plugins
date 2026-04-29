# Go Anti-Patterns

Recurring mistakes — including the ones Claude makes by default. Each entry: what's wrong, what's right, why.

## Outdated stdlib idioms

Claude is biased toward older Go because of training data frequency. See `${CLAUDE_PLUGIN_ROOT}/skills/go-modernize/SKILL.md` for the full mapping. The most common offenders:

| Wrong | Right | Since |
|-------|-------|-------|
| `interface{}` | `any` | 1.18 |
| `for _, v := range s { if v == x { return true } }` | `slices.Contains(s, x)` | 1.21 |
| `for i := 0; i < n; i++` | `for i := range n` | 1.22 |
| `context.WithCancel(ctx)` in tests | `t.Context()` | 1.24 |
| `omitempty` on `time.Time` | `omitzero` | 1.24 |
| Third-party logging libs (logrus/zap) for new code | `log/slog` | 1.21 |

## Naming

### Generic package names

```
util/      ❌
helpers/   ❌
common/    ❌
tools/     ❌
misc/      ❌
shared/    ❌
```

These tell the reader nothing. They become dumping grounds and circular-import magnets. Name packages by **what they do**:

```
✅ retry/       (provides retry primitives)
✅ pgconn/      (Postgres connection helpers)
✅ httpsig/     (HTTP signature verification)
```

If you can't think of a name better than `util`, the package shouldn't exist — put the function in the package that uses it.

### Single-letter and shorthand names

```go
// Wrong: cryptic
func process(m *M, s string, c context.Context) error {
    for _, x := range m.xs {
        if x.s == s {
            return nil
        }
    }
}

// Right: descriptive
func processOrder(ctx context.Context, order *Order, status string) error {
    for _, item := range order.items {
        if item.status == status {
            return nil
        }
    }
}
```

Exception: short scoped names in tight loops (`i`, `j`, `k`, `err`, `ok`, single-letter receivers like `u *User`) are idiomatic.

### Stuttering

```go
// Wrong: package name repeated
package user
type UserService struct {}        // user.UserService — stutters
func NewUserService() *UserService

// Right
package user
type Service struct {}            // user.Service — clean
func NewService() *Service
```

Same for `errors.ErrNotFound` (use `errors.NotFound`), `http.HTTPClient` (use `http.Client`), etc.

## Error handling

### Naked `return err`

```go
// Wrong: caller has no idea where the error came from
if err := repo.Save(ctx, user); err != nil {
    return err
}

// Right: add context with %w to preserve chain
if err := repo.Save(ctx, user); err != nil {
    return fmt.Errorf("save user %s: %w", user.ID(), err)
}
```

Stack traces are unidiomatic in Go — error chains with `%w` and meaningful messages are the substitute. **Every error site should add context.**

### Logging then returning

```go
// Wrong: error is logged twice (here and at the top of the call stack)
if err := repo.Save(ctx, user); err != nil {
    slog.Error("save failed", "error", err)
    return err
}

// Right: log OR return, not both. Usually return — log at the top.
if err := repo.Save(ctx, user); err != nil {
    return fmt.Errorf("save user: %w", err)
}
```

The Uber rule: **handle each error once.** Either you handle it (log + recover) or you propagate it (return wrapped). Don't do both.

### `==` for sentinel errors

```go
// Wrong: breaks if the error is wrapped
if err == sql.ErrNoRows { ... }

// Right
if errors.Is(err, sql.ErrNoRows) { ... }
```

Same for typed errors: use `errors.As`, not type assertion.

## Pointers

### Pointers everywhere ("Go as C")

```go
// Wrong: pointer for primitives, value semantics destroyed
func Add(a *int, b *int) *int {
    s := *a + *b
    return &s
}
```

Go is not C. Pass values unless:

1. The struct is large (heuristic: >64 bytes — but profile, don't guess).
2. The function mutates the receiver/argument.
3. The type's zero value isn't usable and you need explicit nil.

For small structs and primitives — pass by value. Allocator and GC handle the rest.

### Pointer to slice/map

```go
// Wrong: slice and map headers are already references
func process(items *[]Item) { ... }
func lookup(m *map[string]int) { ... }

// Right
func process(items []Item) { ... }
func lookup(m map[string]int) { ... }
```

Mutations through the slice/map header are visible to the caller already. Pointer adds a deref with no benefit.

### Pointer to mutex

```go
// Wrong: mutex is small, pointer adds heap allocation + nil risk
type Cache struct {
    mu *sync.Mutex
    m  map[string]string
}

// Right: zero value of sync.Mutex is usable
type Cache struct {
    mu sync.Mutex
    m  map[string]string
}
```

Same for `sync.RWMutex`, `sync.WaitGroup`, `bytes.Buffer`. Zero values work — embed by value.

## Structs and interfaces

### Big interfaces at definition sites

```go
// Wrong: producer-defined kitchen-sink interface
package userrepo

type UserRepository interface {
    Save(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id UserID) error
    FindByID(ctx context.Context, id UserID) (*User, error)
    FindByEmail(ctx context.Context, e Email) (*User, error)
    ListAll(ctx context.Context) ([]*User, error)
    // ...
}
```

Better: **interfaces belong to the consumer**, defined where they're used, with only the methods that consumer needs.

```go
// Right: consumer defines what it needs
package usecase

type userFinder interface {
    FindByID(ctx context.Context, id userrepo.UserID) (*userrepo.User, error)
}

type GetUserUseCase struct {
    users userFinder  // not the full repository — just what's used
}
```

Repositories can still expose a struct with all methods on the implementation side.

### Embedded types in public API

```go
// Wrong: embedding leaks all methods of *sql.DB into the public API
type Repo struct {
    *sql.DB
}
```

Now callers can call `repo.Close()`, `repo.Driver()`, etc. Use named fields:

```go
type Repo struct {
    db *sql.DB
}
```

### Constructors that return errors but aren't needed

```go
// Wrong: nothing can fail in this constructor
func NewService() (*Service, error) {
    return &Service{}, nil
}

// Right
func NewService() *Service {
    return &Service{}
}
```

Reserve `(T, error)` constructors for things that can actually fail (parse config, open file, etc.).

## Concurrency

### `time.Sleep` in tests

```go
// Wrong: flaky, slow
go worker.Start(ctx)
time.Sleep(100 * time.Millisecond)  // hope worker started?
assert.True(t, worker.Running())
```

Use synchronization primitives or `testing/synctest` (1.25+). If you must wait, use `assert.Eventually`.

### Goroutines without coordination

```go
// Wrong: leak on cancel, errors swallowed
go doThing(ctx)
go doOtherThing(ctx)
```

Use `errgroup`:

```go
g, gctx := errgroup.WithContext(ctx)
g.Go(func() error { return doThing(gctx) })
g.Go(func() error { return doOtherThing(gctx) })
return g.Wait()
```

### Channel buffer for "performance"

```go
// Wrong: 10000-buffer channel as "optimization"
ch := make(chan Event, 10000)
```

Channel buffer should reflect a deliberate decision: 0 (synchronous handoff), 1 (decouple producer/consumer for one item), or *exactly* the batch size. Large buffers usually hide a design problem.

## Configuration

### Hardcoded values

```go
// Wrong
client := http.Client{Timeout: 30 * time.Second}
```

If a value might need tuning per environment, it's config.

```go
// Right
client := http.Client{Timeout: cfg.HTTPTimeout}
```

### `init()` for config / DB / IO

```go
// Wrong: makes testing painful, hides startup work
func init() {
    db = openDB(os.Getenv("DATABASE_URL"))
}
```

Initialize in `main()` and pass dependencies down. `init()` is for things like registering protobuf types, not for IO.

## Testing

### Mocking everything

Don't mock value objects, domain entities, or stdlib. Mock at boundaries: repositories, external clients, time providers. Inside the domain, use real objects.

### Single huge test

```go
// Wrong
func TestUserService(t *testing.T) {
    // 200 lines covering create, update, delete, list, all in one test
}
```

Split with `t.Run`:

```go
func TestUserService(t *testing.T) {
    t.Run("Create", func(t *testing.T) { ... })
    t.Run("Update", func(t *testing.T) { ... })
    t.Run("Delete", func(t *testing.T) { ... })
}
```

### Asserting through reflection / `interface{}`

```go
// Wrong: fragile, opaque error messages
got := result.(map[string]interface{})["data"].([]interface{})[0].(string)
```

Marshal/unmarshal into a real struct and assert on fields.

## Workflow with Claude

### Multi-task prompts

Claude degrades when asked for many things at once ("refactor this, add tests, update docs, and fix the bug in X"). Break into separate turns.

### Verbose test output masking real failures

Run with `go test -v ./...` only when investigating. Default to `go test ./...` so failures stand out.

### Skipping `.gitignore` / `.claudeignore`

Vendor/, generated code, large binaries, model snapshots — exclude from context. Claude reading 50k-line generated files is wasted budget and worse output.

### "Push and pray"

Quality gates (`go vet`, `staticcheck`, `golangci-lint`, `go test`) **must run** before commit. AI-generated code is fast but inconsistent. The `/go-review` slash command in this plugin runs them in one go.

---

## Sources

- Three Dots Labs blog (Go anti-patterns): https://threedots.tech/post/list-of-recommendations-for-idiomatic-go/
- Uber Go Style Guide: https://github.com/uber-go/guide/blob/master/style.md
- Go Code Review Comments: https://go.dev/wiki/CodeReviewComments
- JetBrains "Write Modern Go Code with Junie and Claude": https://blog.jetbrains.com/go/2026/02/20/write-modern-go-code-with-junie-and-claude-code/

---
name: go-modernize
description: "Use modern Go idioms appropriate for the project's Go version (read from go.mod). Activates when generating new Go code, refactoring existing code, or reviewing Go for outdated patterns. Counters Claude's bias toward older idioms (interface{}, manual loops, manual cancellation) by mapping to current stdlib equivalents."
---

# Go Modernization Skill

Claude's training data is biased toward older Go (often pre-1.21). This skill enforces modern stdlib idioms appropriate for the project's actual Go version.

## Step 1 — Read the version

**Always check `go.mod` before generating code.** Look for the `go X.Y` line. If the version is older than the recommendation here, do **not** use a feature it doesn't support.

```bash
head -5 go.mod | grep '^go '
```

## Step 2 — Apply version-appropriate idioms

### Always (Go 1.18+)

| Outdated | Modern | Why |
|----------|--------|-----|
| `interface{}` | `any` | Built-in alias since 1.18; clearer intent |
| Untyped literals where generics fit | Generic functions | Type safety, less duplication |

### Go 1.21+ (released Aug 2023)

Use the new stdlib packages instead of writing equivalents.

| Don't | Do | Notes |
|-------|----|----|
| Manual `for _, v := range s { if v == x { return true } }` | `slices.Contains(s, x)` | `slices` package |
| Manual `for i, v := range s { if v == x { return i } }` | `slices.Index(s, x)` | |
| `sort.Slice(s, func(i,j int) bool { return s[i] < s[j] })` | `slices.Sort(s)` | Faster, simpler |
| Manual map key extraction | `maps.Keys(m)` / `maps.Values(m)` (returns iter in 1.23+) | |
| `if a > b { return a }; return b` | `max(a, b)` (or `min`) | Built-in |
| Nested ternary fallbacks | `cmp.Or(a, b, c)` | First non-zero |
| `logrus`, `zap`, `zerolog` for new code | `log/slog` | Stdlib structured logging |
| `for k := range m { delete(m, k) }` | `clear(m)` | Built-in; also clears slices |
| Custom min/max helpers | `min(a,b)` / `max(a,b)` built-ins | |

### Go 1.22+ (released Feb 2024)

| Don't | Do |
|-------|----|
| `for i := 0; i < n; i++ { ... }` | `for i := range n { ... }` |
| `math/rand` for new code | `math/rand/v2` (better API, faster) |
| Worry about loop variable capture in goroutines | Per-iteration scoping is automatic |

### Go 1.23+ (released Aug 2024)

| Don't | Do |
|-------|----|
| Custom iterator types | `iter.Seq[T]` / `iter.Seq2[K,V]` and range-over-func |
| Hand-roll deduplication for canonicalization | `unique.Make(value)` |

### Go 1.24+ (released Feb 2025)

| Don't | Do |
|-------|----|
| `ctx, cancel := context.WithCancel(...)` in tests | `t.Context()` (auto-cancelled at test end) |
| `omitempty` on `time.Time` (broken — never empty) | `omitzero` JSON tag |
| `for i := 0; i < b.N; i++` in benchmarks | `for b.Loop()` |
| Manual finalizers via `runtime.SetFinalizer` | `weak.Pointer[T]` / `runtime.AddCleanup` |
| Type alias without generics | Generic type aliases |

### Go 1.25+ (released Aug 2025)

| Don't | Do |
|-------|----|
| Real-time `time.Sleep` in time-dependent tests | `testing/synctest` (synthetic clock) |
| Read-modify-write on `sync.Map` with race window | `sync.Map.Compute` / `CompareAndSwap` |
| Custom JSON encoder for streaming | `encoding/json/v2` (experimental in 1.25, opt-in) |

## Step 3 — Don't invent features

If you're not sure a feature exists in the project's Go version:

1. **Check the official release notes**: https://go.dev/doc/devel/release
2. **Run `go doc <pkg>.<symbol>`** to verify.
3. When in doubt, use the older idiom that definitely works.

The table above is conservative. New Go versions may add more — re-read this skill against the latest Go release notes when you bump the project.

## Step 4 — When upgrading an existing codebase

If `go.mod` was just bumped:

1. Run `gopls codeaction` or `golangci-lint run --enable-only=usestdlibvars,usesvars,modernize` (when available) to flag candidates.
2. Apply changes one category at a time (e.g., all `interface{}` → `any` in one commit).
3. Re-run tests after each batch.

## Anti-recommendations

Don't:

- Use `os.ReadFile` / `os.WriteFile` for new code? **No, use them.** They've been stable since 1.16 — don't reach back to `ioutil`.
- Use third-party `errors` packages (e.g., `pkg/errors`) when stdlib `fmt.Errorf("...: %w", err)` + `errors.Is` / `errors.As` covers it.
- Use third-party logging libs in new code (slog covers structured logging well — only justify alternatives by concrete need).
- Pull in `golang.org/x/exp/slices` — promoted to `slices` in stdlib since 1.21.

## Quick verification

When you finish writing code:

```bash
# Find places where modernization is still needed:
grep -rn 'interface{}' --include='*.go' .            # → any
grep -rn 'for.*; i++' --include='*.go' .             # → for range n (1.22+)
grep -rn 'github.com/sirupsen/logrus\|go.uber.org/zap' .  # → log/slog
grep -rn 'context.WithCancel' --include='*_test.go' . # → t.Context() (1.24+)
```

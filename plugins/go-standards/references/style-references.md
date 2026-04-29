# Go Style References

External authoritative style guides. Our plugin's `code-style.md` is opinionated and project-specific; these fill in everything else and resolve borderline cases.

## The four canonical sources

| Source | URL | Best for |
|--------|-----|----------|
| Effective Go | https://go.dev/doc/effective_go | Foundational idioms — required reading |
| Go Code Review Comments | https://go.dev/wiki/CodeReviewComments | Short canonical checklist used by reviewers |
| Google Go Style Guide | https://google.github.io/styleguide/go/ | Three-tier (Style Guide / Style Decisions / Best Practices). Reasoning, not just rules |
| Uber Go Style Guide | https://github.com/uber-go/guide/blob/master/style.md | The most prescriptive; testable in CI |

## What each adds beyond our `code-style.md`

### Effective Go

Foundational. Read once, refer back when stuck. Adds:
- Idiomatic control flow (`if x, err := ...; err != nil`)
- Goroutine/channel idioms beyond `errgroup`
- `init()` semantics
- Embedding vs. composition tradeoffs
- Slice mechanics (capacity, sharing)

### Google Go Style Guide

Three layers, each more strict:

1. **Style Guide** — minimum bar (formatting, naming).
2. **Style Decisions** — choices Google made and why. Use as a tiebreaker in code review disagreements.
3. **Best Practices** — patterns to reach for over time.

Particularly useful for:
- Doc comment style (godoc rendering)
- Test error message format ("got X, want Y")
- Error string format (lowercase, no trailing punctuation)

### Uber Go Style Guide

Most prescriptive. Many rules map directly to lints. Adds rules our docs don't enforce:

| Rule | Why |
|------|-----|
| Zero-value `sync.Mutex` (no `*sync.Mutex`) | Smaller, no nil risk |
| Copy slices/maps at boundaries | Prevent aliasing bugs |
| Channel size ∈ {0, 1} | Larger sizes hide design issues |
| No embedded types in public structs | Leaks methods, breaks compatibility |
| Enums start at 1 (not 0) | Distinguish "unset" from "first value" |
| All marshaled structs must have tags | Field renames silently change wire format otherwise |
| No `init()` for IO/config | Hard to test, hides startup work |
| `time.Duration` for periods, `time.Time` for instants | Don't pass durations as `int64` |
| Handle errors once | Log OR return, never both |
| Prefix verbose error variables with `Err` | `ErrNotFound`, not `NotFound` |
| Receiver names: 1-2 chars, consistent | `u *User` not `user *User` everywhere |

### Go Code Review Comments

The original short checklist that golangci-lint rules came from. Read in 15 minutes; covers:
- Error string format
- Receiver names and types
- Empty slices via `var s []T`, not `[]T{}`
- Mixed caps for naming
- Comment punctuation

## Overlap with our docs

Our `code-style.md` and `error-handling.md` already cover:
- Wrap errors with `%w` and context
- Use `errors.Is` / `errors.As`
- Receiver names short and consistent
- Clean Architecture layer boundaries
- Test naming convention
- Minimal comments policy

So **don't re-derive** these from the external guides. Where we conflict with an external guide, our docs win (we're more opinionated about CA/DDD and stack choice).

## When to consult which

| Question | Source |
|----------|--------|
| "Is this idiomatic Go?" | Effective Go |
| "Code review pushback says X — is X correct?" | Code Review Comments |
| "Two valid options — which is preferred?" | Google Style Decisions |
| "Is this rule worth enforcing in CI?" | Uber Style Guide |
| "How should this CA layer be structured?" | `${CLAUDE_PLUGIN_ROOT}/references/clean-architecture.md` (ours) |
| "What modern stdlib should I use?" | `${CLAUDE_PLUGIN_ROOT}/skills/go-modernize/SKILL.md` (ours) |

## golangci-lint as the enforcement layer

Most of these style rules are encoded in `golangci-lint`. Our `linting.md` describes our recommended config. Cross-reference these linters with the guides above:

| Linter | Maps to |
|--------|---------|
| `errcheck` | Code Review Comments (error handling) |
| `revive` | Code Review Comments (broad) |
| `staticcheck` | All four (broad) |
| `gocritic` | Uber Style Guide (many) |
| `gosec` | Security best practices |
| `errorlint` | Uber "errors.Is/As" rule |
| `predeclared` | Don't shadow stdlib names |
| `stylecheck` | Google + Code Review Comments |

If a rule from the guides isn't enforced by your `.golangci.yml`, decide: enforce it (add the linter) or accept the risk (document why).

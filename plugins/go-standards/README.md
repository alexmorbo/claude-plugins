# go-standards

Claude Code plugin with Go microservices development standards: Clean Architecture, DDD, testing, linting, error handling, project structure.

Originally extracted from a homelab monorepo to make these standards reusable across projects and machines.

## Install

```
/plugin marketplace add git@github.com:alexmorbo/claude-plugins.git
/plugin install go-standards@alexmorbo-plugins
```

See the [marketplace README](../../README.md) for HTTPS install / update / uninstall.

## What's inside

### Skills (auto-activated)

Claude loads these automatically when the conversation matches their description.

| Skill | Activates when |
|-------|----------------|
| `go-clean-architecture` | Implementing features, creating components, modifying Go code in domain/application/infrastructure/interface layers |
| `go-code-planning` | Creating story files, planning features, writing technical specifications |
| `go-error-handling` | Creating error types, handling errors, wrapping errors, error boundaries |
| `go-testing` | Writing tests, reviewing coverage, creating mocks |
| `go-linting` | After Go code changes, before completing implementation |
| `go-modernize` | Generating new Go code or refactoring — counters Claude's bias toward older idioms (`interface{}`, manual loops, etc.) by mapping to current stdlib (`any`, `slices.Contains`, `t.Context()`, `slog`, …) based on `go.mod` version |
| `go-libraries-first` | Before writing any helper/util/wrapper — checks a curated catalog ("for X use library Y") to prevent reinventing retry, backoff, validation, MQTT, Modbus, etc. |

Each skill points at deeper reference docs in `references/` via `${CLAUDE_PLUGIN_ROOT}` so the agent can read them on demand.

### Slash commands

| Command | Purpose |
|---------|---------|
| `/ca-init-go` | Scaffold a new Go service with Clean Architecture structure |
| `/ca-validate-go` | Validate an existing Go service for CA compliance |
| `/go-review` | Diff-aware review: `go vet` + `staticcheck` + `golangci-lint` + `gosec` + `govulncheck` + tests + build, with CRITICAL/HIGH/MEDIUM severity report |
| `/go-gen-test` | Generate a table-driven test for a function/method, layered to its CA layer (domain/application/infrastructure/interface), with edge cases and modern stdlib idioms |
| `/go-audit` | Pre-release safety audit: vulnerabilities, security findings, license compliance, outdated deps, module hygiene |

### Hooks

| Hook | Trigger | Action |
|------|---------|--------|
| `PostToolUse` (Write/Edit) | After a `.go` file is written | Auto-format with `goimports -w` (falls back to `gofmt -w` if `goimports` not installed) |
| `PreToolUse` (Write/Edit) | Before any file write | Block edits to generated files: `*.pb.go`, `*_gen.go`, `*_mock.go`, `mocks/`, `generated/` |

The post-hook reads `git diff` to find changed `.go` files in `$CLAUDE_PROJECT_DIR`. Install `goimports` for the better behavior:

```bash
go install golang.org/x/tools/cmd/goimports@latest
```

### Reference documentation

Full docs in `references/`:

| Doc | Topic |
|-----|-------|
| `clean-architecture.md` | 4 layers, dependency rule |
| `ddd.md` | Entities, Value Objects, Aggregates |
| `project-structure.md` | Standard directory layout |
| `code-style.md` | Naming, comment policy |
| `libraries.md` | Approved libraries (Gin, GORM, go-redis, etc.) |
| `error-handling.md` | Custom errors, wrapping, HTTP mapping |
| `testing.md` | Coverage targets, mocking |
| `linting.md` | golangci-lint config, common fixes |
| `configuration.md` | Env vars, secrets |
| `api-design.md` | REST conventions, versioning |
| `security.md` | Validation, auth |
| `ci-cd.md` | GitLab CI patterns |
| `observability.md` | VictoriaMetrics/Logs |
| `logging-format.md` | Structured slog |
| `agent-workflow/` | Multi-agent (Opus → Haiku) story-driven workflow |
| `anti-patterns.md` | Go anti-patterns and Claude's common mistakes (outdated idioms, generic package names, naked `return err`, pointer overuse, etc.) |
| `style-references.md` | Pointers to Effective Go, Google, Uber, Code Review Comments + table of what each adds beyond our docs |
| `mcp-servers.md` | Recommended MCP servers for Go work (`gopls mcp`, `hloiseau/mcp-gopls`) with `.mcp.json` snippets |

These don't load automatically — skills reference them by path so the agent reads them when needed.

## Recommended project CLAUDE.md snippet

After installing the plugin, paste this into your project's `CLAUDE.md` so future sessions know which patterns apply (the plugin can't auto-inject CLAUDE.md content):

```markdown
## Go Services

This project uses the [`go-standards` plugin](https://github.com/alexmorbo/claude-plugins) for Go conventions.

**Auto-activated skills:** `go-clean-architecture`, `go-code-planning`, `go-error-handling`, `go-testing`, `go-linting`, `go-modernize`, `go-libraries-first`.

**Slash commands:** `/ca-init-go`, `/ca-validate-go`, `/go-review`, `/go-gen-test`, `/go-audit`.

**Hooks:** auto-`goimports` (or `gofmt` fallback) on `.go` writes; block edits to generated files (`.pb.go`, `_gen.go`, `_mock.go`, `mocks/`, `generated/`).

**Key principles:**
- Clean Architecture: Domain → Application → Infrastructure → Interface
- Minimal comments, self-documenting code
- Standard stack: Gin, GORM, go-redis, log/slog, VictoriaMetrics/metrics, testify
- Coverage: 95% domain, 80% overall
```

## Project-specific notes (from origin)

A few references in `references/` mention things that are specific to the original homelab project and may not apply elsewhere:

- `linting.md` / `golangci-lint` config: `local-prefixes: gitlab.morbo.dev` — change to your own module prefix.
- `ci-cd.md`: GitLab CI with `homelab-ci` shared templates — internal repo, use as a structural reference only.
- `libraries.md`: example reference implementation links to `gitlab.morbo.dev/homelab/grona-lund-bot` (private).

These don't break anything — Claude reads the docs as guidance, not as code to execute.

## Versioning

Semantic-ish. Bump `version` in both `.claude-plugin/plugin.json` (this plugin) and the corresponding entry in `../../.claude-plugin/marketplace.json` when changing skills/commands/hooks.

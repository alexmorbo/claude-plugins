# Go Services Development Standards

This documentation defines standards for developing Go microservices in the homelab ecosystem.

## Quick Links

### Architecture & Design

| Document | Description |
|----------|-------------|
| [Clean Architecture](clean-architecture.md) | 4 layers, dependency rule, layer responsibilities |
| [DDD Patterns](ddd.md) | Entities, Value Objects, Aggregates, Repositories |
| [Project Structure](project-structure.md) | Standard directory layout and file templates |
| [API Design](api-design.md) | REST API, versioning, request/response formats |

### Development

| Document | Description |
|----------|-------------|
| [Code Style](code-style.md) | Minimal comments, naming conventions |
| [Linting](linting.md) | golangci-lint, quality checks, coverage verification |
| [Libraries](libraries.md) | Approved libraries (Gin, GORM, go-redis, etc.) |
| [Error Handling](error-handling.md) | Custom errors, wrapping, HTTP mapping |
| [Configuration](configuration.md) | Environment variables, secrets |
| [Testing](testing.md) | Hybrid approach, coverage targets, mocking |

### Operations

| Document | Description |
|----------|-------------|
| [CI/CD](ci-cd.md) | GitLab CI with homelab-ci |
| [Observability](observability.md) | VictoriaMetrics, VictoriaLogs stack |
| [Logging](logging-format.md) | Structured JSON logging with slog |
| [Security](security.md) | Validation, authentication, best practices |

## Agent Workflow

For AI-assisted development workflow, see [agent-workflow/](agent-workflow/):

| Document | Purpose |
|----------|---------|
| [Overview](agent-workflow/README.md) | Multi-agent workflow explanation |
| [Story Template](agent-workflow/story-template.md) | Task documentation format (Markdown + YAML) |
| [Planning Guide](agent-workflow/planning-guide.md) | For planning agents (Opus / Sonnet) |
| [Implementation Guide](agent-workflow/implementation-guide.md) | For implementation agents (Haiku) |

## Claude Code Skills & Hooks

Auto-activated capabilities in `.claude/`:

### Skills (Auto-Discovery)

Claude automatically loads these when working with Go code:

| Skill | Activates When |
|-------|----------------|
| `go-clean-architecture` | Implementing features, creating components |
| `go-error-handling` | Creating/handling errors, error wrapping |
| `go-testing` | Writing tests, checking coverage |
| `go-linting` | After code changes, before completion |

### Hooks (Automatic Actions)

| Hook | Trigger | Action |
|------|---------|--------|
| PostToolUse (Write/Edit) | After `.go` file changes | Auto-format with `gofmt` |
| PreToolUse (Write/Edit) | Before modifying files | Block generated files (`.pb.go`, `_gen.go`, `mocks/`) |

## Key Principles

1. **Clean Architecture** - Domain → Application → Infrastructure → Interface
2. **Minimal Comments** - Code should be self-documenting
3. **Structured Logging** - JSON format with 5 standard groups
4. **Standard Libraries** - Gin, GORM, go-redis, VictoriaMetrics/metrics, slog
5. **Test Coverage** - 95% domain, 80% overall

## Standard Stack

| Component | Library |
|-----------|---------|
| HTTP Server | `github.com/gin-gonic/gin` |
| PostgreSQL | `gorm.io/gorm` |
| Redis/Valkey | `github.com/redis/go-redis/v9` |
| S3 Storage | `github.com/aws/aws-sdk-go-v2` |
| Telegram | `github.com/mymmrac/telego` |
| Logging | `log/slog` (stdlib) |
| Metrics | `github.com/VictoriaMetrics/metrics` |
| Testing | `github.com/stretchr/testify` |

## Reference Implementation

See [grona-lund-bot](https://gitlab.morbo.dev/homelab/grona-lund-bot) as the canonical example.

---
name: go-libraries-first
description: "Before writing any helper, util, wrapper, or 'glue' code in Go, check if a maintained library already solves it. Activates when about to implement: HTTP retry, backoff, config loading, validation, CLI parsing, scheduling, MQTT, Modbus, graceful shutdown, structured logging, metrics, ID generation, error wrapping, or any 'small thing'. Strongly prefer the curated library catalog below over hand-rolled code."
---

# Go Libraries-First Skill

**Rule: don't write a util when a library exists.** Hand-rolled "small helpers" accrete into untested, undocumented project-specific dialects. This skill provides a curated catalog so Claude reaches for the same answer every time.

## When this skill triggers

Before writing any of these, check the catalog:

- A retry loop, backoff, exponential delay
- Config loading from env / file / flags
- Struct field validation (email, URL, length, regex)
- CLI flags / subcommands
- Cron-like scheduling
- Graceful shutdown coordination
- Structured logging
- Metrics/Prometheus exposition
- UUID/ULID/snowflake generation
- HTTP middleware (rate limit, CORS, auth)
- Protocol clients (MQTT, Modbus, gRPC, AMQP, NATS)
- Test containers for integration tests

If the task matches anything below — **use the recommended library**. Don't write your own.

---

## Catalog

### HTTP server

| Need | Library | Notes |
|------|---------|-------|
| HTTP framework | `github.com/gin-gonic/gin` | Project default; chosen for middleware ecosystem and binding |
| Lightweight router (alternative) | `github.com/go-chi/chi/v5` | Stdlib-style; choose if not using Gin |
| Middleware: CORS | `github.com/gin-contrib/cors` | |
| Middleware: rate limit | `github.com/ulule/limiter/v3` | Backed by Redis or in-memory |
| Health/readiness probes | `github.com/heptiolabs/healthcheck` | Composable checks |

### HTTP client

| Need | Library | Notes |
|------|---------|-------|
| Standard requests | `net/http` (stdlib) | Always wrap with `context.Context` |
| Retry with backoff | `github.com/hashicorp/go-retryablehttp` | Drop-in replacement for `http.Client` |
| Rich client (chainable) | `github.com/go-resty/resty/v2` | Use when verbose request building gets in the way |

### Configuration

| Need | Library | Notes |
|------|---------|-------|
| Env vars via struct tags | `github.com/caarlos0/env/v11` | Simplest; default for small services |
| Multi-source (file + env + flags) | `github.com/spf13/viper` | Use only when actually needed |
| Config validation | `github.com/go-playground/validator/v10` | Pair with caarlos0/env |
| Secrets from SOPS/Vault | Read at boot, parse with above | Don't write your own decryption |

### CLI

| Need | Library | Notes |
|------|---------|-------|
| Subcommands + flags | `github.com/spf13/cobra` | De-facto standard (kubectl, hugo, gh) |
| Single-binary flags only | `flag` (stdlib) | When cobra is overkill |
| Interactive prompts | `github.com/charmbracelet/huh` | Modern survey replacement |

### Logging

| Need | Library | Notes |
|------|---------|-------|
| Structured JSON logging | `log/slog` (stdlib, 1.21+) | **Default. Don't reach for logrus/zap/zerolog without a concrete reason.** |
| slog handler with file rotation | `gopkg.in/natefinch/lumberjack.v2` | Wrap a slog handler |
| Pretty console handler (dev) | `github.com/lmittmann/tint` | slog handler with colors |

### Metrics & observability

| Need | Library | Notes |
|------|---------|-------|
| App metrics | `github.com/VictoriaMetrics/metrics` | Project default; lighter than prometheus client |
| Prometheus-flavored | `github.com/prometheus/client_golang` | Use only if Prometheus exposition required |
| Distributed tracing | `go.opentelemetry.io/otel` | Standard tracing API |
| OpenTelemetry exporters | `go.opentelemetry.io/otel/exporters/otlp/otlptrace` | |

### Errors

| Need | Library | Notes |
|------|---------|-------|
| Wrap with context | `fmt.Errorf("...: %w", err)` (stdlib) | **Default. No third-party errors lib unless stack traces needed.** |
| Stack traces | `github.com/cockroachdb/errors` | Justify before pulling in; conflicts with stdlib idioms |
| Error groups (concurrent) | `golang.org/x/sync/errgroup` | First error cancels siblings |

### Concurrency

| Need | Library | Notes |
|------|---------|-------|
| Concurrent operations with cancel-on-error | `golang.org/x/sync/errgroup` | |
| Concurrency limiter | `golang.org/x/sync/semaphore` | |
| De-duplicate inflight calls | `golang.org/x/sync/singleflight` | Cache stampede protection |
| Object pool with cleanup | `sync.Pool` (stdlib) | |
| Synthetic clock for tests | `testing/synctest` (1.25+) | Don't use `time.Sleep` in tests |

### Retry & resilience

| Need | Library | Notes |
|------|---------|-------|
| Generic retry with backoff | `github.com/cenkalti/backoff/v4` | Constant, exponential, custom strategies |
| Circuit breaker | `github.com/sony/gobreaker` | Maintained, simple API |
| Rate limiter (token bucket) | `golang.org/x/time/rate` | Stdlib-adjacent, no extra deps |

### Database (SQL)

| Need | Library | Notes |
|------|---------|-------|
| ORM | `gorm.io/gorm` | Project default |
| Lower-level SQL | `github.com/jmoiron/sqlx` | Alternative when GORM is overkill |
| Migrations | `github.com/golang-migrate/migrate/v4` | Standard; supports many drivers |
| Postgres driver | `github.com/jackc/pgx/v5` | Native; faster than lib/pq |
| Connection pooling (pgx) | Built-in via pgxpool | Don't add another pool |

### Database (Redis)

| Need | Library | Notes |
|------|---------|-------|
| Redis client | `github.com/redis/go-redis/v9` | Project default |
| Distributed lock | `github.com/bsm/redislock` | If you need locking, don't roll it |

### Validation & ID generation

| Need | Library | Notes |
|------|---------|-------|
| Struct field validation | `github.com/go-playground/validator/v10` | Wide tag vocabulary |
| UUID v4/v7 | `github.com/google/uuid` | |
| ULID (sortable) | `github.com/oklog/ulid/v2` | When ID order matters |
| Snowflake-style | `github.com/bwmarrin/snowflake` | Distributed ID with timestamp |

### Time / scheduling

| Need | Library | Notes |
|------|---------|-------|
| In-process cron | `github.com/robfig/cron/v3` | Maintained |
| Job scheduler with persistence | `github.com/go-co-op/gocron/v2` | Alternative; richer API |
| Time parsing/formatting | `time` (stdlib) | Don't pull `now`-style helpers |

### Graceful shutdown

| Need | Library | Notes |
|------|---------|-------|
| Coordinate goroutines | `golang.org/x/sync/errgroup` + `signal.NotifyContext` | The canonical pattern |
| Process supervision | `github.com/oklog/run` | Multiple actor groups |

### Protocol clients

| Need | Library | Notes |
|------|---------|-------|
| MQTT | `github.com/eclipse/paho.mqtt.golang` | Client v3.1.1 / v5 (separate pkg) |
| MQTT v5 | `github.com/eclipse/paho.golang` | |
| Modbus (TCP/RTU) | `github.com/grid-x/modbus` | Active maintenance |
| gRPC | `google.golang.org/grpc` + `google.golang.org/protobuf` | Generate with `buf generate` |
| AMQP / RabbitMQ | `github.com/rabbitmq/amqp091-go` | Official |
| NATS | `github.com/nats-io/nats.go` | |
| Kafka | `github.com/twmb/franz-go` | Performant; alternative `segmentio/kafka-go` |
| Telegram bot | `github.com/mymmrac/telego` | Project default |
| AWS SDK | `github.com/aws/aws-sdk-go-v2` | Always v2, never v1 |
| S3-compatible | aws-sdk-go-v2 (s3 module) | Same SDK, just a different endpoint |

### Testing

| Need | Library | Notes |
|------|---------|-------|
| Assertions | `github.com/stretchr/testify/assert` (continue) or `require` (stop on fail) | Project default |
| Mocks | `github.com/stretchr/testify/mock` | Pair with `vektra/mockery` v3 to generate |
| Mock generation | `github.com/vektra/mockery/v3` | Or `matryer/moq` for simpler interfaces |
| Integration containers | `github.com/testcontainers/testcontainers-go` | Postgres, Redis, etc. for real-DB tests |
| HTTP fakes | `net/http/httptest` (stdlib) | |
| Snapshot tests | `github.com/bradleyjkemp/cupaloy/v2` | When asserting large outputs |
| Table-driven helpers | Just use standard `t.Run(name, func)` | No library needed |

### Templating / serialization

| Need | Library | Notes |
|------|---------|-------|
| JSON | `encoding/json` (stdlib) | `encoding/json/v2` opt-in 1.25+ for streaming/perf |
| YAML | `gopkg.in/yaml.v3` | Or `github.com/goccy/go-yaml` for stricter |
| TOML | `github.com/BurntSushi/toml` | |
| Templates | `text/template` / `html/template` (stdlib) | |
| Protobuf | `google.golang.org/protobuf` | |

---

## How to apply

When you're about to write code that does any of the above:

1. **Check the catalog.** Find the matching row.
2. **Add the import** instead of writing the helper.
3. **If the library doesn't fit** the use case, document *why* in a comment before writing custom code (so the next reader knows it was a deliberate choice, not ignorance of the library).
4. **If you found a category that's not in the catalog**, the catalog is wrong — flag it to the user so it can be updated.

## What this skill is NOT

- Not a license to add 30 dependencies to a 200-LOC service. Use only what you actually need.
- Not a freeze. If a library in the catalog goes unmaintained, swap it and update this file.
- Not a replacement for understanding what the library does. Read the README before importing.

## Verification

```bash
# Find suspicious "I rewrote a library" code:
grep -rn 'func.*Retry\|func.*Backoff\|func.*Validate' --include='*.go' . | grep -v '_test.go\|vendor/'
# If you see hand-rolled retry/backoff/validation in non-test code, it likely belongs to a library above.
```

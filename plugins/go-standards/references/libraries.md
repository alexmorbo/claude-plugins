# Standard Go Libraries

This document defines the approved libraries for common tasks in Go services.

## Core Libraries

### HTTP Server

**Library:** `github.com/gin-gonic/gin`

Gin web framework for building HTTP APIs with built-in middleware, validation, and binding.

```go
import (
    "github.com/gin-gonic/gin"
)

func NewRouter(
    userHandler *handler.UserHandler,
    healthHandler *handler.HealthHandler,
) *gin.Engine {
    // Production mode
    gin.SetMode(gin.ReleaseMode)

    r := gin.New()

    // Global middleware
    r.Use(gin.Recovery())
    r.Use(middleware.RequestID())
    r.Use(middleware.Logging(logger))
    r.Use(middleware.Metrics())

    // Health endpoints
    r.GET("/health/live", healthHandler.Live)
    r.GET("/health/ready", healthHandler.Ready)

    // API routes
    api := r.Group("/api/v1")
    {
        api.POST("/users", userHandler.Create)
        api.GET("/users/:id", userHandler.GetByID)
        api.PUT("/users/:id", userHandler.Update)
        api.DELETE("/users/:id", userHandler.Delete)
    }

    return r
}

// Handler example with binding
func (h *UserHandler) Create(c *gin.Context) {
    var input dto.CreateUserInput
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    output, err := h.createUser.Execute(c.Request.Context(), input)
    if err != nil {
        h.handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, output)
}

// Server setup
func main() {
    router := NewRouter(userHandler, healthHandler)

    server := &http.Server{
        Addr:         ":8080",
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
    }

    server.ListenAndServe()
}
```

**Key features:**
- Built-in JSON binding and validation
- Path parameters via `c.Param("id")`
- Query parameters via `c.Query("key")`
- Middleware chain with `Use()`
- Route groups for API versioning
- Automatic panic recovery

### PostgreSQL

**Library:** `gorm.io/gorm` + `gorm.io/driver/postgres`

GORM as ORM with PostgreSQL driver.

```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func NewPostgresDB(cfg DatabaseConfig) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(cfg.DSN()), &gorm.Config{
        Logger: NewGormLogger(logger),
    })
    if err != nil {
        return nil, err
    }

    sqlDB, _ := db.DB()
    sqlDB.SetMaxOpenConns(25)
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)

    return db, nil
}
```

### Redis / Valkey

**Library:** `github.com/redis/go-redis/v9`

```go
import "github.com/redis/go-redis/v9"

func NewRedisClient(cfg RedisConfig) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:     cfg.Addr,
        Password: cfg.Password,
        DB:       cfg.DB,
    })
}

// Usage
func (c *SessionCache) Set(ctx context.Context, key string, value any, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    return c.client.Set(ctx, key, data, ttl).Err()
}
```

### S3 Storage

**Library:** `github.com/aws/aws-sdk-go-v2`

Compatible with MinIO and other S3-compatible storages.

```go
import (
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

func NewS3Client(cfg S3Config) (*s3.Client, error) {
    awsCfg, err := config.LoadDefaultConfig(context.Background(),
        config.WithRegion(cfg.Region),
        config.WithCredentialsProvider(credentials.NewStaticCredentialsProvider(
            cfg.AccessKey,
            cfg.SecretKey,
            "",
        )),
    )
    if err != nil {
        return nil, err
    }

    client := s3.NewFromConfig(awsCfg, func(o *s3.Options) {
        if cfg.Endpoint != "" {
            o.BaseEndpoint = &cfg.Endpoint
            o.UsePathStyle = true // Required for MinIO
        }
    })

    return client, nil
}
```

### Telegram Bot

**Library:** `github.com/mymmrac/telego`

```go
import (
    "github.com/mymmrac/telego"
    th "github.com/mymmrac/telego/telegohandler"
)

func NewTelegramBot(token string) (*telego.Bot, error) {
    bot, err := telego.NewBot(token)
    if err != nil {
        return nil, err
    }
    return bot, nil
}

// Webhook handler
func (h *WebhookHandler) HandleUpdate(w http.ResponseWriter, r *http.Request) {
    update, err := h.bot.UpdateFromRequest(r)
    if err != nil {
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }

    // Process update
    h.processUpdate(r.Context(), update)

    w.WriteHeader(http.StatusOK)
}
```

### Logging

**Library:** `log/slog` (stdlib)

See [logging-format.md](logging-format.md) for detailed format specification.

```go
import "log/slog"

func NewLogger(cfg LogConfig) *slog.Logger {
    opts := &slog.HandlerOptions{
        Level: parseLevel(cfg.Level),
    }

    var handler slog.Handler
    if cfg.Format == "json" {
        handler = slog.NewJSONHandler(cfg.Output, opts)
    } else {
        handler = slog.NewTextHandler(cfg.Output, opts)
    }

    return slog.New(handler)
}
```

### Metrics

**Library:** `github.com/VictoriaMetrics/metrics`

VictoriaMetrics native client for Prometheus-compatible metrics.

```go
import "github.com/VictoriaMetrics/metrics"

var (
    httpRequestsTotal = metrics.NewCounter(`http_requests_total{handler="api"}`)
    httpRequestDuration = metrics.NewHistogram(`http_request_duration_seconds{handler="api"}`)
)

func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        httpRequestsTotal.Inc()

        next.ServeHTTP(w, r)

        httpRequestDuration.UpdateDuration(start)
    })
}

// Expose metrics endpoint
func MetricsHandler(w http.ResponseWriter, r *http.Request) {
    metrics.WritePrometheus(w, true)
}
```

### Testing

**Library:** `github.com/stretchr/testify`

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/mock"
)

func TestCreateUser(t *testing.T) {
    // Assertions
    assert.Equal(t, expected, actual)
    assert.NoError(t, err)

    // Fatal assertions
    require.NotNil(t, user)

    // Mocking
    mockRepo := new(MockUserRepository)
    mockRepo.On("Save", mock.Anything, mock.Anything).Return(nil)
}
```

### Integration Testing

**Library:** `github.com/testcontainers/testcontainers-go`

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func setupPostgresContainer(t *testing.T) *postgres.PostgresContainer {
    ctx := context.Background()

    container, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    require.NoError(t, err)

    t.Cleanup(func() {
        container.Terminate(ctx)
    })

    return container
}
```

### UUID

**Library:** `github.com/google/uuid`

```go
import "github.com/google/uuid"

func NewUserID() UserID {
    return UserID{value: uuid.New().String()}
}
```

### Database Migrations

**Library:** `github.com/golang-migrate/migrate/v4`

```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(databaseURL, migrationsPath string) error {
    m, err := migrate.New(
        "file://"+migrationsPath,
        databaseURL,
    )
    if err != nil {
        return err
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }
    return nil
}
```

## Summary Table

| Purpose | Library | Import Path |
|---------|---------|-------------|
| HTTP Server | Gin | `github.com/gin-gonic/gin` |
| PostgreSQL | GORM | `gorm.io/gorm`, `gorm.io/driver/postgres` |
| Redis/Valkey | go-redis | `github.com/redis/go-redis/v9` |
| S3 Storage | AWS SDK v2 | `github.com/aws/aws-sdk-go-v2` |
| Telegram Bot | Telego | `github.com/mymmrac/telego` |
| Logging | stdlib | `log/slog` |
| Metrics | VictoriaMetrics | `github.com/VictoriaMetrics/metrics` |
| Testing | Testify | `github.com/stretchr/testify` |
| Integration Tests | Testcontainers | `github.com/testcontainers/testcontainers-go` |
| UUID | Google UUID | `github.com/google/uuid` |
| Migrations | golang-migrate | `github.com/golang-migrate/migrate/v4` |

## Prohibited Libraries

Do not use these libraries in new services:

| Library | Reason | Alternative |
|---------|--------|-------------|
| `github.com/gorilla/mux` | Deprecated, unnecessary | Gin |
| `github.com/labstack/echo` | Less popular than Gin | Gin |
| `github.com/sirupsen/logrus` | Structured logging in stdlib now | `log/slog` |
| `github.com/prometheus/client_golang` | VictoriaMetrics is lighter | `github.com/VictoriaMetrics/metrics` |
| `database/sql` directly | Missing ORM features | `gorm.io/gorm` |

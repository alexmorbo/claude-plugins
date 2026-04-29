# Go Service Project Structure

## Standard Directory Layout

```
service-name/
├── cmd/
│   └── server/
│       └── main.go              # Application entry point
│
├── domain/                       # LAYER 1: Pure business logic
│   ├── entity/                   # Business objects with identity
│   │   ├── user.go
│   │   ├── user_test.go          # Unit tests alongside code
│   │   └── order.go
│   ├── valueobject/              # Immutable value types
│   │   ├── user_id.go
│   │   ├── email.go
│   │   ├── email_test.go         # Unit tests
│   │   └── money.go
│   ├── repository/               # Data access interfaces
│   │   ├── user_repository.go
│   │   └── order_repository.go
│   ├── service/                  # Domain services (optional)
│   │   └── pricing_service.go
│   └── event/                    # Domain events (optional)
│       └── user_created.go
│
├── application/                  # LAYER 2: Use cases
│   ├── usecase/                  # Business operations
│   │   ├── create_user.go
│   │   ├── create_user_test.go   # Unit tests with mocks
│   │   ├── get_user.go
│   │   └── process_order.go
│   ├── dto/                      # Data transfer objects
│   │   ├── user_dto.go
│   │   ├── order_dto.go
│   │   └── errors.go
│   └── port/                     # External service interfaces
│       ├── email_sender.go
│       ├── payment_gateway.go
│       └── logger.go
│
├── infrastructure/               # LAYER 3: Implementations
│   ├── persistence/              # Database repositories
│   │   ├── user_repository_postgres.go
│   │   ├── order_repository_postgres.go
│   │   ├── models.go             # GORM database models
│   │   └── gorm_logger.go        # Custom GORM logger
│   ├── http/                     # HTTP clients for external APIs
│   │   ├── payment_adapter.go
│   │   └── logging_client.go     # HTTP RoundTripper with logging
│   ├── cache/                    # Redis/Valkey implementations
│   │   └── session_cache.go
│   ├── storage/                  # S3/MinIO implementations
│   │   └── file_storage.go
│   ├── config/                   # Configuration loading
│   │   └── config.go
│   ├── logger/                   # Logging setup
│   │   ├── logger.go             # slog configuration
│   │   ├── fields.go             # Structured field groups
│   │   └── context.go            # Request ID helpers
│   ├── metrics/                  # VictoriaMetrics collectors
│   │   ├── metrics.go            # Registry
│   │   ├── http_server.go        # HTTP metrics
│   │   └── database.go           # DB metrics
│   └── migrations/               # Database migrations
│       └── migrations.go
│
├── interface/                    # LAYER 4: Entry points
│   ├── http/                     # HTTP API
│   │   ├── handler/              # Request handlers
│   │   │   ├── user_handler.go
│   │   │   ├── order_handler.go
│   │   │   └── health_handler.go
│   │   ├── middleware/           # HTTP middleware
│   │   │   ├── logging.go
│   │   │   ├── request_id.go
│   │   │   ├── auth.go
│   │   │   └── metrics.go
│   │   ├── request/              # Request DTOs (optional)
│   │   │   └── user_request.go
│   │   └── response/             # Response DTOs (optional)
│   │       └── user_response.go
│   └── telegram/                 # Telegram bot (if needed)
│       └── message_handler.go
│
├── documentation/                # Service documentation
│   └── stories/                  # Task stories for agents
│       └── 001-initial-setup.md
│
├── tests/                        # Integration & E2E tests ONLY
│   ├── integration/              # Tests with real dependencies
│   │   ├── testcontainers.go     # Shared container setup
│   │   └── user_repository_integration_test.go
│   └── e2e/                      # Full API tests
│       ├── helpers.go            # E2E test helpers
│       └── api_test.go
│
├── .gitlab-ci.yml                # CI configuration
├── .golangci.yml                 # Linter configuration
├── go.mod                        # Go modules
├── go.sum                        # Dependencies checksum
├── Makefile                      # Build commands
├── Dockerfile                    # Container image
└── README.md                     # Project documentation
```

## File Templates

### cmd/server/main.go

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"

    "service/application/usecase"
    "service/infrastructure/config"
    "service/infrastructure/logger"
    "service/infrastructure/persistence"
    "service/interface/http/handler"
    "service/interface/http/middleware"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        slog.Error("failed to load config", "error", err)
        os.Exit(1)
    }

    // Initialize logger
    log := logger.New(logger.Config{
        Level:  cfg.LogLevel,
        Format: "json",
        Output: os.Stdout,
    })
    slog.SetDefault(log)

    // Initialize infrastructure
    db, err := persistence.NewPostgresDB(cfg.Database)
    if err != nil {
        slog.Error("failed to connect to database", "error", err)
        os.Exit(1)
    }

    // Initialize repositories
    userRepo := persistence.NewUserRepositoryPostgres(db)

    // Initialize use cases
    createUser := usecase.NewCreateUserUseCase(userRepo, log)
    getUser := usecase.NewGetUserUseCase(userRepo)

    // Initialize handlers
    userHandler := handler.NewUserHandler(createUser, getUser)
    healthHandler := handler.NewHealthHandler(db)

    // Setup Gin router
    gin.SetMode(gin.ReleaseMode)
    router := gin.New()

    // Global middleware
    router.Use(gin.Recovery())
    router.Use(middleware.RequestID())
    router.Use(middleware.Logging(log))

    // Health endpoints
    router.GET("/health/live", healthHandler.Live)
    router.GET("/health/ready", healthHandler.Ready)

    // API routes
    api := router.Group("/api/v1")
    {
        api.POST("/users", userHandler.Create)
        api.GET("/users/:id", userHandler.GetByID)
    }

    // Setup server
    server := &http.Server{
        Addr:         cfg.ServerAddr,
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Graceful shutdown
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh

        slog.Info("shutting down server")
        cancel()

        shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer shutdownCancel()

        if err := server.Shutdown(shutdownCtx); err != nil {
            slog.Error("server shutdown error", "error", err)
        }
    }()

    slog.Info("starting server", "addr", cfg.ServerAddr)
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        slog.Error("server error", "error", err)
        os.Exit(1)
    }
}
```

### infrastructure/config/config.go

```go
package config

import (
    "fmt"
    "os"
    "strconv"
)

type Config struct {
    ServerAddr string
    LogLevel   string

    Database DatabaseConfig
    Redis    RedisConfig
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    Name     string
    SSLMode  string
}

type RedisConfig struct {
    Addr     string
    Password string
    DB       int
}

func Load() (*Config, error) {
    cfg := &Config{
        ServerAddr: getEnv("SERVER_ADDR", ":8080"),
        LogLevel:   getEnv("LOG_LEVEL", "info"),

        Database: DatabaseConfig{
            Host:     getEnv("DATABASE_HOST", "localhost"),
            Port:     getEnvInt("DATABASE_PORT", 5432),
            User:     getEnv("DATABASE_USER", "postgres"),
            Password: getEnv("DATABASE_PASSWORD", ""),
            Name:     getEnv("DATABASE_NAME", "service"),
            SSLMode:  getEnv("DATABASE_SSLMODE", "disable"),
        },

        Redis: RedisConfig{
            Addr:     getEnv("REDIS_ADDR", "localhost:6379"),
            Password: getEnv("REDIS_PASSWORD", ""),
            DB:       getEnvInt("REDIS_DB", 0),
        },
    }

    if cfg.Database.Password == "" {
        return nil, fmt.Errorf("DATABASE_PASSWORD is required")
    }

    return cfg, nil
}

func (d DatabaseConfig) DSN() string {
    return fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        d.Host, d.Port, d.User, d.Password, d.Name, d.SSLMode,
    )
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if i, err := strconv.Atoi(value); err == nil {
            return i
        }
    }
    return defaultValue
}
```

### Makefile

```makefile
.PHONY: build test lint run clean

BINARY_NAME=service
GO_VERSION=1.24

build:
	CGO_ENABLED=0 go build -ldflags='-w -s' -trimpath -o bin/$(BINARY_NAME) cmd/server/main.go

test:
	go test ./... -v -race -coverprofile=coverage.out

test-domain:
	go test ./domain/... -v -race -coverprofile=domain-coverage.out

lint:
	golangci-lint run --timeout=5m

run:
	go run cmd/server/main.go

clean:
	rm -rf bin/ coverage.out

docker-build:
	docker build -t $(BINARY_NAME):latest .

migrate-up:
	go run cmd/migrate/main.go up

migrate-down:
	go run cmd/migrate/main.go down
```

### Dockerfile

```dockerfile
# Build stage
FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='-w -s' \
    -trimpath \
    -o /app/bin/service \
    cmd/server/main.go

# Runtime stage
FROM alpine:3.20

RUN apk --no-cache add ca-certificates tzdata

WORKDIR /app

COPY --from=builder /app/bin/service .

USER nobody:nobody

EXPOSE 8080

ENTRYPOINT ["./service"]
```

### .golangci.yml

```yaml
run:
  timeout: 5m
  go: "1.24"

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
    - gofmt
    - goimports
    - misspell
    - unconvert
    - gocritic

linters-settings:
  govet:
    check-shadowing: true
  goimports:
    local-prefixes: service

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck
```

## Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Package | lowercase, short | `user`, `order` |
| File | snake_case | `user_repository.go` |
| Entity | PascalCase | `User`, `Order` |
| Value Object | PascalCase | `UserID`, `Email` |
| Repository Interface | `<Entity>Repository` | `UserRepository` |
| Repository Impl | `<Entity>Repository<DB>` | `UserRepositoryPostgres` |
| Use Case | `<Action><Entity>UseCase` | `CreateUserUseCase` |
| Handler | `<Entity>Handler` | `UserHandler` |
| DTO Input | `<Action><Entity>Input` | `CreateUserInput` |
| DTO Output | `<Action><Entity>Output` | `CreateUserOutput` |

# Configuration Management

## Overview

Configuration is loaded from environment variables at startup. No config files, no live reloading.

## Configuration Structure

```go
// infrastructure/config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
    "strings"
    "time"
)

type Config struct {
    App      AppConfig
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
    S3       S3Config
    Telegram TelegramConfig
}

type AppConfig struct {
    Name        string
    Environment string // development, staging, production
    Debug       bool
}

type ServerConfig struct {
    Host         string
    Port         int
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
    IdleTimeout  time.Duration
}

type DatabaseConfig struct {
    Host            string
    Port            int
    User            string
    Password        string
    Name            string
    SSLMode         string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

type RedisConfig struct {
    Addr     string
    Password string
    DB       int
}

type S3Config struct {
    Endpoint  string
    Region    string
    Bucket    string
    AccessKey string
    SecretKey string
}

type TelegramConfig struct {
    Token         string
    WebhookURL    string
    WebhookSecret string
}
```

## Loading Configuration

```go
// infrastructure/config/config.go
package config

func Load() (*Config, error) {
    cfg := &Config{
        App: AppConfig{
            Name:        getEnv("APP_NAME", "service"),
            Environment: getEnv("APP_ENV", "development"),
            Debug:       getEnvBool("APP_DEBUG", false),
        },
        Server: ServerConfig{
            Host:         getEnv("SERVER_HOST", "0.0.0.0"),
            Port:         getEnvInt("SERVER_PORT", 8080),
            ReadTimeout:  getEnvDuration("SERVER_READ_TIMEOUT", 15*time.Second),
            WriteTimeout: getEnvDuration("SERVER_WRITE_TIMEOUT", 15*time.Second),
            IdleTimeout:  getEnvDuration("SERVER_IDLE_TIMEOUT", 60*time.Second),
        },
        Database: DatabaseConfig{
            Host:            getEnv("DATABASE_HOST", "localhost"),
            Port:            getEnvInt("DATABASE_PORT", 5432),
            User:            getEnv("DATABASE_USER", "postgres"),
            Password:        getEnvRequired("DATABASE_PASSWORD"),
            Name:            getEnv("DATABASE_NAME", "service"),
            SSLMode:         getEnv("DATABASE_SSLMODE", "disable"),
            MaxOpenConns:    getEnvInt("DATABASE_MAX_OPEN_CONNS", 25),
            MaxIdleConns:    getEnvInt("DATABASE_MAX_IDLE_CONNS", 10),
            ConnMaxLifetime: getEnvDuration("DATABASE_CONN_MAX_LIFETIME", 5*time.Minute),
        },
        Redis: RedisConfig{
            Addr:     getEnv("REDIS_ADDR", "localhost:6379"),
            Password: getEnv("REDIS_PASSWORD", ""),
            DB:       getEnvInt("REDIS_DB", 0),
        },
        S3: S3Config{
            Endpoint:  getEnv("S3_ENDPOINT", ""),
            Region:    getEnv("S3_REGION", "us-east-1"),
            Bucket:    getEnv("S3_BUCKET", ""),
            AccessKey: getEnv("S3_ACCESS_KEY", ""),
            SecretKey: getEnv("S3_SECRET_KEY", ""),
        },
        Telegram: TelegramConfig{
            Token:         getEnv("TELEGRAM_BOT_TOKEN", ""),
            WebhookURL:    getEnv("TELEGRAM_WEBHOOK_URL", ""),
            WebhookSecret: getEnv("TELEGRAM_WEBHOOK_SECRET", ""),
        },
    }

    if err := cfg.Validate(); err != nil {
        return nil, fmt.Errorf("config validation: %w", err)
    }

    return cfg, nil
}

func (c *Config) Validate() error {
    if c.Database.Password == "" {
        return fmt.Errorf("DATABASE_PASSWORD is required")
    }
    if c.App.Environment != "development" && c.App.Environment != "staging" && c.App.Environment != "production" {
        return fmt.Errorf("APP_ENV must be development, staging, or production")
    }
    return nil
}
```

## Helper Functions

```go
// infrastructure/config/helpers.go
package config

import (
    "os"
    "strconv"
    "strings"
    "time"
)

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvRequired(key string) string {
    value := os.Getenv(key)
    if value == "" {
        panic(fmt.Sprintf("required environment variable %s is not set", key))
    }
    return value
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if i, err := strconv.Atoi(value); err == nil {
            return i
        }
    }
    return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if b, err := strconv.ParseBool(value); err == nil {
            return b
        }
    }
    return defaultValue
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if d, err := time.ParseDuration(value); err == nil {
            return d
        }
    }
    return defaultValue
}

func getEnvSlice(key string, defaultValue []string, separator string) []string {
    if value := os.Getenv(key); value != "" {
        return strings.Split(value, separator)
    }
    return defaultValue
}
```

## Derived Values

```go
// infrastructure/config/config.go

func (c *ServerConfig) Addr() string {
    return fmt.Sprintf("%s:%d", c.Host, c.Port)
}

func (c *DatabaseConfig) DSN() string {
    return fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        c.Host, c.Port, c.User, c.Password, c.Name, c.SSLMode,
    )
}

func (c *DatabaseConfig) URL() string {
    return fmt.Sprintf(
        "postgres://%s:%s@%s:%d/%s?sslmode=%s",
        c.User, c.Password, c.Host, c.Port, c.Name, c.SSLMode,
    )
}

func (c *Config) IsProduction() bool {
    return c.App.Environment == "production"
}

func (c *Config) IsDevelopment() bool {
    return c.App.Environment == "development"
}
```

## Usage in main.go

```go
package main

import (
    "log/slog"
    "os"

    "service/infrastructure/config"
    "service/infrastructure/logger"
)

func main() {
    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        slog.Error("failed to load config", "error", err)
        os.Exit(1)
    }

    // Initialize logger based on config
    log := logger.New(logger.Config{
        Level:  logLevel(cfg),
        Format: "json",
        Output: os.Stdout,
    })
    slog.SetDefault(log)

    slog.Info("starting service",
        "name", cfg.App.Name,
        "environment", cfg.App.Environment,
        "addr", cfg.Server.Addr(),
    )

    // ... rest of initialization
}

func logLevel(cfg *config.Config) string {
    if cfg.App.Debug {
        return "debug"
    }
    if cfg.IsDevelopment() {
        return "debug"
    }
    return "info"
}
```

## Environment Variables Reference

### Application

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_NAME` | `service` | Service name for logging |
| `APP_ENV` | `development` | Environment: development, staging, production |
| `APP_DEBUG` | `false` | Enable debug logging |

### Server

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_HOST` | `0.0.0.0` | Server bind host |
| `SERVER_PORT` | `8080` | Server port |
| `SERVER_READ_TIMEOUT` | `15s` | HTTP read timeout |
| `SERVER_WRITE_TIMEOUT` | `15s` | HTTP write timeout |
| `SERVER_IDLE_TIMEOUT` | `60s` | HTTP idle timeout |

### Database

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_HOST` | `localhost` | PostgreSQL host |
| `DATABASE_PORT` | `5432` | PostgreSQL port |
| `DATABASE_USER` | `postgres` | PostgreSQL user |
| `DATABASE_PASSWORD` | **required** | PostgreSQL password |
| `DATABASE_NAME` | `service` | Database name |
| `DATABASE_SSLMODE` | `disable` | SSL mode |
| `DATABASE_MAX_OPEN_CONNS` | `25` | Max open connections |
| `DATABASE_MAX_IDLE_CONNS` | `10` | Max idle connections |
| `DATABASE_CONN_MAX_LIFETIME` | `5m` | Connection max lifetime |

### Redis

| Variable | Default | Description |
|----------|---------|-------------|
| `REDIS_ADDR` | `localhost:6379` | Redis address |
| `REDIS_PASSWORD` | `` | Redis password |
| `REDIS_DB` | `0` | Redis database |

### S3

| Variable | Default | Description |
|----------|---------|-------------|
| `S3_ENDPOINT` | `` | S3/MinIO endpoint |
| `S3_REGION` | `us-east-1` | S3 region |
| `S3_BUCKET` | `` | S3 bucket name |
| `S3_ACCESS_KEY` | `` | Access key |
| `S3_SECRET_KEY` | `` | Secret key |

### Telegram

| Variable | Default | Description |
|----------|---------|-------------|
| `TELEGRAM_BOT_TOKEN` | `` | Bot token |
| `TELEGRAM_WEBHOOK_URL` | `` | Webhook URL |
| `TELEGRAM_WEBHOOK_SECRET` | `` | Webhook secret |

## Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: service-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "secret-password"
  REDIS_PASSWORD: "redis-secret"
  TELEGRAM_BOT_TOKEN: "123456:ABC..."
```

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: service
          envFrom:
            - secretRef:
                name: service-secrets
          env:
            - name: APP_ENV
              value: "production"
            - name: DATABASE_HOST
              value: "postgres.database.svc"
```

## Testing with Configuration

```go
func TestConfig_Validate(t *testing.T) {
    tests := []struct {
        name    string
        setup   func()
        wantErr bool
    }{
        {
            name: "valid config",
            setup: func() {
                os.Setenv("DATABASE_PASSWORD", "test")
                os.Setenv("APP_ENV", "development")
            },
            wantErr: false,
        },
        {
            name: "missing password",
            setup: func() {
                os.Unsetenv("DATABASE_PASSWORD")
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.setup()
            _, err := config.Load()
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

## Best Practices

1. **Required vs Optional** - Use `getEnvRequired()` for secrets, `getEnv()` with defaults for optional
2. **Validation** - Validate config at startup, fail fast
3. **No Config Files** - Environment variables only for 12-factor compliance
4. **Derived Values** - Compute DSN, URLs as methods, not stored
5. **Immutable** - Config is read-only after initialization
6. **Testing** - Use environment variables in tests, reset after

# Structured Logging Format

## Overview

All Go services use `log/slog` (stdlib) with JSON output format for structured logging.

## Logger Configuration

```go
package logger

import (
    "io"
    "log/slog"
    "time"
)

type Config struct {
    Level  string    // "debug", "info", "warn", "error"
    Format string    // "json" or "text"
    Output io.Writer // typically os.Stdout
}

func New(cfg Config) *slog.Logger {
    level := parseLevel(cfg.Level)

    opts := &slog.HandlerOptions{
        Level:     level,
        AddSource: false,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            if a.Key == slog.TimeKey {
                if t, ok := a.Value.Any().(time.Time); ok {
                    a.Value = slog.StringValue(t.Format(time.RFC3339Nano))
                }
            }
            return a
        },
    }

    var handler slog.Handler
    if cfg.Format == "json" {
        handler = slog.NewJSONHandler(cfg.Output, opts)
    } else {
        handler = slog.NewTextHandler(cfg.Output, opts)
    }

    return slog.New(handler)
}

func parseLevel(level string) slog.Level {
    switch level {
    case "debug":
        return slog.LevelDebug
    case "warn":
        return slog.LevelWarn
    case "error":
        return slog.LevelError
    default:
        return slog.LevelInfo
    }
}
```

## Log Groups

Use five standard log groups for consistent log structure.

### 1. HTTP Group

For HTTP request/response logging:

```go
func HTTPFields(requestID, method, path, userAgent, remoteIP string, statusCode int, durationMs, requestSize, responseSize int64) slog.Attr {
    return slog.Group("http",
        slog.String("request_id", requestID),
        slog.String("method", method),
        slog.String("path", path),
        slog.String("user_agent", userAgent),
        slog.String("remote_ip", remoteIP),
        slog.Int("status_code", statusCode),
        slog.Int64("duration_ms", durationMs),
        slog.Int64("request_size", requestSize),
        slog.Int64("response_size", responseSize),
    )
}
```

**Output:**
```json
{
  "time": "2025-01-16T10:30:00.123456789Z",
  "level": "INFO",
  "msg": "HTTP request completed",
  "http": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "method": "POST",
    "path": "/api/v1/users",
    "user_agent": "Mozilla/5.0",
    "remote_ip": "10.90.12.50",
    "status_code": 201,
    "duration_ms": 45,
    "request_size": 256,
    "response_size": 128
  }
}
```

### 2. Database Group

For database operations:

```go
func DatabaseFields(operation, table string, durationMs, rowsAffected int64, errMsg string) slog.Attr {
    attrs := []any{
        slog.String("operation", operation),
        slog.String("table", table),
        slog.Int64("duration_ms", durationMs),
        slog.Int64("rows_affected", rowsAffected),
    }
    if errMsg != "" {
        attrs = append(attrs, slog.String("error", errMsg))
    }
    return slog.Group("database", attrs...)
}
```

**Output:**
```json
{
  "time": "2025-01-16T10:30:00.234567890Z",
  "level": "INFO",
  "msg": "Database query",
  "database": {
    "operation": "insert",
    "table": "users",
    "duration_ms": 12,
    "rows_affected": 1
  }
}
```

### 3. External API Group

For external HTTP calls:

```go
func ExternalFields(url, method string, statusCode int, durationMs int64, errMsg string) slog.Attr {
    attrs := []any{
        slog.String("url", url),
        slog.String("method", method),
        slog.Int("status_code", statusCode),
        slog.Int64("duration_ms", durationMs),
    }
    if errMsg != "" {
        attrs = append(attrs, slog.String("error", errMsg))
    }
    return slog.Group("external", attrs...)
}
```

**Output:**
```json
{
  "time": "2025-01-16T10:30:00.345678901Z",
  "level": "INFO",
  "msg": "External API call",
  "external": {
    "url": "https://api.payment.com/charge",
    "method": "POST",
    "status_code": 200,
    "duration_ms": 456
  }
}
```

### 4. Application Group

For business events:

```go
func ApplicationFields(event string, attrs ...any) slog.Attr {
    allAttrs := []any{slog.String("event", event)}
    allAttrs = append(allAttrs, attrs...)
    return slog.Group("application", allAttrs...)
}
```

**Standard events:**
- `user_created` - New user registered
- `user_updated` - User profile changed
- `order_created` - New order placed
- `order_completed` - Order fulfilled
- `payment_processed` - Payment succeeded
- `payment_failed` - Payment failed

**Output:**
```json
{
  "time": "2025-01-16T10:30:00.456789012Z",
  "level": "INFO",
  "msg": "User created successfully",
  "application": {
    "event": "user_created",
    "user_id": "123",
    "email": "user@example.com"
  }
}
```

### 5. Bot Group (for Telegram services)

For Telegram bot operations:

```go
func BotFields(updateID, chatID, userID int64, messageType string, attrs ...any) slog.Attr {
    allAttrs := []any{
        slog.Int64("update_id", updateID),
        slog.Int64("chat_id", chatID),
        slog.Int64("user_id", userID),
        slog.String("message_type", messageType),
    }
    allAttrs = append(allAttrs, attrs...)
    return slog.Group("bot", allAttrs...)
}
```

**Output:**
```json
{
  "time": "2025-01-16T10:30:00.567890123Z",
  "level": "INFO",
  "msg": "Message received",
  "bot": {
    "update_id": 123456789,
    "chat_id": -100123456789,
    "user_id": 987654321,
    "message_type": "command"
  }
}
```

## Request ID Correlation

### Context Helpers

```go
package logger

import "context"

type contextKey string

const RequestIDKey contextKey = "request_id"

func WithRequestID(ctx context.Context, requestID string) context.Context {
    return context.WithValue(ctx, RequestIDKey, requestID)
}

func GetRequestID(ctx context.Context) string {
    if requestID, ok := ctx.Value(RequestIDKey).(string); ok {
        return requestID
    }
    return ""
}
```

### Usage

```go
// Middleware sets request ID
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()
        ctx := logger.WithRequestID(r.Context(), requestID)
        w.Header().Set("X-Request-ID", requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// All subsequent logs include request ID
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    requestID := logger.GetRequestID(r.Context())

    slog.InfoContext(r.Context(), "Creating user",
        slog.String("request_id", requestID),
        // ...
    )
}
```

## GORM Logger Integration

Custom GORM logger that outputs to slog:

```go
package persistence

import (
    "context"
    "log/slog"
    "regexp"
    "time"

    "gorm.io/gorm/logger"
)

type GormLogger struct {
    logger *slog.Logger
}

func NewGormLogger(l *slog.Logger) *GormLogger {
    return &GormLogger{logger: l}
}

func (l *GormLogger) LogMode(level logger.LogLevel) logger.Interface {
    return l
}

func (l *GormLogger) Info(ctx context.Context, msg string, args ...interface{}) {
    l.logger.InfoContext(ctx, msg)
}

func (l *GormLogger) Warn(ctx context.Context, msg string, args ...interface{}) {
    l.logger.WarnContext(ctx, msg)
}

func (l *GormLogger) Error(ctx context.Context, msg string, args ...interface{}) {
    l.logger.ErrorContext(ctx, msg)
}

func (l *GormLogger) Trace(ctx context.Context, begin time.Time, fc func() (string, int64), err error) {
    elapsed := time.Since(begin)
    sql, rows := fc()

    operation := extractOperation(sql)
    table := extractTable(sql)

    level := slog.LevelInfo
    errStr := ""
    if err != nil {
        level = slog.LevelError
        errStr = err.Error()
    }

    l.logger.Log(ctx, level, "Database query",
        DatabaseFields(operation, table, elapsed.Milliseconds(), rows, errStr),
    )
}

var operationRegex = regexp.MustCompile(`(?i)^(SELECT|INSERT|UPDATE|DELETE)`)
var tableRegex = regexp.MustCompile(`(?i)(?:FROM|INTO|UPDATE)\s+["']?(\w+)["']?`)

func extractOperation(sql string) string {
    if match := operationRegex.FindString(sql); match != "" {
        return strings.ToLower(match)
    }
    return "unknown"
}

func extractTable(sql string) string {
    if matches := tableRegex.FindStringSubmatch(sql); len(matches) > 1 {
        return matches[1]
    }
    return "unknown"
}
```

## HTTP Logging Middleware

```go
package middleware

import (
    "log/slog"
    "net/http"
    "time"

    "service/infrastructure/logger"
)

type responseWriter struct {
    http.ResponseWriter
    statusCode   int
    bytesWritten int64
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.bytesWritten += int64(n)
    return n, err
}

func Logging(log *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            requestID := logger.GetRequestID(r.Context())

            wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
            next.ServeHTTP(wrapped, r)

            log.InfoContext(r.Context(), "HTTP request completed",
                logger.HTTPFields(
                    requestID,
                    r.Method,
                    r.URL.Path,
                    r.UserAgent(),
                    r.RemoteAddr,
                    wrapped.statusCode,
                    time.Since(start).Milliseconds(),
                    r.ContentLength,
                    wrapped.bytesWritten,
                ),
            )
        })
    }
}
```

## Environment Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_LEVEL` | `info` | Minimum log level |
| `LOG_FORMAT` | `json` | Output format (json/text) |

Use `text` format for local development, `json` for production.

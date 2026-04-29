# Observability Stack

## Overview

Our observability stack consists of:

| Component | Tool | Purpose |
|-----------|------|---------|
| Metrics | VictoriaMetrics | Time-series metrics storage |
| Logs | VictoriaLogs | Log aggregation and search |
| Traces | OpenTelemetry (future) | Distributed tracing |

## Metrics

### VictoriaMetrics Client

Use the native VictoriaMetrics client for Prometheus-compatible metrics:

```go
import "github.com/VictoriaMetrics/metrics"
```

### Standard Metrics

Every service should expose these metrics:

#### HTTP Server Metrics

```go
var (
    httpRequestsTotal = metrics.NewCounter(`http_requests_total{service="%s",handler="%s",method="%s",status="%d"}`)
    httpRequestDuration = metrics.NewHistogram(`http_request_duration_seconds{service="%s",handler="%s"}`)
    httpRequestsInFlight = metrics.NewGauge(`http_requests_in_flight{service="%s"}`, nil)
)
```

#### Database Metrics

```go
var (
    dbQueriesTotal = metrics.NewCounter(`db_queries_total{service="%s",operation="%s",table="%s"}`)
    dbQueryDuration = metrics.NewHistogram(`db_query_duration_seconds{service="%s",operation="%s"}`)
    dbConnectionsOpen = metrics.NewGauge(`db_connections_open{service="%s"}`, func() float64 {
        stats := db.Stats()
        return float64(stats.OpenConnections)
    })
)
```

#### Business Metrics

Define service-specific business metrics:

```go
var (
    ordersCreatedTotal = metrics.NewCounter(`orders_created_total{service="order-service"}`)
    ordersValue = metrics.NewHistogram(`orders_value_dollars{service="order-service"}`)
    usersActiveGauge = metrics.NewGauge(`users_active{service="auth-service"}`, nil)
)
```

### Metrics Endpoint

Expose metrics at `/metrics`:

```go
func MetricsHandler(w http.ResponseWriter, r *http.Request) {
    metrics.WritePrometheus(w, true)
}

mux.HandleFunc("GET /metrics", MetricsHandler)
```

### Metric Naming Conventions

Follow Prometheus naming conventions:

| Type | Pattern | Example |
|------|---------|---------|
| Counter | `*_total` | `http_requests_total` |
| Gauge | current value | `db_connections_open` |
| Histogram | `*_seconds`, `*_bytes` | `http_request_duration_seconds` |

Labels:
- `service` - service name
- `handler` - HTTP handler or operation name
- `method` - HTTP method
- `status` - HTTP status code
- `operation` - DB operation (select, insert, update, delete)
- `table` - database table name

## Logs

### Format

All logs must be in JSON format. See [logging-format.md](logging-format.md) for detailed specification.

### Log Levels

| Level | Usage |
|-------|-------|
| `DEBUG` | Detailed debugging information (disabled in production) |
| `INFO` | Normal operational messages |
| `WARN` | Warning conditions that should be reviewed |
| `ERROR` | Error conditions requiring attention |

### Structured Fields

Use consistent field groups for correlation:

```go
// HTTP requests
slog.Info("request completed",
    slog.Group("http",
        slog.String("request_id", requestID),
        slog.String("method", "POST"),
        slog.String("path", "/api/v1/users"),
        slog.Int("status_code", 201),
        slog.Int64("duration_ms", 45),
    ),
)

// Database operations
slog.Info("query executed",
    slog.Group("database",
        slog.String("operation", "insert"),
        slog.String("table", "users"),
        slog.Int64("duration_ms", 12),
        slog.Int64("rows_affected", 1),
    ),
)
```

### Log Correlation

Use request IDs to correlate logs across the request lifecycle:

```go
// Middleware adds request ID to context
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()
        ctx := context.WithValue(r.Context(), RequestIDKey, requestID)
        w.Header().Set("X-Request-ID", requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// All logs include request ID
func GetRequestID(ctx context.Context) string {
    if id, ok := ctx.Value(RequestIDKey).(string); ok {
        return id
    }
    return ""
}
```

## Health Checks

### Endpoints

Every service must expose:

| Endpoint | Purpose | Checks |
|----------|---------|--------|
| `GET /health/live` | Liveness probe | Service is running |
| `GET /health/ready` | Readiness probe | Service can handle requests |

### Implementation

```go
type HealthHandler struct {
    db    *gorm.DB
    redis *redis.Client
}

// Liveness - always returns 200 if service is running
func (h *HealthHandler) Live(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}

// Readiness - checks dependencies
func (h *HealthHandler) Ready(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    // Check database
    if sqlDB, err := h.db.DB(); err != nil || sqlDB.PingContext(ctx) != nil {
        http.Error(w, "database unavailable", http.StatusServiceUnavailable)
        return
    }

    // Check Redis
    if err := h.redis.Ping(ctx).Err(); err != nil {
        http.Error(w, "redis unavailable", http.StatusServiceUnavailable)
        return
    }

    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}
```

### Kubernetes Probes

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

## Future: Distributed Tracing

OpenTelemetry tracing is planned for future implementation:

```go
// Future implementation example
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*User, error) {
    ctx, span := otel.Tracer("user-service").Start(ctx, "CreateUser")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.email", input.Email),
    )

    // ...
}
```

## Alerting Guidelines

### Critical Alerts (Page)
- Service down (no successful health checks)
- Error rate > 5% for 5 minutes
- Latency P99 > 5s for 5 minutes

### Warning Alerts (Notify)
- Error rate > 1% for 10 minutes
- Latency P95 > 1s for 10 minutes
- Database connections > 80% capacity
- Memory usage > 80%

Create new Go service with Clean Architecture structure

Initialize a new Go service following Clean Architecture principles and homelab patterns:

## 🎯 Usage
**Command**: `/ca-init-go [service-name]`
**Example**: `/ca-init-go nutrition-service`

## 🏗️ What it creates

### Complete Clean Architecture Structure
```
nutrition-service/
├── cmd/
│   └── server/
│       └── main.go            # Application entry point
├── domain/                    # Pure business logic
│   ├── entity/                # Business entities
│   ├── valueobject/           # Value objects
│   ├── service/               # Domain services
│   ├── repository/            # Repository interfaces
│   └── event/                 # Domain events
├── application/               # Use cases & orchestration
│   ├── usecase/               # Single-purpose use cases
│   ├── dto/                   # Application DTOs
│   ├── port/                  # External service interfaces
│   └── service/               # Application services
├── infrastructure/            # Implementation details
│   ├── persistence/           # Database implementations
│   ├── http/                  # External service clients
│   ├── messaging/             # Event publishers/consumers
│   └── cache/                 # Caching implementations
├── interface/                 # External interfaces
│   ├── http/                  # HTTP handlers
│   ├── request/               # HTTP request models
│   ├── response/              # HTTP response models
│   └── middleware/            # HTTP middleware
├── tests/                     # Test files
│   ├── unit/                  # Unit tests (domain focus)
│   ├── integration/           # Integration tests
│   └── e2e/                   # End-to-end tests
├── go.mod                     # Go module
├── go.sum                     # Dependencies
├── .gitlab-ci.yml             # CI configuration
├── Dockerfile                 # Container configuration
├── README.md                  # Service documentation
└── .gitignore                 # Git ignore rules
```

## 📝 Generated Code Examples

### Domain Entity
```go
// domain/entity/nutrition_profile.go
package entity

import "time"

type NutritionProfile struct {
    id          NutritionProfileID
    userID      UserID
    calories    Calories
    macros      MacroProfile
    createdAt   time.Time
    updatedAt   time.Time
}

func NewNutritionProfile(userID UserID, calories Calories, macros MacroProfile) (*NutritionProfile, error) {
    if err := calories.Validate(); err != nil {
        return nil, err
    }

    return &NutritionProfile{
        id:        GenerateNutritionProfileID(),
        userID:    userID,
        calories:  calories,
        macros:    macros,
        createdAt: time.Now(),
        updatedAt: time.Now(),
    }, nil
}
```

### Use Case
```go
// application/usecase/create_nutrition_profile.go
package usecase

type CreateNutritionProfileUseCase struct {
    nutritionRepo repository.NutritionRepository
    eventBus      port.EventBus
}

func (uc *CreateNutritionProfileUseCase) Execute(ctx context.Context, req CreateNutritionProfileRequest) (*NutritionProfile, error) {
    // Domain validation
    profile, err := entity.NewNutritionProfile(req.UserID, req.Calories, req.Macros)
    if err != nil {
        return nil, err
    }

    // Save through repository
    if err := uc.nutritionRepo.Save(ctx, profile); err != nil {
        return nil, err
    }

    // Publish domain event
    event := event.NewNutritionProfileCreated(profile.ID(), profile.UserID())
    uc.eventBus.Publish(ctx, event)

    return profile, nil
}
```

### HTTP Handler
```go
// interface/http/nutrition_handler.go
package http

func (h *NutritionHandler) CreateProfile(c *gin.Context) {
    var req request.CreateNutritionProfileRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, response.Error("Invalid request"))
        return
    }

    profile, err := h.createProfileUseCase.Execute(c.Request.Context(), req.ToUseCase())
    if err != nil {
        c.JSON(http.StatusInternalServerError, response.Error(err.Error()))
        return
    }

    c.JSON(http.StatusCreated, response.FromNutritionProfile(profile))
}
```

## ⚙️ Configuration Files

### GitLab CI Configuration
```yaml
# .gitlab-ci.yml
include:
  - project: homelab/homelab-ci
    file: go-service.yml

variables:
  SERVICE_NAME: "nutrition-service"
  COVERAGE_THRESHOLD: "85"
```

### Go Module
```go
// go.mod
module nutrition-service

go 1.23

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/google/uuid v1.3.0
    // Common library for infrastructure
)
```

### Dockerfile
```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server cmd/server/main.go

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

## 📚 Additional Features

### Testing Setup
- Unit test templates for domain entities
- Integration test examples with testcontainers
- Mocking interfaces for use cases
- Coverage configuration

### Documentation
- Service README with API documentation
- Architecture decision records (ADRs)
- Development setup instructions
- Deployment guides

### Development Tools
- golangci-lint configuration
- Pre-commit hooks setup
- Docker Compose for local development
- Makefile with common commands

## 🚀 Next Steps After Creation

1. **Navigate to Service**: `cd nutrition-service`
2. **Initialize Git**: `git init && git add . && git commit -m "feat: initial CA structure"`
3. **Install Dependencies**: `go mod tidy`
4. **Run Tests**: `go test ./...`
5. **Start Development**: Begin implementing domain entities and use cases

## 🔗 Related Commands

- `/ca-validate-go` - Validate the created service
- `/ca-validate-frontend` - For frontend applications
- Use after creation to verify compliance

## 📖 Template Sources

Templates are based on:
- homelab Clean Architecture patterns
- eat-it project CA documentation
- Production-ready Go service structure
- Industry best practices for microservices

See: `homelab-ci/documentation/ca/` for detailed CA documentation.
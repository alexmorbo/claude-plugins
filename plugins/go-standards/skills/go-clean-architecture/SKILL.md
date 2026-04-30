---
name: go-clean-architecture
description: "Enforces Clean Architecture patterns in Go microservices. Use when implementing features, creating new components, modifying existing code, or reviewing Go service structure. Applies to any task involving Go code in domain, application, infrastructure, or interface layers."
---

# Go Clean Architecture Skill

This skill ensures all Go code follows Clean Architecture principles from `${CLAUDE_PLUGIN_ROOT}/references/clean-architecture.md`.

## Layer Structure

```
service/
├── cmd/server/main.go           # Entry point (wiring only)
├── domain/                       # Layer 1: Business logic (innermost)
│   ├── entity/                   # Business objects with identity
│   ├── valueobject/              # Immutable values without identity
│   ├── repository/               # Repository interfaces (NOT implementations)
│   ├── service/                  # Domain services (complex business logic)
│   └── event/                    # Domain events
├── application/                  # Layer 2: Use cases
│   ├── usecase/                  # Application services (orchestration)
│   ├── dto/                      # Data transfer objects
│   └── port/                     # Port interfaces (inbound/outbound)
├── infrastructure/               # Layer 3: External concerns
│   ├── persistence/              # Repository implementations (PostgreSQL, Redis)
│   ├── external/                 # External service clients
│   ├── messaging/                # Message queue implementations
│   └── config/                   # Configuration loading
└── interface/                    # Layer 4: Entry points (outermost)
    ├── http/                     # HTTP handlers, middleware, routes
    │   ├── handler/
    │   ├── middleware/
    │   └── router/
    └── grpc/                     # gRPC handlers (if needed)
```

## Dependency Rule

**Dependencies point INWARD only:**

```
interface → infrastructure → application → domain
```

- Domain depends on NOTHING
- Application depends only on Domain
- Infrastructure depends on Domain and Application
- Interface depends on all inner layers

## Enforcement Rules

### 1. Domain Layer Purity

Domain layer MUST NOT import:
- `net/http`, `github.com/gin-gonic/gin`
- `gorm.io/gorm`, `database/sql`
- `github.com/redis/go-redis`
- Any infrastructure packages

Domain layer CAN import:
- Standard library (strings, time, errors, fmt)
- Domain packages only

### 2. Repository Pattern

```go
// domain/repository/user_repository.go - INTERFACE only
type UserRepository interface {
    Save(ctx context.Context, user *entity.User) error
    FindByID(ctx context.Context, id valueobject.UserID) (*entity.User, error)
    FindByEmail(ctx context.Context, email valueobject.Email) (*entity.User, error)
}

// infrastructure/persistence/user_repository_postgres.go - IMPLEMENTATION
type UserRepositoryPostgres struct {
    db *gorm.DB
}

func (r *UserRepositoryPostgres) Save(ctx context.Context, user *entity.User) error {
    // GORM implementation
}
```

### 3. Use Case Pattern

```go
// application/usecase/create_user.go
type CreateUserUseCase struct {
    userRepo   repository.UserRepository  // Interface from domain
    eventBus   port.EventPublisher        // Interface from application
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, input dto.CreateUserInput) (*dto.CreateUserOutput, error) {
    // 1. Create domain objects
    email, err := valueobject.NewEmail(input.Email)
    if err != nil {
        return nil, fmt.Errorf("invalid email: %w", err)
    }

    // 2. Business logic
    user, err := entity.NewUser(email, input.Name)
    if err != nil {
        return nil, fmt.Errorf("create user: %w", err)
    }

    // 3. Persist via repository
    if err := uc.userRepo.Save(ctx, user); err != nil {
        return nil, fmt.Errorf("save user: %w", err)
    }

    // 4. Return DTO
    return &dto.CreateUserOutput{
        ID:    user.ID().String(),
        Email: user.Email().Value(),
    }, nil
}
```

### 4. Handler Pattern

```go
// interface/http/handler/user_handler.go
type UserHandler struct {
    createUser *usecase.CreateUserUseCase
}

func (h *UserHandler) Create(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    output, err := h.createUser.Execute(c.Request.Context(), dto.CreateUserInput{
        Email: req.Email,
        Name:  req.Name,
    })
    if err != nil {
        // Map domain errors to HTTP status
        c.JSON(mapErrorToStatus(err), gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, output)
}
```

## Code-First Planning Integration

When planning (Opus), write COMPLETE code in story files following this structure:

```markdown
### 1. Domain Layer

#### File: `domain/valueobject/email.go`

```go
// Complete implementation - Opus writes ALL code
```

#### File: `domain/entity/user.go`

```go
// Complete implementation - Opus writes ALL code
```

### 2. Application Layer
...
```

**Haiku only copies this code to files. All decisions made by Opus.**

## When Creating New Code

1. **New feature** → Start from domain (entities, value objects)
2. **New endpoint** → Create use case first, then handler
3. **New integration** → Add port interface in application, implement in infrastructure
4. **Refactoring** → Verify no layer violations introduced

## Validation Checklist

Before completing implementation:

- [ ] Domain has no infrastructure imports
- [ ] Repositories are interfaces in domain, implementations in infrastructure
- [ ] Use cases orchestrate domain objects, don't contain business logic
- [ ] Handlers only do HTTP concerns (binding, response formatting)
- [ ] All dependencies injected via constructors
- [ ] No circular dependencies between packages

## Quick Verification Commands

```bash
# Check domain imports (should show only stdlib + domain packages)
go list -f '{{.Imports}}' ./domain/...

# Build to verify compilation
go build ./...

# Run linter
golangci-lint run ./...
```

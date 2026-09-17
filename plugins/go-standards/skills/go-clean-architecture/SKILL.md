---
name: go-clean-architecture
description: "Clean Architecture patterns for Go services, with explicit applicability criteria. Use when implementing features, creating new components, modifying existing code, or reviewing Go service structure. Read the applicability section first - four layer directories are for complex domains, not for every service."
---

# Go Clean Architecture Skill

This skill covers Clean Architecture principles from `${CLAUDE_PLUGIN_ROOT}/references/clean-architecture.md` **and when to apply them**.

## Applicability: Read This First

The principles are near-universal. **The four layer directories are not.**

Splitting a service into `domain/application/infrastructure/interface` buys
isolation at the cost of indirection. For a small service that trade is a loss,
and shipping it anyway is the most common form of over-engineering in Go. Three
Dots Labs - who popularised Clean Architecture in Go - draw the same line:
the goal is not to use an architecture, it is to use something useful here.

### Use the four-layer structure when

- The domain is genuinely complex - real rules, real invariants, not CRUD
- Several people work in the codebase at once and need stable boundaries
- The service will be maintained for years
- A domain has more than one consumer, or more than one transport

### Use a flat structure when

- The domain is thin, or the service is mostly pipes
- One or two people own it
- There is a single transport and a single consumer per domain
- You cannot yet name what would go in `application/` that is not the use case itself

### Default for a small service

```
cmd/<service>/main.go          entry point
internal/
  config/                      configuration
  observability/               logging, metrics, health
  <domain>/                    model, logic and data access together
  <domain>/                    one package per domain, not per artifact type
  wiring/                      dependency construction
  httpapi/                     handlers
```

Group by **feature**, not by artifact type. `internal/catalog` holds the catalog
model, its rules and its repository; `internal/order` holds orders. A reader
opening one directory sees the whole feature.

The principles still apply in this layout:

- Dependencies point inward: `httpapi` → `catalog`, never the reverse
- Interfaces are declared by the **consumer**, not next to the implementation
- Business rules do not import `net/http`, `pgx` or `gorm`
- Dependencies are injected via constructors, assembled in `wiring/`

### Growing into layers

Introduce a layer when something concrete forces it - a second transport, a
second consumer of the same domain, a domain package that has outgrown one
directory. Extracting `domain/` from a working `internal/catalog` is a
mechanical refactor. Guessing the split up front, before the domain is
understood, rarely survives contact with it.

**Do not restructure an existing service to match this skill.** Match the
codebase you are in.

## Layer Structure

For services that meet the applicability criteria above:

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

## Contract-First Planning Integration

When planning, the story states **which structure applies and why**, then gives
the contracts - signatures, schemas, error shapes, invariants. Not bodies.

```markdown
## Scope

Flat structure: single transport, one consumer per domain. `internal/catalog`
holds model, rules and repository together.

## Contracts

```go
package catalog

type Repository interface {
    Upsert(ctx context.Context, item Item) error
    ListByOwner(ctx context.Context, owner OwnerID) ([]Item, error)
}
```

`Repository` is declared in `internal/catalog` because the use case there
consumes it; the Postgres implementation lives in the same package and is
constructed in `internal/wiring`.

Invariant: `Upsert` is idempotent on `(owner_id, name)`.
```

Implementation writes the bodies in the repository, against `go build`,
`golangci-lint` and `go test`. See
`${CLAUDE_PLUGIN_ROOT}/references/agent-workflow/planning-guide.md`.

## When Creating New Code

1. **New feature** → Start from domain (entities, value objects)
2. **New endpoint** → Create use case first, then handler
3. **New integration** → Add port interface in application, implement in infrastructure
4. **Refactoring** → Verify no layer violations introduced

## Validation Checklist

Applies to both structures:

- [ ] Business rules import no transport or storage packages
- [ ] Interfaces declared by the consumer, not beside the implementation
- [ ] Use cases orchestrate; they do not hold business rules
- [ ] Handlers do only transport concerns (binding, status mapping)
- [ ] All dependencies injected via constructors, assembled in one place
- [ ] No circular dependencies between packages

Additionally, for the four-layer structure:

- [ ] `domain/` has no infrastructure imports
- [ ] Repository interfaces in `domain/`, implementations in `infrastructure/`
- [ ] Layer dependencies point inward only

## Quick Verification Commands

```bash
# Four-layer: domain should import only stdlib + domain packages
go list -f '{{.Imports}}' ./domain/...

# Flat: check a domain package for transport/storage leakage
go list -f '{{.ImportPath}} {{.Imports}}' ./internal/... | grep -E 'net/http|gorm|pgx'

# Build and lint
go build ./...
golangci-lint run ./...
```

For a hard guarantee rather than a checklist, encode the rule as a lint
policy - `depguard` in golangci-lint, or an AST test behind a build tag that
fails the build when a forbidden import appears. A rule enforced by tooling
survives; a rule living in a document does not.

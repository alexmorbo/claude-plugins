# Clean Architecture for Go Services

## Overview

Our Go services follow Clean Architecture (CA) principles combined with Domain-Driven Design (DDD) patterns. This ensures:
- Testability through dependency inversion
- Framework independence
- Business logic isolation
- Clear separation of concerns

## The Four Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Interface Layer                          │
│            (HTTP handlers, CLI, message handlers)           │
├─────────────────────────────────────────────────────────────┤
│                  Infrastructure Layer                       │
│        (Databases, external APIs, config, logging)          │
├─────────────────────────────────────────────────────────────┤
│                   Application Layer                         │
│              (Use cases, DTOs, port interfaces)             │
├─────────────────────────────────────────────────────────────┤
│                     Domain Layer                            │
│    (Entities, value objects, repository interfaces)         │
└─────────────────────────────────────────────────────────────┘
              ↑ Dependencies point INWARD ↑
```

## Dependency Rule

**Dependencies must point inward.** Inner layers must not know about outer layers.

- Domain knows nothing about Application, Infrastructure, or Interface
- Application knows only Domain
- Infrastructure knows Domain and Application (implements their interfaces)
- Interface knows all layers (wires everything together)

## Layer Details

### Domain Layer

The innermost layer containing pure business logic with **zero external dependencies**.

**Contains:**
- **Entities** - Business objects with identity and lifecycle
- **Value Objects** - Immutable objects defined by their attributes
- **Repository Interfaces** - Contracts for data access (no implementations)
- **Domain Events** - Business events that occurred
- **Domain Services** - Stateless operations on multiple entities

**Rules:**
- No imports from `database/sql`, `net/http`, `gorm`, etc.
- Only stdlib packages allowed: `time`, `errors`, `fmt`, `strings`, `context`
- Exception: `github.com/google/uuid` for ID generation

```go
// domain/entity/user.go
package entity

import (
    "time"
    "project/domain/valueobject"
)

type User struct {
    id        valueobject.UserID
    email     valueobject.Email
    name      string
    createdAt time.Time
}

func NewUser(email valueobject.Email, name string) (*User, error) {
    if name == "" {
        return nil, ErrInvalidName
    }
    return &User{
        id:        valueobject.NewUserID(),
        email:     email,
        name:      name,
        createdAt: time.Now().UTC(),
    }, nil
}

func (u *User) ID() valueobject.UserID { return u.id }
func (u *User) Email() valueobject.Email { return u.email }
```

```go
// domain/valueobject/email.go
package valueobject

import (
    "errors"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

type Email struct {
    value string
}

func NewEmail(value string) (Email, error) {
    if !emailRegex.MatchString(value) {
        return Email{}, errors.New("invalid email format")
    }
    return Email{value: value}, nil
}

func (e Email) Value() string { return e.value }
```

```go
// domain/repository/user_repository.go
package repository

import (
    "context"
    "project/domain/entity"
    "project/domain/valueobject"
)

type UserRepository interface {
    Save(ctx context.Context, user *entity.User) error
    FindByID(ctx context.Context, id valueobject.UserID) (*entity.User, error)
    FindByEmail(ctx context.Context, email valueobject.Email) (*entity.User, error)
    Delete(ctx context.Context, id valueobject.UserID) error
}
```

### Application Layer

Orchestrates business workflows through **use cases**.

**Contains:**
- **Use Cases** - Single-purpose business operations
- **DTOs** - Data transfer objects for input/output
- **Ports** - Interfaces for external services
- **Application Services** - Optional coordination layer

```go
// application/usecase/create_user.go
package usecase

import (
    "context"
    "project/domain/entity"
    "project/domain/repository"
    "project/domain/valueobject"
    "project/application/dto"
    "project/application/port"
)

type CreateUserUseCase struct {
    userRepo    repository.UserRepository
    emailSender port.EmailSender
    logger      port.Logger
}

func NewCreateUserUseCase(
    userRepo repository.UserRepository,
    emailSender port.EmailSender,
    logger port.Logger,
) *CreateUserUseCase {
    return &CreateUserUseCase{
        userRepo:    userRepo,
        emailSender: emailSender,
        logger:      logger,
    }
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, input dto.CreateUserInput) (*dto.CreateUserOutput, error) {
    email, err := valueobject.NewEmail(input.Email)
    if err != nil {
        return nil, dto.ErrInvalidEmail
    }

    existing, _ := uc.userRepo.FindByEmail(ctx, email)
    if existing != nil {
        return nil, dto.ErrEmailAlreadyExists
    }

    user, err := entity.NewUser(email, input.Name)
    if err != nil {
        return nil, err
    }

    if err := uc.userRepo.Save(ctx, user); err != nil {
        return nil, err
    }

    uc.emailSender.SendWelcome(ctx, email)

    return &dto.CreateUserOutput{
        ID:    user.ID().Value(),
        Email: user.Email().Value(),
        Name:  input.Name,
    }, nil
}
```

```go
// application/dto/user.go
package dto

import "errors"

var (
    ErrInvalidEmail      = errors.New("invalid email")
    ErrEmailAlreadyExists = errors.New("email already exists")
)

type CreateUserInput struct {
    Email string
    Name  string
}

type CreateUserOutput struct {
    ID    string
    Email string
    Name  string
}
```

```go
// application/port/email_sender.go
package port

import "context"
import "project/domain/valueobject"

type EmailSender interface {
    SendWelcome(ctx context.Context, email valueobject.Email) error
}
```

### Infrastructure Layer

Implements interfaces defined in Domain and Application layers.

**Contains:**
- **Repository Implementations** - Database access using GORM/SQL
- **External Service Adapters** - HTTP clients, message queues
- **Configuration** - Environment loading
- **Logging** - Logger implementation
- **Metrics** - Prometheus/VictoriaMetrics collectors

```go
// infrastructure/persistence/user_repository_postgres.go
package persistence

import (
    "context"
    "gorm.io/gorm"
    "project/domain/entity"
    "project/domain/repository"
    "project/domain/valueobject"
)

type UserRepositoryPostgres struct {
    db *gorm.DB
}

func NewUserRepositoryPostgres(db *gorm.DB) repository.UserRepository {
    return &UserRepositoryPostgres{db: db}
}

func (r *UserRepositoryPostgres) Save(ctx context.Context, user *entity.User) error {
    model := r.toModel(user)
    return r.db.WithContext(ctx).Save(model).Error
}

func (r *UserRepositoryPostgres) FindByID(ctx context.Context, id valueobject.UserID) (*entity.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).First(&model, "id = ?", id.Value()).Error; err != nil {
        return nil, err
    }
    return r.toEntity(&model), nil
}

// Mapper methods - separate DB models from domain entities
func (r *UserRepositoryPostgres) toModel(u *entity.User) *UserModel {
    return &UserModel{
        ID:        u.ID().Value(),
        Email:     u.Email().Value(),
        Name:      u.Name(),
        CreatedAt: u.CreatedAt(),
    }
}

func (r *UserRepositoryPostgres) toEntity(m *UserModel) *entity.User {
    return entity.RestoreUser(
        valueobject.MustUserID(m.ID),
        valueobject.MustEmail(m.Email),
        m.Name,
        m.CreatedAt,
    )
}
```

### Interface Layer

Entry points for external interactions.

**Contains:**
- **HTTP Handlers** - REST API endpoints
- **Middleware** - Auth, logging, metrics, request ID
- **Message Handlers** - Telegram bot, webhooks
- **CLI Commands** - Admin utilities

```go
// interface/http/handler/user_handler.go
package handler

import (
    "encoding/json"
    "net/http"
    "project/application/dto"
    "project/application/usecase"
)

type UserHandler struct {
    createUser *usecase.CreateUserUseCase
}

func NewUserHandler(createUser *usecase.CreateUserUseCase) *UserHandler {
    return &UserHandler{createUser: createUser}
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var input dto.CreateUserInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    output, err := h.createUser.Execute(r.Context(), input)
    if err != nil {
        switch err {
        case dto.ErrInvalidEmail:
            http.Error(w, err.Error(), http.StatusBadRequest)
        case dto.ErrEmailAlreadyExists:
            http.Error(w, err.Error(), http.StatusConflict)
        default:
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(output)
}
```

## Testing Strategy by Layer

| Layer | Test Type | Mocking | Coverage Target |
|-------|-----------|---------|-----------------|
| Domain | Unit | None | 95% |
| Application | Unit | Repository, ports | 80% |
| Infrastructure | Integration | External services (testcontainers) | 70% |
| Interface | Integration | Use cases | 70% |

## Dependency Injection

Use constructor injection and wire dependencies in `cmd/server/main.go`:

```go
func main() {
    // Infrastructure
    db := setupDatabase()
    logger := setupLogger()

    // Repositories
    userRepo := persistence.NewUserRepositoryPostgres(db)

    // Adapters
    emailSender := smtp.NewEmailSender(config.SMTP)

    // Use Cases
    createUser := usecase.NewCreateUserUseCase(userRepo, emailSender, logger)

    // Handlers
    userHandler := handler.NewUserHandler(createUser)

    // Router
    mux := http.NewServeMux()
    mux.HandleFunc("POST /users", userHandler.Create)

    http.ListenAndServe(":8080", mux)
}
```

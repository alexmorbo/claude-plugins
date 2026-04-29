---
name: go-error-handling
description: "Enforces proper error handling patterns in Go services. Use when creating error types, handling errors, wrapping errors with context, or implementing error boundaries between layers. Applies to any Go code that returns or processes errors."
---

# Go Error Handling Skill

This skill ensures consistent error handling across Go microservices based on `${CLAUDE_PLUGIN_ROOT}/references/error-handling.md`.

## Core Principles

1. **Always wrap errors with context**
2. **Use custom error types at boundaries**
3. **Preserve error chain for debugging**
4. **Map errors to appropriate HTTP status codes**

## Error Wrapping

### Always Add Context

```go
// BAD - No context, hard to debug
if err := repo.Save(ctx, user); err != nil {
    return err
}

// GOOD - Context added
if err := repo.Save(ctx, user); err != nil {
    return fmt.Errorf("save user %s: %w", user.ID(), err)
}
```

### Preserve Error Chain

```go
// BAD - Breaks error chain
return errors.New("failed to save user")

// GOOD - Preserves cause
return fmt.Errorf("save user: %w", err)
```

## Custom Error Types

### Domain Errors

```go
// domain/errors.go
package domain

import "errors"

var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
    ErrInvalidInput  = errors.New("invalid input")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrForbidden     = errors.New("forbidden")
)

// Typed error with context
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

func NewValidationError(field, message string) error {
    return &ValidationError{Field: field, Message: message}
}
```

### Application Errors

```go
// application/errors.go
package application

type UseCaseError struct {
    Code    string
    Message string
    Cause   error
}

func (e *UseCaseError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("%s: %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

func (e *UseCaseError) Unwrap() error {
    return e.Cause
}

// Factory functions
func ErrUserNotFound(id string) error {
    return &UseCaseError{
        Code:    "USER_NOT_FOUND",
        Message: fmt.Sprintf("user with id %s not found", id),
        Cause:   domain.ErrNotFound,
    }
}

func ErrEmailAlreadyExists(email string) error {
    return &UseCaseError{
        Code:    "EMAIL_EXISTS",
        Message: fmt.Sprintf("email %s already registered", email),
        Cause:   domain.ErrAlreadyExists,
    }
}
```

## Error Checking

### Use errors.Is for Sentinel Errors

```go
// BAD
if err == domain.ErrNotFound {

// GOOD
if errors.Is(err, domain.ErrNotFound) {
```

### Use errors.As for Typed Errors

```go
// BAD
if ve, ok := err.(*ValidationError); ok {

// GOOD
var ve *ValidationError
if errors.As(err, &ve) {
    log.Printf("Field: %s, Message: %s", ve.Field, ve.Message)
}
```

## HTTP Error Mapping

```go
// interface/http/errors.go
package http

import (
    "errors"
    "net/http"

    "service/domain"
)

func MapErrorToStatus(err error) int {
    switch {
    case errors.Is(err, domain.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, domain.ErrAlreadyExists):
        return http.StatusConflict
    case errors.Is(err, domain.ErrInvalidInput):
        return http.StatusBadRequest
    case errors.Is(err, domain.ErrUnauthorized):
        return http.StatusUnauthorized
    case errors.Is(err, domain.ErrForbidden):
        return http.StatusForbidden
    default:
        return http.StatusInternalServerError
    }
}

// Error response structure
type ErrorResponse struct {
    Error   string `json:"error"`
    Code    string `json:"code,omitempty"`
    Details any    `json:"details,omitempty"`
}

func HandleError(c *gin.Context, err error) {
    status := MapErrorToStatus(err)

    var useCaseErr *application.UseCaseError
    if errors.As(err, &useCaseErr) {
        c.JSON(status, ErrorResponse{
            Error: useCaseErr.Message,
            Code:  useCaseErr.Code,
        })
        return
    }

    var validationErr *domain.ValidationError
    if errors.As(err, &validationErr) {
        c.JSON(status, ErrorResponse{
            Error: validationErr.Error(),
            Code:  "VALIDATION_ERROR",
            Details: map[string]string{
                "field": validationErr.Field,
            },
        })
        return
    }

    // Generic error (don't leak internal details)
    c.JSON(status, ErrorResponse{
        Error: "internal server error",
        Code:  "INTERNAL_ERROR",
    })
}
```

## Layer Boundaries

### Infrastructure → Application

```go
// infrastructure/persistence/user_repository.go
func (r *UserRepositoryPostgres) FindByID(ctx context.Context, id valueobject.UserID) (*entity.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).First(&model, "id = ?", id.String()).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound  // Translate to domain error
        }
        return nil, fmt.Errorf("query user by id: %w", err)
    }
    return model.ToEntity()
}
```

### Application → Interface

```go
// application/usecase/get_user.go
func (uc *GetUserUseCase) Execute(ctx context.Context, id string) (*dto.UserOutput, error) {
    userID, err := valueobject.NewUserID(id)
    if err != nil {
        return nil, fmt.Errorf("invalid user id: %w", domain.ErrInvalidInput)
    }

    user, err := uc.userRepo.FindByID(ctx, userID)
    if err != nil {
        if errors.Is(err, domain.ErrNotFound) {
            return nil, ErrUserNotFound(id)  // Wrap with application error
        }
        return nil, fmt.Errorf("find user: %w", err)
    }

    return dto.NewUserOutput(user), nil
}
```

## Logging Errors

```go
// Always log with context before returning
func (uc *CreateUserUseCase) Execute(ctx context.Context, input dto.CreateUserInput) (*dto.CreateUserOutput, error) {
    // ... business logic ...

    if err := uc.userRepo.Save(ctx, user); err != nil {
        slog.ErrorContext(ctx, "Failed to save user",
            slog.Group("application",
                slog.String("use_case", "CreateUser"),
                slog.String("user_email", input.Email),
            ),
            slog.String("error", err.Error()),
        )
        return nil, fmt.Errorf("save user: %w", err)
    }

    // ...
}
```

## Validation Checklist

Before completing implementation:

- [ ] All errors wrapped with `fmt.Errorf("context: %w", err)`
- [ ] Sentinel errors checked with `errors.Is()`
- [ ] Typed errors checked with `errors.As()`
- [ ] Domain errors don't leak infrastructure details
- [ ] HTTP handlers map errors to appropriate status codes
- [ ] Errors logged before returning from use cases
- [ ] No naked `return err` without context

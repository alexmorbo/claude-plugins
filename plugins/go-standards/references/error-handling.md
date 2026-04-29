# Error Handling

## Philosophy

- Errors are values, not exceptions
- Handle errors explicitly at every level
- Provide context when wrapping errors
- Use sentinel errors for expected conditions
- Use custom error types when additional context is needed

## Error Types

### Sentinel Errors

Predefined errors for expected conditions. Declare in domain layer.

```go
// domain/entity/errors.go
package entity

import "errors"

var (
    ErrUserNotFound      = errors.New("user not found")
    ErrInvalidEmail      = errors.New("invalid email format")
    ErrEmptyFirstName    = errors.New("first name cannot be empty")
    ErrOrderNotDraft     = errors.New("order is not in draft status")
    ErrInsufficientFunds = errors.New("insufficient funds")
)
```

```go
// application/dto/errors.go
package dto

import "errors"

var (
    ErrEmailAlreadyExists = errors.New("email already exists")
    ErrUnauthorized       = errors.New("unauthorized")
    ErrForbidden          = errors.New("forbidden")
    ErrValidation         = errors.New("validation error")
)
```

### Custom Error Types

When you need additional context or fields.

```go
// domain/entity/errors.go
package entity

import "fmt"

// ValidationError contains field-level validation details
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

func NewValidationError(field, message string) ValidationError {
    return ValidationError{Field: field, Message: message}
}

// NotFoundError with entity type
type NotFoundError struct {
    Entity string
    ID     string
}

func (e NotFoundError) Error() string {
    return fmt.Sprintf("%s with id %s not found", e.Entity, e.ID)
}

func NewNotFoundError(entity, id string) NotFoundError {
    return NotFoundError{Entity: entity, ID: id}
}
```

### Domain Errors Interface

```go
// domain/entity/errors.go
package entity

// DomainError marker interface for domain-specific errors
type DomainError interface {
    error
    IsDomainError() bool
}

type domainError struct {
    message string
    code    string
}

func (e domainError) Error() string      { return e.message }
func (e domainError) IsDomainError() bool { return true }
func (e domainError) Code() string       { return e.code }

func NewDomainError(code, message string) DomainError {
    return domainError{code: code, message: message}
}

// Predefined domain errors
var (
    ErrUserNotActive = NewDomainError("USER_NOT_ACTIVE", "user account is not active")
    ErrOrderExpired  = NewDomainError("ORDER_EXPIRED", "order has expired")
)
```

## Error Wrapping

### Standard Pattern

Always add context when propagating errors.

```go
import "fmt"

// Use %w to wrap (allows errors.Is/As)
func (r *UserRepository) FindByID(ctx context.Context, id UserID) (*User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).First(&model, "id = ?", id.Value()).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("find user by id %s: %w", id.Value(), err)
    }
    return r.toEntity(&model), nil
}
```

### Context Guidelines

```go
// BAD - no context
return err

// BAD - loses original error
return errors.New("database error")

// BAD - too verbose
return fmt.Errorf("failed to execute database query to find user by id in users table: %w", err)

// GOOD - concise context
return fmt.Errorf("find user %s: %w", id, err)
return fmt.Errorf("save order: %w", err)
return fmt.Errorf("send email to %s: %w", email, err)
```

### Wrapping in Layers

```go
// Repository (infrastructure)
func (r *UserRepo) Save(ctx context.Context, user *User) error {
    if err := r.db.Save(user).Error; err != nil {
        return fmt.Errorf("save user: %w", err)
    }
    return nil
}

// Use case (application)
func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*User, error) {
    user, err := entity.NewUser(input.Email, input.Name)
    if err != nil {
        return nil, err // Don't wrap domain errors
    }

    if err := uc.repo.Save(ctx, user); err != nil {
        return nil, fmt.Errorf("create user: %w", err)
    }
    return user, nil
}

// Handler (interface)
func (h *UserHandler) Create(c *gin.Context) {
    output, err := h.createUser.Execute(c.Request.Context(), input)
    if err != nil {
        h.handleError(c, err) // Map to HTTP response
        return
    }
    c.JSON(http.StatusCreated, output)
}
```

## Error Checking

### errors.Is

Check if error is or wraps a specific sentinel error.

```go
if errors.Is(err, ErrUserNotFound) {
    // Handle not found
}

if errors.Is(err, gorm.ErrRecordNotFound) {
    return nil, ErrUserNotFound
}

if errors.Is(err, context.DeadlineExceeded) {
    // Handle timeout
}
```

### errors.As

Extract custom error type from error chain.

```go
var validationErr ValidationError
if errors.As(err, &validationErr) {
    log.Printf("Validation failed on field %s: %s", validationErr.Field, validationErr.Message)
}

var notFoundErr NotFoundError
if errors.As(err, &notFoundErr) {
    log.Printf("%s not found: %s", notFoundErr.Entity, notFoundErr.ID)
}
```

## HTTP Error Mapping

### Error Handler

```go
// interface/http/handler/error_handler.go
package handler

import (
    "errors"
    "net/http"

    "github.com/gin-gonic/gin"
    "service/application/dto"
    "service/domain/entity"
)

type ErrorResponse struct {
    Error   string `json:"error"`
    Code    string `json:"code,omitempty"`
    Details any    `json:"details,omitempty"`
}

func HandleError(c *gin.Context, err error) {
    // Domain errors
    if errors.Is(err, entity.ErrUserNotFound) {
        c.JSON(http.StatusNotFound, ErrorResponse{
            Error: "User not found",
            Code:  "USER_NOT_FOUND",
        })
        return
    }

    if errors.Is(err, entity.ErrInvalidEmail) {
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Error: "Invalid email format",
            Code:  "INVALID_EMAIL",
        })
        return
    }

    // Application errors
    if errors.Is(err, dto.ErrEmailAlreadyExists) {
        c.JSON(http.StatusConflict, ErrorResponse{
            Error: "Email already registered",
            Code:  "EMAIL_EXISTS",
        })
        return
    }

    if errors.Is(err, dto.ErrUnauthorized) {
        c.JSON(http.StatusUnauthorized, ErrorResponse{
            Error: "Unauthorized",
            Code:  "UNAUTHORIZED",
        })
        return
    }

    if errors.Is(err, dto.ErrForbidden) {
        c.JSON(http.StatusForbidden, ErrorResponse{
            Error: "Forbidden",
            Code:  "FORBIDDEN",
        })
        return
    }

    // Validation errors
    var validationErr entity.ValidationError
    if errors.As(err, &validationErr) {
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Error: "Validation failed",
            Code:  "VALIDATION_ERROR",
            Details: map[string]string{
                validationErr.Field: validationErr.Message,
            },
        })
        return
    }

    // Default: internal error (don't expose details)
    slog.ErrorContext(c.Request.Context(), "Internal error", "error", err)
    c.JSON(http.StatusInternalServerError, ErrorResponse{
        Error: "Internal server error",
        Code:  "INTERNAL_ERROR",
    })
}
```

### Using in Handlers

```go
func (h *UserHandler) Create(c *gin.Context) {
    var input dto.CreateUserInput
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Error: "Invalid request body",
            Code:  "INVALID_REQUEST",
        })
        return
    }

    output, err := h.createUser.Execute(c.Request.Context(), input)
    if err != nil {
        HandleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, output)
}
```

## Error Logging

### When to Log

```go
// DON'T log at every level - creates duplicate logs
func (r *UserRepo) Save(ctx context.Context, user *User) error {
    if err := r.db.Save(user).Error; err != nil {
        log.Error("failed to save user", "error", err) // DON'T
        return fmt.Errorf("save user: %w", err)
    }
    return nil
}

// DO log at the boundary (handler level)
func (h *UserHandler) Create(c *gin.Context) {
    output, err := h.createUser.Execute(c.Request.Context(), input)
    if err != nil {
        // Log unexpected errors
        if !isExpectedError(err) {
            slog.ErrorContext(c.Request.Context(), "Create user failed", "error", err)
        }
        HandleError(c, err)
        return
    }
    c.JSON(http.StatusCreated, output)
}

func isExpectedError(err error) bool {
    return errors.Is(err, entity.ErrUserNotFound) ||
           errors.Is(err, entity.ErrInvalidEmail) ||
           errors.Is(err, dto.ErrEmailAlreadyExists)
}
```

## Panic Recovery

Only use panic for truly unrecoverable situations. Gin's Recovery middleware handles panics.

```go
// In main.go
router := gin.New()
router.Use(gin.Recovery()) // Catches panics

// In domain - panic only for programming errors
func MustEmail(value string) Email {
    email, err := NewEmail(value)
    if err != nil {
        panic(fmt.Sprintf("invalid email: %s", value))
    }
    return email
}

// Never panic in normal business logic
func (u *User) UpdateEmail(email Email) error {
    // return error, don't panic
    if email.IsEmpty() {
        return ErrInvalidEmail
    }
    u.email = email
    return nil
}
```

## Testing Errors

```go
func TestCreateUserUseCase_DuplicateEmail(t *testing.T) {
    mockRepo := new(MockUserRepository)
    mockRepo.On("FindByEmail", mock.Anything, mock.Anything).
        Return(&entity.User{}, nil) // User exists

    uc := NewCreateUserUseCase(mockRepo)
    _, err := uc.Execute(context.Background(), dto.CreateUserInput{
        Email: "existing@example.com",
    })

    assert.ErrorIs(t, err, dto.ErrEmailAlreadyExists)
}

func TestCreateUserUseCase_ValidationError(t *testing.T) {
    uc := NewCreateUserUseCase(nil)
    _, err := uc.Execute(context.Background(), dto.CreateUserInput{
        Email: "invalid-email",
    })

    var validationErr entity.ValidationError
    assert.True(t, errors.As(err, &validationErr))
    assert.Equal(t, "email", validationErr.Field)
}
```

## Summary

| Situation | Pattern |
|-----------|---------|
| Expected conditions | Sentinel errors (`var ErrNotFound = errors.New(...)`) |
| Need extra context | Custom error types |
| Propagating errors | Wrap with `fmt.Errorf("context: %w", err)` |
| Checking error type | `errors.Is(err, target)` |
| Extracting error data | `errors.As(err, &target)` |
| HTTP responses | Map to appropriate status codes |
| Logging | Only at boundaries, not every level |

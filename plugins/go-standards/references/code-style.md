# Go Code Style Guide

## Comments

### Core Rule
**Comments should only appear where the code is not self-explanatory.** Do not comment every line or function.

### When to Comment

**DO comment:**
- Complex algorithms or non-obvious logic
- Performance-critical sections with specific optimizations
- Workarounds for bugs or limitations in external libraries
- Public API documentation (Godoc format)
- TODO/FIXME items with context

**DO NOT comment:**
- Obvious operations (`i++` doesn't need "increment i")
- Self-descriptive function/variable names
- Standard patterns (repository methods, HTTP handlers)
- Every struct field

### Examples

```go
// BAD - over-commenting
// GetUserByID retrieves a user from the database by their ID
// Parameters:
//   - ctx: context for cancellation
//   - id: the user ID to look up
// Returns:
//   - *User: the found user
//   - error: any error that occurred
func (r *UserRepository) GetUserByID(ctx context.Context, id UserID) (*User, error) {
    // Create a new user variable to store the result
    var user User
    // Execute the database query
    result := r.db.WithContext(ctx).First(&user, id)
    // Check if there was an error
    if result.Error != nil {
        // Return nil and the error
        return nil, result.Error
    }
    // Return the user and nil error
    return &user, nil
}

// GOOD - minimal, self-documenting code
func (r *UserRepository) GetUserByID(ctx context.Context, id UserID) (*User, error) {
    var user User
    if err := r.db.WithContext(ctx).First(&user, id).Error; err != nil {
        return nil, err
    }
    return &user, nil
}
```

### Godoc for Public APIs

```go
// UserService handles user-related business operations.
// It coordinates between the repository layer and external services.
type UserService struct {
    repo   UserRepository
    logger *slog.Logger
}

// CreateUser registers a new user with the given profile data.
// Returns ErrDuplicateEmail if email already exists.
func (s *UserService) CreateUser(ctx context.Context, input CreateUserInput) (*User, error) {
    // implementation
}
```

## Naming Conventions

### Packages
- Short, lowercase, single-word names
- Avoid generic names like `util`, `common`, `helpers`
- Use domain-specific names: `user`, `order`, `payment`

### Variables
- Short names for short scopes: `i`, `ctx`, `err`
- Descriptive names for wider scopes: `userRepository`, `httpClient`
- Avoid stuttering: `user.Name` not `user.UserName`

### Functions
- Verb-noun pattern: `CreateUser`, `GetOrder`, `ValidateInput`
- Boolean functions: `Is*`, `Has*`, `Can*` - `IsValid`, `HasPermission`
- Constructors: `New*` - `NewUserService`, `NewHTTPClient`

### Interfaces
- Single-method interfaces: method name + `-er` suffix: `Reader`, `Writer`, `Closer`
- Multi-method interfaces: noun describing behavior: `Repository`, `Service`
- Avoid `I` prefix: `UserRepository` not `IUserRepository`

### Constants
- Use `camelCase` for unexported, `PascalCase` for exported
- Group related constants with `const` block
- Use `iota` for enums

```go
const (
    StatusPending Status = iota
    StatusActive
    StatusInactive
)
```

## Error Handling

### Error Types
- Use sentinel errors for expected conditions: `var ErrNotFound = errors.New("not found")`
- Use custom error types for errors needing context
- Wrap errors with `fmt.Errorf("context: %w", err)`

### Error Messages
- Start with lowercase (will be concatenated)
- Include relevant context
- No punctuation at end

```go
// GOOD
return fmt.Errorf("user %d not found in chat %d: %w", userID, chatID, err)

// BAD
return fmt.Errorf("User not found.") // uppercase, punctuation, no context
```

## Formatting

### Tool Enforcement
- `gofmt` / `goimports` for formatting
- `golangci-lint` for linting

### Line Length
- Soft limit: 100 characters
- Hard limit: 120 characters
- Break long function signatures across multiple lines

### Blank Lines
- One blank line between functions
- No blank lines inside short functions
- Blank lines to separate logical blocks in longer functions

## Struct Tags

Use consistent tag ordering:
```go
type User struct {
    ID        int64     `json:"id" gorm:"primaryKey"`
    Email     string    `json:"email" gorm:"uniqueIndex"`
    CreatedAt time.Time `json:"created_at" gorm:"autoCreateTime"`
}
```

## Testing

### Test Names
- `Test<Function>_<Scenario>_<ExpectedBehavior>`
- Use table-driven tests for multiple cases

```go
func TestUserService_CreateUser_WithDuplicateEmail_ReturnsError(t *testing.T) {
    // ...
}

func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        input   CreateUserInput
        want    *User
        wantErr error
    }{
        {
            name:  "valid input creates user",
            input: CreateUserInput{Email: "test@example.com"},
            want:  &User{Email: "test@example.com"},
        },
        {
            name:    "empty email returns validation error",
            input:   CreateUserInput{Email: ""},
            wantErr: ErrInvalidEmail,
        },
    }
    // ...
}
```

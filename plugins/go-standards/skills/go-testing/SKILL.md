---
name: go-testing
description: "Enforces testing standards for Go microservices. Use when writing tests, reviewing test coverage, creating mocks, or implementing test patterns. Applies to unit tests, integration tests, and any test-related code in Go services."
---

# Go Testing Skill

This skill ensures comprehensive testing based on `${CLAUDE_PLUGIN_ROOT}/references/testing.md`.

## Coverage Requirements

| Layer | Minimum Coverage |
|-------|-----------------|
| Domain | **95%** |
| Application | **80%** |
| Infrastructure | **70%** |
| Interface | **70%** |
| **Overall** | **80%** |

## Test Organization

```
service/
├── domain/
│   ├── entity/
│   │   ├── user.go
│   │   └── user_test.go              # Unit tests alongside code
│   └── valueobject/
│       ├── email.go
│       └── email_test.go
├── application/
│   └── usecase/
│       ├── create_user.go
│       └── create_user_test.go       # Unit tests with mocks
├── infrastructure/
│   └── persistence/
│       ├── user_repository.go
│       └── user_repository_test.go   # Unit tests (mocked DB)
├── interface/
│   └── http/handler/
│       ├── user_handler.go
│       └── user_handler_test.go      # HTTP handler tests
└── tests/                             # Integration & E2E only
    ├── integration/
    │   └── user_repository_integration_test.go
    └── e2e/
        └── api_test.go
```

## Test Patterns

### Table-Driven Tests

```go
func TestNewEmail(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid email", "user@example.com", false},
        {"valid with subdomain", "user@mail.example.com", false},
        {"invalid - no @", "userexample.com", true},
        {"invalid - no domain", "user@", true},
        {"empty", "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            email, err := NewEmail(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
                assert.Nil(t, email)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.input, email.Value())
            }
        })
    }
}
```

### Domain Layer Tests (95% coverage)

Pure unit tests, no mocks needed:

```go
// domain/entity/user_test.go
func TestUser_UpdateProfile(t *testing.T) {
    tests := []struct {
        name      string
        user      *User
        newName   string
        wantErr   bool
    }{
        {
            name:    "valid update",
            user:    mustCreateUser(t, "test@example.com", "John"),
            newName: "Jane",
            wantErr: false,
        },
        {
            name:    "empty name rejected",
            user:    mustCreateUser(t, "test@example.com", "John"),
            newName: "",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := tt.user.UpdateName(tt.newName)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.newName, tt.user.Name())
            }
        })
    }
}

func mustCreateUser(t *testing.T, email, name string) *User {
    t.Helper()
    e, _ := valueobject.NewEmail(email)
    u, err := NewUser(e, name)
    require.NoError(t, err)
    return u
}
```

### Application Layer Tests (80% coverage)

Use mocks for dependencies:

```go
// application/usecase/create_user_test.go
type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Save(ctx context.Context, user *entity.User) error {
    args := m.Called(ctx, user)
    return args.Error(0)
}

func (m *MockUserRepository) FindByEmail(ctx context.Context, email valueobject.Email) (*entity.User, error) {
    args := m.Called(ctx, email)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*entity.User), args.Error(1)
}

func TestCreateUserUseCase_Execute(t *testing.T) {
    tests := []struct {
        name      string
        input     dto.CreateUserInput
        setupMock func(*MockUserRepository)
        wantErr   error
    }{
        {
            name: "success",
            input: dto.CreateUserInput{
                Email: "new@example.com",
                Name:  "John",
            },
            setupMock: func(m *MockUserRepository) {
                m.On("FindByEmail", mock.Anything, mock.Anything).Return(nil, nil)
                m.On("Save", mock.Anything, mock.Anything).Return(nil)
            },
            wantErr: nil,
        },
        {
            name: "email already exists",
            input: dto.CreateUserInput{
                Email: "existing@example.com",
                Name:  "John",
            },
            setupMock: func(m *MockUserRepository) {
                existingUser := &entity.User{} // simplified
                m.On("FindByEmail", mock.Anything, mock.Anything).Return(existingUser, nil)
            },
            wantErr: ErrEmailAlreadyExists,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockRepo := new(MockUserRepository)
            tt.setupMock(mockRepo)

            uc := NewCreateUserUseCase(mockRepo)
            _, err := uc.Execute(context.Background(), tt.input)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
            } else {
                assert.NoError(t, err)
            }

            mockRepo.AssertExpectations(t)
        })
    }
}
```

### Interface Layer Tests (70% coverage)

HTTP handler tests:

```go
// interface/http/handler/user_handler_test.go
func TestUserHandler_Create(t *testing.T) {
    tests := []struct {
        name           string
        body           map[string]any
        setupMock      func(*MockCreateUserUseCase)
        expectedStatus int
    }{
        {
            name: "success",
            body: map[string]any{
                "email": "test@example.com",
                "name":  "John",
            },
            setupMock: func(m *MockCreateUserUseCase) {
                m.On("Execute", mock.Anything, mock.Anything).Return(&dto.CreateUserOutput{
                    ID:    "user-123",
                    Email: "test@example.com",
                }, nil)
            },
            expectedStatus: http.StatusCreated,
        },
        {
            name: "invalid email",
            body: map[string]any{
                "email": "invalid",
                "name":  "John",
            },
            setupMock: func(m *MockCreateUserUseCase) {
                m.On("Execute", mock.Anything, mock.Anything).Return(nil, domain.ErrInvalidInput)
            },
            expectedStatus: http.StatusBadRequest,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            gin.SetMode(gin.TestMode)

            mockUC := new(MockCreateUserUseCase)
            tt.setupMock(mockUC)

            handler := NewUserHandler(mockUC)
            router := gin.New()
            router.POST("/users", handler.Create)

            body, _ := json.Marshal(tt.body)
            req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewReader(body))
            req.Header.Set("Content-Type", "application/json")

            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            assert.Equal(t, tt.expectedStatus, w.Code)
            mockUC.AssertExpectations(t)
        })
    }
}
```

### Integration Tests

```go
// tests/integration/user_repository_integration_test.go
func TestUserRepositoryPostgres_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()
    container, err := NewPostgresContainer(ctx)
    require.NoError(t, err)
    defer container.Terminate(ctx)

    db, err := container.GetGormDB()
    require.NoError(t, err)

    repo := persistence.NewUserRepositoryPostgres(db)

    t.Run("save and find by id", func(t *testing.T) {
        email, _ := valueobject.NewEmail("test@example.com")
        user, _ := entity.NewUser(email, "Test")

        err := repo.Save(ctx, user)
        require.NoError(t, err)

        found, err := repo.FindByID(ctx, user.ID())
        require.NoError(t, err)
        assert.Equal(t, user.Email().Value(), found.Email().Value())
    })
}
```

## Running Tests

```bash
# All unit tests
go test ./...

# With coverage
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out

# Skip integration tests
go test ./... -short

# Specific layer
go test ./domain/... -cover
go test ./application/... -cover

# Verbose with race detection
go test ./... -v -race
```

## Test Naming Convention

```
Test<Type>_<Method>_<Scenario>
```

Examples:
- `TestUser_UpdateProfile_ValidData`
- `TestCreateUserUseCase_Execute_DuplicateEmail`
- `TestUserHandler_Create_InvalidJSON`

## Mocking Guidelines

1. **Mock at boundaries** - Mock repositories, external services
2. **Don't mock domain** - Domain should be testable without mocks
3. **Use testify/mock** - Standard mocking library
4. **Assert expectations** - Always call `mockObj.AssertExpectations(t)`

## Verification Before Completion

```bash
# 1. Run all tests
go test ./... -v

# 2. Check coverage meets threshold
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out | tail -1
# Must show >= 80%

# 3. Check domain coverage specifically
go test ./domain/... -cover
# Must show >= 95%
```

## Checklist

- [ ] All new code has corresponding tests
- [ ] Table-driven tests used for multiple scenarios
- [ ] Mocks only at layer boundaries
- [ ] Test names follow convention
- [ ] Coverage meets layer requirements
- [ ] Integration tests skip in short mode
- [ ] No flaky tests (race conditions, timing)

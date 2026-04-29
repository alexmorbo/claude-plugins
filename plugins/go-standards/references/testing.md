# Testing Strategy

## Overview

We use a **hybrid approach** for test organization:
- **Unit tests** - alongside the code (`*_test.go`)
- **Integration/E2E tests** - in `tests/` directory

This follows Go conventions while supporting Clean Architecture separation.

## Test Organization

```
service/
├── domain/
│   ├── entity/
│   │   ├── user.go
│   │   └── user_test.go              # Unit tests
│   └── valueobject/
│       ├── email.go
│       └── email_test.go             # Unit tests
│
├── application/
│   └── usecase/
│       ├── create_user.go
│       └── create_user_test.go       # Unit tests with mocks
│
├── infrastructure/
│   └── persistence/
│       ├── user_repository.go
│       └── user_repository_test.go   # Unit tests (mocked DB)
│
├── interface/
│   └── http/
│       └── handler/
│           ├── user_handler.go
│           └── user_handler_test.go  # HTTP handler tests
│
└── tests/                             # Integration & E2E only
    ├── integration/
    │   ├── user_repository_integration_test.go
    │   └── testcontainers.go          # Shared test containers
    └── e2e/
        ├── api_test.go
        └── helpers.go
```

## Test Types by Layer

### Domain Layer - Unit Tests (95% coverage)

Pure business logic tests. No mocks needed - domain has no dependencies.

```go
// domain/entity/user_test.go
package entity

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestNewUser(t *testing.T) {
    tests := []struct {
        name      string
        email     string
        firstName string
        wantErr   bool
    }{
        {
            name:      "valid user",
            email:     "test@example.com",
            firstName: "John",
            wantErr:   false,
        },
        {
            name:      "empty first name",
            email:     "test@example.com",
            firstName: "",
            wantErr:   true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            email, _ := valueobject.NewEmail(tt.email)
            user, err := NewUser(email, tt.firstName)

            if tt.wantErr {
                assert.Error(t, err)
                assert.Nil(t, user)
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.email, user.Email().Value())
            }
        })
    }
}
```

```go
// domain/valueobject/email_test.go
package valueobject

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

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
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.input, email.Value())
            }
        })
    }
}
```

### Application Layer - Unit Tests with Mocks (80% coverage)

Test use cases with mocked repositories and ports.

```go
// application/usecase/create_user_test.go
package usecase

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"

    "service/application/dto"
    "service/domain/entity"
    "service/domain/valueobject"
)

// Mock repository
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
                Email:     "new@example.com",
                FirstName: "John",
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
                Email:     "existing@example.com",
                FirstName: "John",
            },
            setupMock: func(m *MockUserRepository) {
                existingUser, _ := entity.NewUser(
                    valueobject.MustEmail("existing@example.com"),
                    "Existing",
                )
                m.On("FindByEmail", mock.Anything, mock.Anything).Return(existingUser, nil)
            },
            wantErr: dto.ErrEmailAlreadyExists,
        },
        {
            name: "invalid email",
            input: dto.CreateUserInput{
                Email:     "invalid-email",
                FirstName: "John",
            },
            setupMock: func(m *MockUserRepository) {},
            wantErr:   dto.ErrInvalidEmail,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockRepo := new(MockUserRepository)
            tt.setupMock(mockRepo)

            uc := NewCreateUserUseCase(mockRepo, nil)
            output, err := uc.Execute(context.Background(), tt.input)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
                assert.Nil(t, output)
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.input.Email, output.Email)
            }

            mockRepo.AssertExpectations(t)
        })
    }
}
```

### Infrastructure Layer - Unit + Integration Tests (70% coverage)

#### Unit tests (mocked)
```go
// infrastructure/persistence/user_repository_test.go
package persistence

import (
    "context"
    "testing"

    "github.com/DATA-DOG/go-sqlmock"
    "github.com/stretchr/testify/assert"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func setupMockDB(t *testing.T) (*gorm.DB, sqlmock.Sqlmock) {
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatalf("failed to create sqlmock: %v", err)
    }

    gormDB, err := gorm.Open(postgres.New(postgres.Config{
        Conn: db,
    }), &gorm.Config{})
    if err != nil {
        t.Fatalf("failed to create gorm db: %v", err)
    }

    return gormDB, mock
}

func TestUserRepositoryPostgres_FindByID(t *testing.T) {
    db, mock := setupMockDB(t)
    repo := NewUserRepositoryPostgres(db)

    rows := sqlmock.NewRows([]string{"id", "email", "first_name", "created_at"}).
        AddRow("user-123", "test@example.com", "John", time.Now())

    mock.ExpectQuery(`SELECT \* FROM "users"`).
        WithArgs("user-123").
        WillReturnRows(rows)

    user, err := repo.FindByID(context.Background(), valueobject.MustUserID("user-123"))

    assert.NoError(t, err)
    assert.Equal(t, "test@example.com", user.Email().Value())
}
```

#### Integration tests (real DB via testcontainers)
```go
// tests/integration/user_repository_integration_test.go
package integration

import (
    "context"
    "testing"

    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/suite"
)

type UserRepositoryIntegrationSuite struct {
    suite.Suite
    container *PostgresContainer
    repo      repository.UserRepository
}

func (s *UserRepositoryIntegrationSuite) SetupSuite() {
    ctx := context.Background()
    container, err := NewPostgresContainer(ctx)
    require.NoError(s.T(), err)
    s.container = container

    db, err := container.GetGormDB()
    require.NoError(s.T(), err)

    s.repo = persistence.NewUserRepositoryPostgres(db)
}

func (s *UserRepositoryIntegrationSuite) TearDownSuite() {
    s.container.Terminate(context.Background())
}

func (s *UserRepositoryIntegrationSuite) TestSaveAndFindByID() {
    ctx := context.Background()

    email, _ := valueobject.NewEmail("integration@test.com")
    user, _ := entity.NewUser(email, "Integration")

    err := s.repo.Save(ctx, user)
    s.Require().NoError(err)

    found, err := s.repo.FindByID(ctx, user.ID())
    s.Require().NoError(err)
    s.Equal(user.Email().Value(), found.Email().Value())
}

func TestUserRepositoryIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    suite.Run(t, new(UserRepositoryIntegrationSuite))
}
```

### Interface Layer - HTTP Handler Tests (70% coverage)

```go
// interface/http/handler/user_handler_test.go
package handler

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockCreateUserUseCase struct {
    mock.Mock
}

func (m *MockCreateUserUseCase) Execute(ctx context.Context, input dto.CreateUserInput) (*dto.CreateUserOutput, error) {
    args := m.Called(ctx, input)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*dto.CreateUserOutput), args.Error(1)
}

func setupTestRouter(handler *UserHandler) *gin.Engine {
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.POST("/users", handler.Create)
    return r
}

func TestUserHandler_Create(t *testing.T) {
    tests := []struct {
        name           string
        body           map[string]any
        setupMock      func(*MockCreateUserUseCase)
        expectedStatus int
        expectedBody   map[string]any
    }{
        {
            name: "success",
            body: map[string]any{
                "email":      "test@example.com",
                "first_name": "John",
            },
            setupMock: func(m *MockCreateUserUseCase) {
                m.On("Execute", mock.Anything, mock.Anything).Return(&dto.CreateUserOutput{
                    ID:    "user-123",
                    Email: "test@example.com",
                }, nil)
            },
            expectedStatus: http.StatusCreated,
            expectedBody: map[string]any{
                "id":    "user-123",
                "email": "test@example.com",
            },
        },
        {
            name: "invalid email",
            body: map[string]any{
                "email":      "invalid",
                "first_name": "John",
            },
            setupMock: func(m *MockCreateUserUseCase) {
                m.On("Execute", mock.Anything, mock.Anything).Return(nil, dto.ErrInvalidEmail)
            },
            expectedStatus: http.StatusBadRequest,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockUC := new(MockCreateUserUseCase)
            tt.setupMock(mockUC)

            handler := NewUserHandler(mockUC)
            router := setupTestRouter(handler)

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

### E2E Tests

```go
// tests/e2e/api_test.go
package e2e

import (
    "bytes"
    "encoding/json"
    "net/http"
    "testing"

    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/suite"
)

type APITestSuite struct {
    suite.Suite
    baseURL   string
    pgContainer *PostgresContainer
    app       *App
}

func (s *APITestSuite) SetupSuite() {
    // Start containers and app
    s.pgContainer, _ = NewPostgresContainer(context.Background())
    s.app = StartApp(s.pgContainer.ConnectionString())
    s.baseURL = s.app.URL()
}

func (s *APITestSuite) TearDownSuite() {
    s.app.Stop()
    s.pgContainer.Terminate(context.Background())
}

func (s *APITestSuite) TestCreateAndGetUser() {
    // Create user
    createBody := map[string]string{
        "email":      "e2e@test.com",
        "first_name": "E2E",
    }
    body, _ := json.Marshal(createBody)

    resp, err := http.Post(s.baseURL+"/api/v1/users", "application/json", bytes.NewReader(body))
    s.Require().NoError(err)
    s.Equal(http.StatusCreated, resp.StatusCode)

    var created map[string]string
    json.NewDecoder(resp.Body).Decode(&created)
    resp.Body.Close()

    // Get user
    resp, err = http.Get(s.baseURL + "/api/v1/users/" + created["id"])
    s.Require().NoError(err)
    s.Equal(http.StatusOK, resp.StatusCode)
}

func TestAPI(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping e2e test")
    }
    suite.Run(t, new(APITestSuite))
}
```

## Running Tests

```bash
# All unit tests
go test ./...

# With coverage
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out

# Skip integration/e2e (short mode)
go test ./... -short

# Only integration tests
go test ./tests/integration/... -v

# Only e2e tests
go test ./tests/e2e/... -v

# Specific layer
go test ./domain/... -v
go test ./application/... -v
```

## Coverage Requirements

| Layer | Minimum | Type |
|-------|---------|------|
| Domain | 95% | Unit |
| Application | 80% | Unit + mocks |
| Infrastructure | 70% | Unit + integration |
| Interface | 70% | Handler tests |
| **Overall** | **80%** | Combined |

## Test Naming Convention

```
Test<Type>_<Method>_<Scenario>_<ExpectedResult>
```

Examples:
- `TestUser_UpdateProfile_ValidData_Success`
- `TestCreateUserUseCase_Execute_DuplicateEmail_ReturnsError`
- `TestUserHandler_Create_InvalidJSON_Returns400`

## Testcontainers Setup

```go
// tests/integration/testcontainers.go
package integration

import (
    "context"
    "fmt"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

type PostgresContainer struct {
    *postgres.PostgresContainer
}

func NewPostgresContainer(ctx context.Context) (*PostgresContainer, error) {
    container, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(60*time.Second),
        ),
    )
    if err != nil {
        return nil, err
    }

    return &PostgresContainer{container}, nil
}

func (c *PostgresContainer) GetGormDB() (*gorm.DB, error) {
    connStr, err := c.ConnectionString(context.Background())
    if err != nil {
        return nil, err
    }

    return gorm.Open(postgres.Open(connStr), &gorm.Config{})
}
```

## Best Practices

1. **Table-driven tests** - Use `[]struct{}` for multiple cases
2. **Parallel tests** - Use `t.Parallel()` where safe
3. **Test isolation** - Each test should be independent
4. **Meaningful names** - Describe scenario and expected result
5. **No test logic in production** - Use build tags if needed
6. **Mock at boundaries** - Mock repositories, not internal functions

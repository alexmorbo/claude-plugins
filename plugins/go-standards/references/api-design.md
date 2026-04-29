# REST API Design

## URL Structure

### Base URL

```
https://api.service.morbo.dev/api/v1
```

### Resource Naming

- Use plural nouns: `/users`, `/orders`, `/products`
- Use kebab-case for multi-word: `/order-items`, `/user-profiles`
- Nest related resources: `/users/{id}/orders`
- Max 2 levels of nesting

```
GET    /users              # List users
POST   /users              # Create user
GET    /users/{id}         # Get user
PUT    /users/{id}         # Update user
DELETE /users/{id}         # Delete user

GET    /users/{id}/orders  # User's orders
POST   /users/{id}/orders  # Create order for user
```

## HTTP Methods

| Method | Action | Idempotent | Safe |
|--------|--------|------------|------|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Replace | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove | Yes | No |

## Request Format

### Headers

```
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>
X-Request-ID: <uuid>
```

### Create/Update Body

```json
{
  "email": "user@example.com",
  "first_name": "John",
  "last_name": "Doe"
}
```

### Query Parameters

```
GET /users?page=1&per_page=20&sort=created_at&order=desc&status=active
```

| Parameter | Description |
|-----------|-------------|
| `page` | Page number (1-indexed) |
| `per_page` | Items per page (default: 20, max: 100) |
| `sort` | Sort field |
| `order` | Sort order: `asc` or `desc` |
| `filter` | Filter fields (e.g., `status=active`) |

## Response Format

### Success Response

```json
{
  "data": {
    "id": "user-123",
    "email": "user@example.com",
    "first_name": "John",
    "created_at": "2025-01-16T10:30:00Z"
  }
}
```

### List Response with Pagination

```json
{
  "data": [
    {"id": "user-1", "email": "user1@example.com"},
    {"id": "user-2", "email": "user2@example.com"}
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": "invalid email format",
      "first_name": "cannot be empty"
    }
  }
}
```

## HTTP Status Codes

### Success

| Code | Usage |
|------|-------|
| 200 | GET, PUT, PATCH success |
| 201 | POST success (resource created) |
| 204 | DELETE success (no content) |

### Client Errors

| Code | Usage |
|------|-------|
| 400 | Bad request, validation error |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (not authorized) |
| 404 | Resource not found |
| 409 | Conflict (e.g., duplicate email) |
| 422 | Unprocessable entity |
| 429 | Too many requests |

### Server Errors

| Code | Usage |
|------|-------|
| 500 | Internal server error |
| 502 | Bad gateway |
| 503 | Service unavailable |

## Implementation

### Response DTOs

```go
// interface/http/response/response.go
package response

type Response struct {
    Data any `json:"data,omitempty"`
    Meta any `json:"meta,omitempty"`
}

type ErrorResponse struct {
    Error ErrorBody `json:"error"`
}

type ErrorBody struct {
    Code    string         `json:"code"`
    Message string         `json:"message"`
    Details map[string]any `json:"details,omitempty"`
}

type PaginationMeta struct {
    Page       int `json:"page"`
    PerPage    int `json:"per_page"`
    Total      int `json:"total"`
    TotalPages int `json:"total_pages"`
}

func NewPaginationMeta(page, perPage, total int) PaginationMeta {
    totalPages := total / perPage
    if total%perPage > 0 {
        totalPages++
    }
    return PaginationMeta{
        Page:       page,
        PerPage:    perPage,
        Total:      total,
        TotalPages: totalPages,
    }
}
```

### Request DTOs with Validation

```go
// interface/http/request/user_request.go
package request

type CreateUserRequest struct {
    Email     string `json:"email" binding:"required,email"`
    FirstName string `json:"first_name" binding:"required,min=1,max=100"`
    LastName  string `json:"last_name" binding:"max=100"`
}

type UpdateUserRequest struct {
    FirstName string `json:"first_name" binding:"omitempty,min=1,max=100"`
    LastName  string `json:"last_name" binding:"omitempty,max=100"`
}

type ListUsersRequest struct {
    Page    int    `form:"page" binding:"min=1"`
    PerPage int    `form:"per_page" binding:"min=1,max=100"`
    Sort    string `form:"sort" binding:"omitempty,oneof=created_at updated_at email"`
    Order   string `form:"order" binding:"omitempty,oneof=asc desc"`
    Status  string `form:"status" binding:"omitempty,oneof=active inactive"`
}

func (r *ListUsersRequest) SetDefaults() {
    if r.Page == 0 {
        r.Page = 1
    }
    if r.PerPage == 0 {
        r.PerPage = 20
    }
    if r.Sort == "" {
        r.Sort = "created_at"
    }
    if r.Order == "" {
        r.Order = "desc"
    }
}
```

### Handler Implementation

```go
// interface/http/handler/user_handler.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "service/interface/http/request"
    "service/interface/http/response"
)

type UserHandler struct {
    createUser *usecase.CreateUserUseCase
    getUser    *usecase.GetUserUseCase
    listUsers  *usecase.ListUsersUseCase
}

func (h *UserHandler) Create(c *gin.Context) {
    var req request.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, response.ErrorResponse{
            Error: response.ErrorBody{
                Code:    "VALIDATION_ERROR",
                Message: "Invalid request body",
                Details: parseValidationErrors(err),
            },
        })
        return
    }

    output, err := h.createUser.Execute(c.Request.Context(), dto.CreateUserInput{
        Email:     req.Email,
        FirstName: req.FirstName,
        LastName:  req.LastName,
    })
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, response.Response{
        Data: output,
    })
}

func (h *UserHandler) GetByID(c *gin.Context) {
    id := c.Param("id")

    output, err := h.getUser.Execute(c.Request.Context(), id)
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, response.Response{
        Data: output,
    })
}

func (h *UserHandler) List(c *gin.Context) {
    var req request.ListUsersRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(http.StatusBadRequest, response.ErrorResponse{
            Error: response.ErrorBody{
                Code:    "VALIDATION_ERROR",
                Message: "Invalid query parameters",
            },
        })
        return
    }
    req.SetDefaults()

    output, total, err := h.listUsers.Execute(c.Request.Context(), dto.ListUsersInput{
        Page:    req.Page,
        PerPage: req.PerPage,
        Sort:    req.Sort,
        Order:   req.Order,
        Status:  req.Status,
    })
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, response.Response{
        Data: output,
        Meta: response.NewPaginationMeta(req.Page, req.PerPage, total),
    })
}

func (h *UserHandler) Delete(c *gin.Context) {
    id := c.Param("id")

    if err := h.deleteUser.Execute(c.Request.Context(), id); err != nil {
        handleError(c, err)
        return
    }

    c.Status(http.StatusNoContent)
}
```

### Validation Error Parser

```go
// interface/http/handler/validation.go
package handler

import (
    "github.com/go-playground/validator/v10"
)

func parseValidationErrors(err error) map[string]any {
    details := make(map[string]any)

    if validationErrors, ok := err.(validator.ValidationErrors); ok {
        for _, fieldError := range validationErrors {
            field := toSnakeCase(fieldError.Field())
            details[field] = validationMessage(fieldError)
        }
    }

    return details
}

func validationMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return "is required"
    case "email":
        return "must be a valid email"
    case "min":
        return fmt.Sprintf("must be at least %s characters", fe.Param())
    case "max":
        return fmt.Sprintf("must be at most %s characters", fe.Param())
    case "oneof":
        return fmt.Sprintf("must be one of: %s", fe.Param())
    default:
        return "is invalid"
    }
}
```

## API Versioning

### URL Path Versioning

```go
func SetupRouter(
    v1UserHandler *handler.UserHandler,
) *gin.Engine {
    router := gin.New()

    // API v1
    v1 := router.Group("/api/v1")
    {
        v1.POST("/users", v1UserHandler.Create)
        v1.GET("/users/:id", v1UserHandler.GetByID)
    }

    // Future: API v2
    // v2 := router.Group("/api/v2")
    // {
    //     v2.POST("/users", v2UserHandler.Create)
    // }

    return router
}
```

### Versioning Guidelines

1. **Major versions in URL** - `/api/v1`, `/api/v2`
2. **Backward compatible changes** - Don't require new version
3. **Breaking changes** - New major version
4. **Deprecation** - Announce 6 months before removal

## Date/Time Format

Always use ISO 8601 / RFC 3339:

```json
{
  "created_at": "2025-01-16T10:30:00Z",
  "updated_at": "2025-01-16T15:45:30.123Z"
}
```

```go
time.Now().UTC().Format(time.RFC3339Nano)
```

## Idempotency

For POST requests, support idempotency key:

```
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

```go
func (h *OrderHandler) Create(c *gin.Context) {
    idempotencyKey := c.GetHeader("Idempotency-Key")
    if idempotencyKey != "" {
        // Check if request was already processed
        if cached := h.cache.Get(idempotencyKey); cached != nil {
            c.JSON(http.StatusOK, cached)
            return
        }
    }

    // Process request...

    if idempotencyKey != "" {
        h.cache.Set(idempotencyKey, response, 24*time.Hour)
    }
}
```

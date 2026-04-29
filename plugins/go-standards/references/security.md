# Security Best Practices

## Input Validation

### Validation in Request Layer

Use Gin's binding with validator tags:

```go
// interface/http/request/user_request.go
package request

type CreateUserRequest struct {
    Email     string `json:"email" binding:"required,email,max=255"`
    FirstName string `json:"first_name" binding:"required,min=1,max=100,alphaunicode"`
    LastName  string `json:"last_name" binding:"omitempty,max=100,alphaunicode"`
    Age       int    `json:"age" binding:"omitempty,min=0,max=150"`
    Phone     string `json:"phone" binding:"omitempty,e164"`
}

type UpdatePasswordRequest struct {
    CurrentPassword string `json:"current_password" binding:"required,min=8"`
    NewPassword     string `json:"new_password" binding:"required,min=8,max=128,nefield=CurrentPassword"`
}
```

### Validation in Domain Layer

```go
// domain/valueobject/email.go
package valueobject

import (
    "errors"
    "net/mail"
    "strings"
)

var (
    ErrInvalidEmail    = errors.New("invalid email format")
    ErrEmailTooLong    = errors.New("email exceeds maximum length")
    ErrDisposableEmail = errors.New("disposable emails not allowed")
)

var disposableDomains = map[string]bool{
    "tempmail.com":    true,
    "throwaway.email": true,
    // ... more disposable domains
}

func NewEmail(value string) (Email, error) {
    value = strings.TrimSpace(strings.ToLower(value))

    if len(value) > 255 {
        return Email{}, ErrEmailTooLong
    }

    addr, err := mail.ParseAddress(value)
    if err != nil {
        return Email{}, ErrInvalidEmail
    }

    parts := strings.Split(addr.Address, "@")
    if len(parts) != 2 {
        return Email{}, ErrInvalidEmail
    }

    domain := parts[1]
    if disposableDomains[domain] {
        return Email{}, ErrDisposableEmail
    }

    return Email{value: addr.Address}, nil
}
```

### SQL Injection Prevention

GORM uses parameterized queries by default:

```go
// SAFE - parameterized
db.Where("email = ?", email).First(&user)
db.Where("id IN ?", ids).Find(&users)

// UNSAFE - string concatenation
db.Where("email = '" + email + "'").First(&user) // NEVER DO THIS

// SAFE - struct conditions
db.Where(&User{Email: email}).First(&user)
```

### XSS Prevention

For APIs returning JSON, data is inherently escaped. For any HTML rendering:

```go
import "html"

// Escape user content before storing or rendering
sanitized := html.EscapeString(userInput)
```

## Authentication

### JWT with API Keys

```go
// infrastructure/auth/jwt.go
package auth

import (
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

var (
    ErrInvalidToken = errors.New("invalid token")
    ErrExpiredToken = errors.New("token expired")
)

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

type JWTService struct {
    secretKey     []byte
    tokenDuration time.Duration
}

func NewJWTService(secretKey string, duration time.Duration) *JWTService {
    return &JWTService{
        secretKey:     []byte(secretKey),
        tokenDuration: duration,
    }
}

func (s *JWTService) GenerateToken(userID, email string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(s.tokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(s.secretKey)
}

func (s *JWTService) ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (any, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, ErrInvalidToken
        }
        return s.secretKey, nil
    })

    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, ErrExpiredToken
        }
        return nil, ErrInvalidToken
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, ErrInvalidToken
    }

    return claims, nil
}
```

### Auth Middleware

```go
// interface/http/middleware/auth.go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "service/infrastructure/auth"
)

func JWTAuth(jwtService *auth.JWTService) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": gin.H{
                    "code":    "MISSING_TOKEN",
                    "message": "Authorization header required",
                },
            })
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": gin.H{
                    "code":    "INVALID_TOKEN_FORMAT",
                    "message": "Invalid authorization header format",
                },
            })
            return
        }

        claims, err := jwtService.ValidateToken(parts[1])
        if err != nil {
            status := http.StatusUnauthorized
            code := "INVALID_TOKEN"
            message := "Invalid token"

            if errors.Is(err, auth.ErrExpiredToken) {
                code = "TOKEN_EXPIRED"
                message = "Token has expired"
            }

            c.AbortWithStatusJSON(status, gin.H{
                "error": gin.H{
                    "code":    code,
                    "message": message,
                },
            })
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("user_email", claims.Email)
        c.Next()
    }
}

func GetUserID(c *gin.Context) string {
    if userID, exists := c.Get("user_id"); exists {
        return userID.(string)
    }
    return ""
}
```

### API Key Authentication

```go
// interface/http/middleware/api_key.go
package middleware

import (
    "crypto/subtle"
    "net/http"

    "github.com/gin-gonic/gin"
)

func APIKeyAuth(validKey string) gin.HandlerFunc {
    return func(c *gin.Context) {
        apiKey := c.GetHeader("X-API-Key")
        if apiKey == "" {
            apiKey = c.Query("api_key")
        }

        // Constant-time comparison to prevent timing attacks
        if subtle.ConstantTimeCompare([]byte(apiKey), []byte(validKey)) != 1 {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": gin.H{
                    "code":    "INVALID_API_KEY",
                    "message": "Invalid or missing API key",
                },
            })
            return
        }

        c.Next()
    }
}
```

## Password Handling

### Hashing

```go
// infrastructure/auth/password.go
package auth

import (
    "golang.org/x/crypto/bcrypt"
)

const bcryptCost = 12

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcryptCost)
    return string(bytes), err
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### Password Requirements

```go
// domain/valueobject/password.go
package valueobject

import (
    "errors"
    "unicode"
)

var (
    ErrPasswordTooShort   = errors.New("password must be at least 8 characters")
    ErrPasswordTooLong    = errors.New("password must be at most 128 characters")
    ErrPasswordNoUpper    = errors.New("password must contain uppercase letter")
    ErrPasswordNoLower    = errors.New("password must contain lowercase letter")
    ErrPasswordNoDigit    = errors.New("password must contain digit")
)

func ValidatePassword(password string) error {
    if len(password) < 8 {
        return ErrPasswordTooShort
    }
    if len(password) > 128 {
        return ErrPasswordTooLong
    }

    var hasUpper, hasLower, hasDigit bool
    for _, r := range password {
        switch {
        case unicode.IsUpper(r):
            hasUpper = true
        case unicode.IsLower(r):
            hasLower = true
        case unicode.IsDigit(r):
            hasDigit = true
        }
    }

    if !hasUpper {
        return ErrPasswordNoUpper
    }
    if !hasLower {
        return ErrPasswordNoLower
    }
    if !hasDigit {
        return ErrPasswordNoDigit
    }

    return nil
}
```

## Rate Limiting

```go
// interface/http/middleware/rate_limit.go
package middleware

import (
    "net/http"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
)

type RateLimiter struct {
    requests map[string][]time.Time
    mu       sync.Mutex
    limit    int
    window   time.Duration
}

func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
    return &RateLimiter{
        requests: make(map[string][]time.Time),
        limit:    limit,
        window:   window,
    }
}

func (rl *RateLimiter) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        ip := c.ClientIP()

        rl.mu.Lock()
        now := time.Now()
        windowStart := now.Add(-rl.window)

        // Clean old requests
        requests := rl.requests[ip]
        validRequests := make([]time.Time, 0)
        for _, t := range requests {
            if t.After(windowStart) {
                validRequests = append(validRequests, t)
            }
        }

        if len(validRequests) >= rl.limit {
            rl.mu.Unlock()
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": gin.H{
                    "code":    "RATE_LIMIT_EXCEEDED",
                    "message": "Too many requests",
                },
            })
            return
        }

        validRequests = append(validRequests, now)
        rl.requests[ip] = validRequests
        rl.mu.Unlock()

        c.Header("X-RateLimit-Limit", fmt.Sprintf("%d", rl.limit))
        c.Header("X-RateLimit-Remaining", fmt.Sprintf("%d", rl.limit-len(validRequests)))

        c.Next()
    }
}

// Usage
limiter := NewRateLimiter(100, time.Minute) // 100 requests per minute
router.Use(limiter.Middleware())
```

## CORS Configuration

```go
// interface/http/middleware/cors.go
package middleware

import (
    "github.com/gin-gonic/gin"
)

func CORS(allowedOrigins []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        origin := c.GetHeader("Origin")

        allowed := false
        for _, o := range allowedOrigins {
            if o == "*" || o == origin {
                allowed = true
                break
            }
        }

        if allowed {
            c.Header("Access-Control-Allow-Origin", origin)
            c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
            c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
            c.Header("Access-Control-Max-Age", "86400")
        }

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }

        c.Next()
    }
}
```

## Secrets Management

### Environment Variables

Never hardcode secrets:

```go
// BAD
const apiKey = "secret-key-123"

// GOOD
apiKey := os.Getenv("API_KEY")
```

### Sensitive Data in Logs

Never log sensitive data:

```go
// BAD
slog.Info("User login", "password", password)
slog.Info("API call", "token", token)

// GOOD
slog.Info("User login", "user_id", userID)
slog.Info("API call", "endpoint", endpoint)
```

### Masking in Responses

```go
type UserResponse struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    // Password is NEVER included
}

func (u *User) ToResponse() UserResponse {
    return UserResponse{
        ID:    u.ID,
        Email: u.Email,
    }
}
```

## Security Headers

```go
// interface/http/middleware/security.go
package middleware

import "github.com/gin-gonic/gin"

func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Next()
    }
}
```

## Checklist

- [ ] All user input validated and sanitized
- [ ] Parameterized queries for all database operations
- [ ] Passwords hashed with bcrypt (cost >= 12)
- [ ] JWT tokens with reasonable expiration
- [ ] Rate limiting on authentication endpoints
- [ ] CORS properly configured
- [ ] Security headers set
- [ ] No secrets in code or logs
- [ ] HTTPS enforced in production
- [ ] Dependencies regularly updated

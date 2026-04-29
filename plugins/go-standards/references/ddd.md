# Domain-Driven Design (DDD) Patterns

## Overview

DDD helps us model complex business domains in code. We use tactical DDD patterns within Clean Architecture layers.

## Core Concepts

### Ubiquitous Language

Use the same terms in code as in business discussions:

| Business Term | Code |
|--------------|------|
| User registers | `NewUser()` |
| Order is placed | `PlaceOrder()` |
| Payment is processed | `ProcessPayment()` |
| Membership expires | `membership.IsExpired()` |

**Don't use technical jargon in domain layer:**
```go
// BAD - technical terms
func (u *User) InsertIntoDB()
func (o *Order) SerializeToJSON()

// GOOD - business terms
func (u *User) Register()
func (o *Order) Place()
```

## Building Blocks

### 1. Entities

Objects with **identity** that persists over time. Two entities with same attributes but different IDs are different.

```go
// domain/entity/user.go
package entity

import (
    "time"
    "service/domain/valueobject"
)

// User is an aggregate root with identity
type User struct {
    id        valueobject.UserID
    email     valueobject.Email
    firstName string
    lastName  string
    status    UserStatus
    createdAt time.Time
    updatedAt time.Time
}

// Constructor with validation
func NewUser(email valueobject.Email, firstName string) (*User, error) {
    if firstName == "" {
        return nil, ErrEmptyFirstName
    }

    return &User{
        id:        valueobject.NewUserID(),
        email:     email,
        firstName: firstName,
        status:    StatusActive,
        createdAt: time.Now().UTC(),
        updatedAt: time.Now().UTC(),
    }, nil
}

// RestoreUser for repository reconstruction (no validation)
func RestoreUser(
    id valueobject.UserID,
    email valueobject.Email,
    firstName, lastName string,
    status UserStatus,
    createdAt, updatedAt time.Time,
) *User {
    return &User{
        id:        id,
        email:     email,
        firstName: firstName,
        lastName:  lastName,
        status:    status,
        createdAt: createdAt,
        updatedAt: updatedAt,
    }
}

// Business methods
func (u *User) UpdateProfile(firstName, lastName string) error {
    if firstName == "" {
        return ErrEmptyFirstName
    }
    u.firstName = firstName
    u.lastName = lastName
    u.updatedAt = time.Now().UTC()
    return nil
}

func (u *User) Deactivate() {
    u.status = StatusInactive
    u.updatedAt = time.Now().UTC()
}

func (u *User) IsActive() bool {
    return u.status == StatusActive
}

// Getters (no setters - use business methods)
func (u *User) ID() valueobject.UserID     { return u.id }
func (u *User) Email() valueobject.Email   { return u.email }
func (u *User) FirstName() string          { return u.firstName }
func (u *User) LastName() string           { return u.lastName }
func (u *User) Status() UserStatus         { return u.status }
func (u *User) CreatedAt() time.Time       { return u.createdAt }
func (u *User) UpdatedAt() time.Time       { return u.updatedAt }
```

**Key principles:**
- Private fields, public getters
- Business methods for state changes
- Validation in constructor
- `RestoreUser()` for hydration from DB

### 2. Value Objects

Objects defined by their **attributes**, not identity. Immutable.

```go
// domain/valueobject/email.go
package valueobject

import (
    "errors"
    "regexp"
    "strings"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

var ErrInvalidEmail = errors.New("invalid email format")

// Email is an immutable value object
type Email struct {
    value string
}

// NewEmail creates Email with validation
func NewEmail(value string) (Email, error) {
    normalized := strings.ToLower(strings.TrimSpace(value))
    if !emailRegex.MatchString(normalized) {
        return Email{}, ErrInvalidEmail
    }
    return Email{value: normalized}, nil
}

// MustEmail panics on invalid email (for tests only)
func MustEmail(value string) Email {
    email, err := NewEmail(value)
    if err != nil {
        panic(err)
    }
    return email
}

// Value returns the email string
func (e Email) Value() string {
    return e.value
}

// Domain returns email domain
func (e Email) Domain() string {
    parts := strings.Split(e.value, "@")
    if len(parts) == 2 {
        return parts[1]
    }
    return ""
}

// Equals compares two emails
func (e Email) Equals(other Email) bool {
    return e.value == other.value
}
```

```go
// domain/valueobject/money.go
package valueobject

import (
    "errors"
    "fmt"
)

var ErrNegativeAmount = errors.New("amount cannot be negative")
var ErrCurrencyMismatch = errors.New("currency mismatch")

type Money struct {
    amount   int64  // in cents
    currency string // ISO 4217
}

func NewMoney(amount int64, currency string) (Money, error) {
    if amount < 0 {
        return Money{}, ErrNegativeAmount
    }
    return Money{amount: amount, currency: currency}, nil
}

func (m Money) Amount() int64    { return m.amount }
func (m Money) Currency() string { return m.currency }

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, ErrCurrencyMismatch
    }
    return Money{amount: m.amount + other.amount, currency: m.currency}, nil
}

func (m Money) Multiply(factor int64) Money {
    return Money{amount: m.amount * factor, currency: m.currency}
}

func (m Money) String() string {
    return fmt.Sprintf("%.2f %s", float64(m.amount)/100, m.currency)
}

func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}
```

```go
// domain/valueobject/user_id.go
package valueobject

import "github.com/google/uuid"

type UserID struct {
    value string
}

func NewUserID() UserID {
    return UserID{value: uuid.New().String()}
}

func ParseUserID(value string) (UserID, error) {
    if _, err := uuid.Parse(value); err != nil {
        return UserID{}, err
    }
    return UserID{value: value}, nil
}

func MustUserID(value string) UserID {
    id, err := ParseUserID(value)
    if err != nil {
        panic(err)
    }
    return id
}

func (id UserID) Value() string { return id.value }
func (id UserID) IsEmpty() bool { return id.value == "" }

func (id UserID) Equals(other UserID) bool {
    return id.value == other.value
}
```

**When to use Value Objects:**
- IDs (UserID, OrderID, ChatID)
- Email, Phone, Address
- Money, Currency
- Date ranges, Time periods
- Coordinates, Colors
- Any domain concept that's defined by its attributes

### 3. Aggregates

Cluster of entities and value objects with a single **root entity**. All access goes through the root.

```go
// domain/entity/order.go - Order is an aggregate root
package entity

type Order struct {
    id         valueobject.OrderID
    customerID valueobject.UserID
    items      []OrderItem           // Part of aggregate
    status     OrderStatus
    total      valueobject.Money
    createdAt  time.Time
}

// OrderItem is an entity within Order aggregate
type OrderItem struct {
    id        valueobject.OrderItemID
    productID valueobject.ProductID
    quantity  int
    price     valueobject.Money
}

func NewOrder(customerID valueobject.UserID) *Order {
    return &Order{
        id:         valueobject.NewOrderID(),
        customerID: customerID,
        items:      make([]OrderItem, 0),
        status:     StatusDraft,
        total:      valueobject.MustMoney(0, "USD"),
        createdAt:  time.Now().UTC(),
    }
}

// Business methods on aggregate root
func (o *Order) AddItem(productID valueobject.ProductID, quantity int, price valueobject.Money) error {
    if o.status != StatusDraft {
        return ErrOrderNotDraft
    }
    if quantity <= 0 {
        return ErrInvalidQuantity
    }

    item := OrderItem{
        id:        valueobject.NewOrderItemID(),
        productID: productID,
        quantity:  quantity,
        price:     price,
    }
    o.items = append(o.items, item)
    o.recalculateTotal()
    return nil
}

func (o *Order) RemoveItem(itemID valueobject.OrderItemID) error {
    if o.status != StatusDraft {
        return ErrOrderNotDraft
    }

    for i, item := range o.items {
        if item.id.Equals(itemID) {
            o.items = append(o.items[:i], o.items[i+1:]...)
            o.recalculateTotal()
            return nil
        }
    }
    return ErrItemNotFound
}

func (o *Order) Place() error {
    if len(o.items) == 0 {
        return ErrEmptyOrder
    }
    if o.status != StatusDraft {
        return ErrOrderNotDraft
    }
    o.status = StatusPlaced
    return nil
}

func (o *Order) recalculateTotal() {
    total := valueobject.MustMoney(0, "USD")
    for _, item := range o.items {
        itemTotal := item.price.Multiply(int64(item.quantity))
        total, _ = total.Add(itemTotal)
    }
    o.total = total
}

// Items returns a copy to prevent external modification
func (o *Order) Items() []OrderItem {
    result := make([]OrderItem, len(o.items))
    copy(result, o.items)
    return result
}
```

**Aggregate rules:**
- Only root entity has global identity
- External references only to root
- Invariants enforced within aggregate boundary
- Delete cascades within aggregate

### 4. Repository Interfaces

Define data access contracts in domain, implement in infrastructure.

```go
// domain/repository/user_repository.go
package repository

import (
    "context"
    "service/domain/entity"
    "service/domain/valueobject"
)

type UserRepository interface {
    // Commands
    Save(ctx context.Context, user *entity.User) error
    Delete(ctx context.Context, id valueobject.UserID) error

    // Queries
    FindByID(ctx context.Context, id valueobject.UserID) (*entity.User, error)
    FindByEmail(ctx context.Context, email valueobject.Email) (*entity.User, error)
    FindAll(ctx context.Context) ([]*entity.User, error)
    Exists(ctx context.Context, id valueobject.UserID) (bool, error)
    Count(ctx context.Context) (int, error)
}

// Custom errors
var (
    ErrUserNotFound = errors.New("user not found")
)
```

```go
// domain/repository/order_repository.go
package repository

type OrderRepository interface {
    Save(ctx context.Context, order *entity.Order) error
    FindByID(ctx context.Context, id valueobject.OrderID) (*entity.Order, error)
    FindByCustomer(ctx context.Context, customerID valueobject.UserID) ([]*entity.Order, error)
    Delete(ctx context.Context, id valueobject.OrderID) error
}
```

### 5. Domain Services

Stateless operations that don't belong to a single entity.

```go
// domain/service/pricing_service.go
package service

import (
    "service/domain/entity"
    "service/domain/valueobject"
)

// PricingService calculates prices across multiple entities
type PricingService struct{}

func NewPricingService() *PricingService {
    return &PricingService{}
}

func (s *PricingService) CalculateOrderTotal(order *entity.Order, discount *entity.Discount) valueobject.Money {
    total := order.Total()

    if discount != nil && discount.IsValid() {
        total = discount.Apply(total)
    }

    return total
}

func (s *PricingService) CalculateShipping(order *entity.Order, destination valueobject.Address) valueobject.Money {
    // Complex shipping calculation logic
    baseRate := valueobject.MustMoney(500, "USD") // $5.00

    if order.Total().Amount() > 5000 { // Over $50
        return valueobject.MustMoney(0, "USD") // Free shipping
    }

    return baseRate
}
```

**When to use Domain Services:**
- Operation spans multiple aggregates
- Logic doesn't fit in any single entity
- Stateless business calculations

### 6. Domain Events

Record that something happened in the domain.

```go
// domain/event/events.go
package event

import "time"

type DomainEvent interface {
    OccurredAt() time.Time
    EventType() string
}

type BaseEvent struct {
    occurredAt time.Time
}

func (e BaseEvent) OccurredAt() time.Time { return e.occurredAt }

// UserRegistered event
type UserRegistered struct {
    BaseEvent
    UserID    string
    Email     string
    FirstName string
}

func NewUserRegistered(userID, email, firstName string) UserRegistered {
    return UserRegistered{
        BaseEvent: BaseEvent{occurredAt: time.Now().UTC()},
        UserID:    userID,
        Email:     email,
        FirstName: firstName,
    }
}

func (e UserRegistered) EventType() string { return "user.registered" }

// OrderPlaced event
type OrderPlaced struct {
    BaseEvent
    OrderID    string
    CustomerID string
    Total      int64
}

func NewOrderPlaced(orderID, customerID string, total int64) OrderPlaced {
    return OrderPlaced{
        BaseEvent:  BaseEvent{occurredAt: time.Now().UTC()},
        OrderID:    orderID,
        CustomerID: customerID,
        Total:      total,
    }
}

func (e OrderPlaced) EventType() string { return "order.placed" }
```

```go
// Entity can collect events
type User struct {
    // ... fields
    events []event.DomainEvent
}

func (u *User) Register() {
    // ... business logic
    u.events = append(u.events, event.NewUserRegistered(
        u.id.Value(),
        u.email.Value(),
        u.firstName,
    ))
}

func (u *User) Events() []event.DomainEvent {
    return u.events
}

func (u *User) ClearEvents() {
    u.events = nil
}
```

## Directory Structure

```
domain/
├── entity/
│   ├── user.go              # User aggregate root
│   ├── user_status.go       # User status enum
│   ├── order.go             # Order aggregate root
│   ├── order_item.go        # Order item (part of aggregate)
│   └── errors.go            # Domain errors
│
├── valueobject/
│   ├── user_id.go           # User ID
│   ├── order_id.go          # Order ID
│   ├── email.go             # Email
│   ├── money.go             # Money
│   ├── address.go           # Address
│   └── errors.go            # Value object errors
│
├── repository/
│   ├── user_repository.go   # User repository interface
│   └── order_repository.go  # Order repository interface
│
├── service/
│   └── pricing_service.go   # Domain services
│
└── event/
    ├── events.go            # Domain events
    └── handler.go           # Event handler interface
```

## DDD + Clean Architecture Mapping

| DDD Concept | CA Layer | Location |
|-------------|----------|----------|
| Entities | Domain | `domain/entity/` |
| Value Objects | Domain | `domain/valueobject/` |
| Repository Interfaces | Domain | `domain/repository/` |
| Domain Services | Domain | `domain/service/` |
| Domain Events | Domain | `domain/event/` |
| Use Cases | Application | `application/usecase/` |
| Repository Implementations | Infrastructure | `infrastructure/persistence/` |
| Event Handlers | Application/Infrastructure | depends on handler |

## Anti-Patterns to Avoid

### Anemic Domain Model
```go
// BAD - Entity is just data holder
type User struct {
    ID        string
    Email     string
    FirstName string
}

// Logic in service instead of entity
func (s *UserService) UpdateProfile(user *User, firstName string) {
    user.FirstName = firstName  // Direct field access
}

// GOOD - Rich domain model
type User struct {
    id        valueobject.UserID
    email     valueobject.Email
    firstName string
}

func (u *User) UpdateProfile(firstName string) error {
    if firstName == "" {
        return ErrEmptyFirstName
    }
    u.firstName = firstName
    return nil
}
```

### Primitive Obsession
```go
// BAD - Primitives everywhere
func CreateUser(email string, firstName string) (*User, error)
func FindByEmail(email string) (*User, error)

// GOOD - Value objects
func CreateUser(email valueobject.Email, firstName string) (*User, error)
func FindByEmail(email valueobject.Email) (*User, error)
```

## Domain Primitives (Avoiding Primitive Obsession)

### What is Primitive Obsession?

Primitive Obsession is a code smell where primitive types (`string`, `int`, `float64`) are used to represent domain concepts. This leads to:
- **Unclear APIs** - `func SendEmail(to, from, subject, body string)` - easy to mix up parameters
- **Scattered validation** - validation logic duplicated across codebase
- **Security vulnerabilities** - attack vectors in unvalidated input
- **Lost business rules** - domain knowledge hidden in procedural code

### Domain Primitives Pattern

Domain Primitives are **self-validating value objects** that replace primitive types. They encapsulate:
1. **Validation** - enforced at construction time
2. **Business rules** - methods that express domain concepts
3. **Type safety** - compiler catches type mismatches

### When to Create a Domain Primitive

| Condition | Action |
|-----------|--------|
| Value requires validation | Create Domain Primitive |
| Value has business rules/behavior | Create Domain Primitive |
| Value represents domain concept | Create Domain Primitive |
| Same validation in multiple places | Create Domain Primitive |
| Parameter confusion possible | Create Domain Primitive |
| Simple flag or counter | Keep primitive |
| Internal implementation detail | Keep primitive |

### Implementation Pattern

```go
// domain/valueobject/url.go
package valueobject

import (
    "errors"
    "net/url"
    "strings"
)

var (
    ErrEmptyURL    = errors.New("URL cannot be empty")
    ErrInvalidURL  = errors.New("invalid URL format")
    ErrNotHTTPS    = errors.New("URL must use HTTPS")
)

// URL is a validated URL domain primitive
type URL struct {
    value string
}

// NewURL creates URL with validation (for external input)
func NewURL(value string) (URL, error) {
    // 1. Length check first (cheap)
    if value == "" {
        return URL{}, ErrEmptyURL
    }

    // 2. Format validation
    parsed, err := url.Parse(value)
    if err != nil || parsed.Host == "" {
        return URL{}, ErrInvalidURL
    }

    return URL{value: value}, nil
}

// NewHTTPSURL creates URL that must be HTTPS
func NewHTTPSURL(value string) (URL, error) {
    u, err := NewURL(value)
    if err != nil {
        return URL{}, err
    }
    if !strings.HasPrefix(value, "https://") {
        return URL{}, ErrNotHTTPS
    }
    return u, nil
}

// RestoreURL recreates from persistence (no validation)
func RestoreURL(value string) URL {
    return URL{value: value}
}

// String returns URL as string
func (u URL) String() string { return u.value }

// IsEmpty checks if URL is empty
func (u URL) IsEmpty() bool { return u.value == "" }

// Host returns the host part of URL
func (u URL) Host() string {
    parsed, _ := url.Parse(u.value)
    if parsed != nil {
        return parsed.Host
    }
    return ""
}

// IsHTTPS checks if URL uses HTTPS
func (u URL) IsHTTPS() bool {
    return strings.HasPrefix(u.value, "https://")
}
```

### Validation Strategy

Follow this order for efficient validation:

```go
func NewEmail(value string) (Email, error) {
    // 1. Length check (O(1), cheap)
    if len(value) < 5 || len(value) > 320 {
        return Email{}, ErrInvalidEmailLength
    }

    // 2. Simple character check (O(n), medium)
    if !strings.Contains(value, "@") {
        return Email{}, ErrInvalidEmailFormat
    }

    // 3. Regex/complex validation (expensive, last)
    if !emailRegex.MatchString(value) {
        return Email{}, ErrInvalidEmailFormat
    }

    return Email{value: strings.ToLower(value)}, nil
}
```

### Common Domain Primitives

| Domain Concept | Primitive | Why Domain Primitive |
|---------------|-----------|---------------------|
| Email | `string` | Format validation, normalization, domain extraction |
| URL | `string` | Format validation, scheme checking, host extraction |
| Phone | `string` | Format validation, country code handling |
| Money | `int64` | Currency pairing, arithmetic operations |
| Temperature | `float64` | Range validation (0-2 for LLM) |
| TokenLimit | `int` | Range validation (1-128000) |
| Percentage | `float64` | Range validation (0-100) |
| ID | `string` | UUID validation, type safety |
| Name | `string` | Length limits, character validation |
| Prompt | `string` | Non-empty validation, template support |

### Entity Fields: When to Use What

```go
// GOOD - Domain primitives for validated/business concepts
type SummaryPreset struct {
    id                    PresetID        // ID type - validation
    name                  PresetName      // Business concept with length limit
    systemPrompt          SystemPrompt    // Required, non-empty
    userPromptTemplate    PromptTemplate  // Required template
    model                 ModelName       // LLM model name
    maxTokens             TokenLimit      // Range: 1-128000
    temperature           Temperature     // Range: 0.0-2.0
    webhookURL            *URL            // Optional, validated URL

    // GOOD - Simple primitives for non-business data
    description           string          // Optional, no validation needed
    isDefault             bool            // Simple flag
    isActive              bool            // Simple flag
    retryCount            int             // Internal counter
    createdAt             time.Time       // Framework type
    updatedAt             time.Time       // Framework type
}
```

### Immutability in Go

Go's value semantics provide natural immutability:

```go
// Value receiver = copy = immutable
func (t Temperature) Float64() float64 { return t.value }

// Modification returns NEW value
func (t Temperature) Increase(delta float64) (Temperature, error) {
    return NewTemperature(t.value + delta)
}
```

**Never use pointer receivers for value objects** - this breaks immutability.

### Testing Domain Primitives

```go
func TestNewTemperature(t *testing.T) {
    tests := []struct {
        name    string
        value   float64
        wantErr error
    }{
        {"valid zero", 0.0, nil},
        {"valid max", 2.0, nil},
        {"negative", -0.1, ErrInvalidTemperature},
        {"too high", 2.1, ErrInvalidTemperature},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := NewTemperature(tt.value)
            if err != tt.wantErr {
                t.Errorf("NewTemperature(%v) error = %v, want %v",
                    tt.value, err, tt.wantErr)
            }
        })
    }
}
```

### References

- [Avoid Primitive Obsession in Go](https://matteo.vaccari.name/posts/avoid-primitive-obsession-in-go/)
- [Domain-driven Design in Go: Value Objects](https://dennisvis.dev/blog/ddd-in-go-value-objects)
- [Value Objects in Go](https://victoramartinez.com/posts/value-objects-in-go-and-other-domain-driven-design-stuff/)

### Breaking Aggregate Boundaries
```go
// BAD - Accessing aggregate internals
order := orderRepo.FindByID(ctx, orderID)
order.Items[0].Quantity = 10  // Direct modification

// GOOD - Through aggregate root
order := orderRepo.FindByID(ctx, orderID)
order.UpdateItemQuantity(itemID, 10)  // Business method
```

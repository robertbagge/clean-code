# Go Style

General Go idioms that cut across multiple principles.

---

## Accept Interfaces, Return Structs

Functions should accept interfaces (for flexibility and testability) but
return concrete types (for clarity and composability):

```go
// Good: accepts interface, returns struct
func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

// Avoid: returning interface hides the concrete type
func NewUserService(repo UserRepository) UserRepository {
    return &UserService{repo: repo}
}
```

---

## Prefer Concrete Types for Data

Use **concrete types** (not interfaces) for:

* **Configuration structs** — simple data with no behavior
* **Domain models** — entities like `User`, `Order`
* **DTOs** — request/response payloads

```go
type Config struct {
    DatabaseURL string
    LogLevel    string
    MaxRetries  int
}
```

Interfaces are for behavior polymorphism, not data containers.

---

## Keep Packages Small and Cohesive

Each package should have a **single responsibility**:

```go
// Good: focused packages
/user
    user.go        // User struct + domain logic
/repo
    user_repo.go   // UserRepository implementation

// Avoid: "god packages" with unrelated concerns
/models
    user.go
    order.go
    payment.go
    notification.go
    utils.go
```

---

## Constructor Injection

Pass dependencies through constructors, not as globals:

```go
// Good: explicit dependencies
func NewOrderService(
    repo OrderRepository,
    notifier Notifier,
    logger *slog.Logger,
) *OrderService {
    return &OrderService{
        repo:     repo,
        notifier: notifier,
        logger:   logger,
    }
}

// Avoid: hidden global dependencies
var globalRepo = NewSQLRepo()

func NewOrderService() *OrderService {
    return &OrderService{repo: globalRepo}
}
```

---

## Prefer Functions Over Methods for Stateless Operations

If a function doesn't need receiver state, make it a plain function:

```go
// Good: stateless utility as function
func ValidateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return ErrInvalidEmail
    }
    return nil
}

// Unnecessary: method on a zero-value receiver
func (v *Validator) ValidateEmail(email string) error { ... }
```

---

## Further Reading

* [Go Proverbs](https://go-proverbs.github.io/)
* [Effective Go](https://go.dev/doc/effective_go)
* [KISS](../clean-code/kiss.md) — prefer the simplest working design

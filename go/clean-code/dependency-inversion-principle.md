# Dependency Inversion Principle (DIP) in Go

## Overview

The Dependency Inversion Principle states that high-level modules should not
depend on low-level modules; both should depend on abstractions. In Go, this
is achieved through interfaces, allowing high-level business logic to depend
on interface contracts rather than concrete implementations. This makes code
more flexible, testable, and maintainable.

## Core Concept

In Go, DIP means:

* Depend on interfaces, not concrete types
* High-level business logic defines the interfaces it needs
* Low-level implementations satisfy those interfaces
* Use dependency injection to provide implementations
* Balance abstraction with Go’s preference for simplicity

---

## Scaffolding for Examples (so snippets compile)

```go
package users

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "sync"
    "time"
)

var (
    ErrUserNotFound = errors.New("user not found")
)

// Domain model stays concrete.
type User struct {
    ID    string
    Name  string
    Email string
}

// Utility stub for examples.
func generateID() string { return fmt.Sprintf("u_%d", time.Now().UnixNano()) }
```

---

## Implementation Examples

### BAD — Concrete Database Dependency (DIP violation)

> High-level `UserService` depends directly on a concrete `*sql.DB` and SQL.

```go
import "database/sql"

// UserService directly depends on SQL database
type UserService struct {
    db *sql.DB // Concrete dependency
}

func (u *UserService) GetUser(id string) (*User, error) {
    // VIOLATION: Direct SQL dependency inside business logic
    row := u.db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id)

    var user User
    if err := row.Scan(&user.ID, &user.Name, &user.Email); err != nil {
        // Without a domain-level mapping, callers must know about sql.ErrNoRows
        return nil, err
    }
    return &user, nil
}

func (u *UserService) CreateUser(name, email string) error {
    // VIOLATION: Tied to SQL; can’t swap DB without changing business code
    _, err := u.db.Exec("INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
        generateID(), name, email)
    return err
}
```

### GOOD — Abstraction-Based (DIP compliant)

> High-level code declares the behavior it needs; low-level code implements it.

Note:
> The **UserRepository** interface lives alongside the **UserService**
> (the consumer). Concrete repos (**SQLUserRepository**,
> **InMemoryUserRepository**, etc.) just implement it, without importing the
> service package. This follows Go's convention: consumers define interfaces,
> providers implement them.

```go
import "context"

// High-level defines what it needs.
type UserRepository interface {
    GetUser(ctx context.Context, id string) (*User, error)
    CreateUser(ctx context.Context, user *User) error
    UpdateUser(ctx context.Context, user *User) error
    DeleteUser(ctx context.Context, id string) error
}

// UserService depends on abstraction.
type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (u *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    return u.repo.GetUser(ctx, id)
}

func (u *UserService) CreateUser(ctx context.Context, name, email string) (
    *User, error) {
    user := &User{ID: generateID(), Name: name, Email: email}
    if err := u.repo.CreateUser(ctx, user); err != nil {
        return nil, err
    }
    return user, nil
}
```

#### SQL implementation (maps driver not-found to domain not-found)

```go
type SQLUserRepository struct {
    db *sql.DB
}

func NewSQLUserRepository(db *sql.DB) *SQLUserRepository {
    return &SQLUserRepository{db: db}
}

func (s *SQLUserRepository) GetUser(ctx context.Context, id string) (
    *User, error) {
    var user User
    err := s.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = ?", id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }
        return nil, err
    }
    return &user, nil
}

func (s *SQLUserRepository) CreateUser(ctx context.Context, user *User) error {
    _, err := s.db.ExecContext(ctx,
        "INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
        user.ID, user.Name, user.Email,
    )
    return err
}

func (s *SQLUserRepository) UpdateUser(ctx context.Context, user *User) error {
    _, err := s.db.ExecContext(ctx,
        "UPDATE users SET name = ?, email = ? WHERE id = ?",
        user.Name, user.Email, user.ID,
    )
    return err
}

func (s *SQLUserRepository) DeleteUser(ctx context.Context, id string) error {
    _, err := s.db.ExecContext(ctx, "DELETE FROM users WHERE id = ?", id)
    return err
}
```

#### In-memory implementation (great for tests/dev)

```go
type InMemoryUserRepository struct {
    mu    sync.RWMutex
    users map[string]*User
}

func NewInMemoryUserRepository() *InMemoryUserRepository {
    return &InMemoryUserRepository{users: make(map[string]*User)}
}

func (i *InMemoryUserRepository) GetUser(ctx context.Context, id string) (
    *User, error) {
    i.mu.RLock()
    defer i.mu.RUnlock()
    u, ok := i.users[id]
    if !ok {
        return nil, ErrUserNotFound
    }
    return u, nil
}

func (i *InMemoryUserRepository) CreateUser(
    ctx context.Context,
    user *User,
) error {
    i.mu.Lock()
    defer i.mu.Unlock()
    i.users[user.ID] = user
    return nil
}

func (i *InMemoryUserRepository) UpdateUser(
    ctx context.Context,
    user *User,
) error {
    i.mu.Lock()
    defer i.mu.Unlock()
    if _, ok := i.users[user.ID]; !ok {
        return ErrUserNotFound
    }
    i.users[user.ID] = user
    return nil
}

func (i *InMemoryUserRepository) DeleteUser(
    ctx context.Context,
    id string,
) error {
    i.mu.Lock()
    defer i.mu.Unlock()
    delete(i.users, id)
    return nil
}

// (Optional compile-time checks — uncomment if wanted)
// var _ UserRepository = (*SQLUserRepository)(nil)
// var _ UserRepository = (*InMemoryUserRepository)(nil)
```

---

## Anonymous (call-site) Interfaces (keep DIP tight)

```go
// Accept only the behavior you need at a call site.
func EnsureUserExists(
    ctx context.Context,
    c interface {
        GetUser(context.Context, string) (*User, error)
        CreateUser(context.Context, *User) error
    },
    id, name, email string,
) (*User, error) {
    if u, err := c.GetUser(ctx, id); err == nil {
        return u, nil
    } else if !errors.Is(err, ErrUserNotFound) {
        return nil, err
    }

    u := &User{ID: id, Name: name, Email: email}
    if err := c.CreateUser(ctx, u); err != nil {
        return nil, err
    }
    return u, nil
}
```

---

## When to Apply DIP in Go

### Use DIP for

* External dependencies (databases, APIs, file systems)
* Business logic that needs to be testable
* Components that might change or have multiple implementations
* Cross-cutting concerns (logging, metrics, tracing)

### Prefer Concrete Types for

* DTOs / domain models (data)
* Configuration structures
* Simple helpers with no swappability needs

```go
// Behavior abstraction (good DIP)
type Logger interface {
    Log(ctx context.Context, level, message string)
}

// Concrete data (good as struct)
type Config struct {
    DatabaseURL string
    Port        int
    APIKey      string
}
```

---

## Anti-patterns to Avoid

1. **Over-abstraction**: Creating interfaces for everything
2. **Leaky abstractions**: Interfaces that expose implementation details
   (e.g., `*sql.Rows`)
3. **Anemic / vague interfaces**: So generic they don’t convey intent
4. **Provider-owned interfaces**: Defining interfaces in the implementation
   package instead of where they're consumed

---

## Go-Specific DIP Techniques

1. **Accept interfaces, return structs**
2. **Define interfaces at usage points** (keep them close to consumers)
3. **Keep interfaces small** (pairs nicely with ISP)
4. **Constructor injection** (plain constructors or functional options)

---

## Key Takeaways

* Depend on **behavior** (interfaces), not concrete types
* Let **high-level** code declare the contracts it needs
* Inject implementations; don’t construct them inside business logic
* Prefer concrete types for **data & config**
* Mocks/fakes are trivial; tests stay fast

---

## Related Best Practices

For package structure, error placement, and testing strategies, etc., see
👉 [best-practices.md](../best-practices.md)

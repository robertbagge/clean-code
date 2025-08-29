# Interface Segregation Principle (ISP) in Go

## Overview

The Interface Segregation Principle states that no client should be forced
to depend on methods it doesn't use. In Go, this principle is naturally
supported by the language's philosophy of small, focused interfaces.
Go's implicit interface satisfaction and the ability to compose interfaces
make it easy to follow ISP.

## Core Concept

In Go, ISP means:

* Prefer many small interfaces over few large ones
* Interfaces should represent a single capability or behavior
* Clients should depend only on the methods they need
* Use interface composition to build complex contracts from simple ones

Go's proverb "The bigger the interface, the weaker the abstraction"
directly supports ISP.

---

## Scaffolding for Examples (so snippets compile)

```go
package users

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
)

var ErrNotSupported = errors.New("not supported")
var ErrNotFound    = errors.New("not found")

type User struct {
    ID    string
    Email string
}

type UserUpdate struct {
    Email *string
}

type Post struct {
    ID     string
    UserID string
    Title  string
}

type AuthToken struct{ Value string }
```

---

## Examples

### BAD — Monolithic interface violates ISP

### (illustrative, not for production)

> This shows the *pressure* a fat interface creates. It's intentionally
> non-idiomatic and not meant to be implemented.

```go
// BAD: Too many responsibilities in one interface.
type UserService interface {
    CreateUser(email, password string) error
    GetUser(id string) (*User, error)
    UpdateUser(id string, data map[string]interface{}) error
    DeleteUser(id string) error
    AuthenticateUser(email, password string) (AuthToken, error)
    ResetPassword(email string) error
    SendVerificationEmail(userID string) error
    UploadAvatar(userID string, data []byte) error
    GetUserPosts(userID string) ([]Post, error)
    BanUser(userID string) error
}

// Hypothetical client that only needs reads would be *forced* to
// implement unrelated methods if it had to satisfy UserService.
type SimpleUserViewer struct{}
/*
func (s *SimpleUserViewer) GetUser(id string) (*User, error) { ... }
func (s *SimpleUserViewer) CreateUser(email, password string) error {
    return ErrNotSupported
}
// ... 8 more stubs — ISP violation in practice
*/
```

### GOOD — Segregated interfaces align with ISP

```go
// Capability-focused interfaces
type UserReader interface {
    GetUser(ctx context.Context, id string) (*User, error)
}

type UserWriter interface {
    CreateUser(ctx context.Context, email, password string) (*User, error)
    UpdateUser(ctx context.Context, id string, updates UserUpdate) error
    DeleteUser(ctx context.Context, id string) error
}

type UserAuthenticator interface {
    Authenticate(
        ctx context.Context, email, password string,
    ) (AuthToken, error)
}

type UserNotifier interface {
    SendVerificationEmail(ctx context.Context, userID string) error
}

type UserMediaManager interface {
    UploadAvatar(ctx context.Context, userID string, data []byte) error
}

type UserModerator interface {
    BanUser(ctx context.Context, userID string, reason string) error
    UnbanUser(ctx context.Context, userID string) error
}
```

#### Simple implementation that only reads

#### (returns ErrNotFound correctly)

```go
type SimpleUserViewer struct {
    db *sql.DB
}

func (s *SimpleUserViewer) GetUser(
    ctx context.Context,
    id string,
) (*User, error) {
    var u User
    err := s.db.QueryRowContext(ctx,
        "SELECT id, email FROM users WHERE id = $1", id,
    ).Scan(&u.ID, &u.Email)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, err
    }
    return &u, nil
}
```

#### Admin service implements only the capabilities it needs

```go
type AdminUserService struct {
    db       *sql.DB
    notifier UserNotifier
}

func (a *AdminUserService) GetUser(
    ctx context.Context,
    id string,
) (*User, error) {
    var u User
    err := a.db.QueryRowContext(ctx,
        "SELECT id, email FROM users WHERE id = $1", id,
    ).Scan(&u.ID, &u.Email)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, err
    }
    return &u, nil
}

func (a *AdminUserService) BanUser(
    ctx context.Context,
    userID string,
    reason string,
) error {
    _, err := a.db.ExecContext(ctx,
        "UPDATE users SET banned = true, ban_reason = $1 WHERE id = $2",
        reason, userID,
    )
    return err
}
```

#### Client code depends only on what it uses

```go
func ShowUserProfile(
    ctx context.Context,
    reader UserReader,
    userID string,
) error {
    user, err := reader.GetUser(ctx, userID)
    if err != nil {
        return err
    }
    fmt.Printf("User: %s\n", user.Email)
    return nil
}

func ModerateUser(
    ctx context.Context, mod UserModerator, userID string,
) error {
    return mod.BanUser(ctx, userID, "Terms violation")
}
```

---

## Interface Composition (explicit)

```go
// Compose when consumers need multiple capabilities.
type UserReadWriter interface {
    UserReader
    UserWriter
}

func SaveAndShow(
    ctx context.Context,
    rw UserReadWriter,
    id string,
    updates UserUpdate,
) error {
    if err := rw.UpdateUser(ctx, id, updates); err != nil {
        return err
    }
    u, err := rw.GetUser(ctx, id)
    if err != nil {
        return err
    }
    fmt.Println("User:", u.Email)
    return nil
}
```

---

## Anonymous (call-site) Interfaces (keep params tiny)

```go
// Prevents interface creep in shared packages:
func ShowUserProfileLite(
    ctx context.Context,
    reader interface {
        GetUser(context.Context, string) (*User, error)
    },
    userID string,
) error {
    u, err := reader.GetUser(ctx, userID)
    if err != nil {
        return err
    }
    fmt.Println("User:", u.Email)
    return nil
}
```

---

## Testing Tip — Implement only what tests need

```go
type fakeReader struct{ m map[string]*User }

func (f fakeReader) GetUser(
    ctx context.Context,
    id string,
) (*User, error) {
    if u, ok := f.m[id]; ok {
        return u, nil
    }
    return nil, ErrNotFound
}

// Usage in tests:
// _ = ShowUserProfileLite(ctx,
//     fakeReader{m: map[string]*User{"42": {ID:"42", Email:"x@y"}}},
//     "42")
```

---

## Anti-patterns to Avoid

1. **Fat Interfaces**: Large interfaces with many methods
2. **Stub Implementations**: Methods that return "not implemented"
   just to satisfy an interface
3. **Interface Pollution**: Creating interfaces before they’re needed
4. **Exporting Interfaces Prematurely**: Don't export an interface
   unless code outside the package needs it

---

## Go-Specific ISP Techniques

1. **Single-Method Interfaces**: Many stdlib interfaces are one method
   (`io.Reader`, `io.Writer`)
2. **Interface Composition**: Build complex contracts from small
   capabilities
3. **Accept Interfaces, Return Structs**: Keep parameters minimal
   and focused
4. **Interface Discovery**: Extract interfaces from concrete types
   when consumers emerge

---

## Key Takeaways

* Keep interfaces small and focused
* Compose interfaces when multiple capabilities are required
* Clients should depend only on the methods they actually use
* Go's implicit interface satisfaction + call-site interfaces
  make ISP natural
* Follow stdlib patterns (e.g., `io` package)

---

## Related Best Practices

For package structure, error placement, and testing strategies,
etc., see
👉 [best-practices.md](../best-practices.md)

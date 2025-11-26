# Package Structure

A clean package structure helps enforce DIP and keep dependencies flowing in
the right direction.

---

## Suggested Layout

```txt
/domain
    user.go         // domain model (User struct)
    errors.go       // domain-level errors (ErrUserNotFound, ErrEmailTaken)
/service
    user_service.go // business logic (depends on domain + repo interface)
/repo
    sqlrepo.go      // low-level implementation of UserRepository
    mongorepo.go
    inmemoryrepo.go // test/development fake
```

---

## Guidelines

### 1. Domain Package

* Contains entities (structs like `User`, `Order`) and domain-level errors.
* Errors represent *business concerns*, not infra details.

```go
package domain

var (
    ErrUserNotFound     = errors.New("user not found")
    ErrEmailAlreadyUsed = errors.New("email already used")
)
```

### 2. Service Package

* Contains high-level business logic (`UserService`).
* Defines the **interfaces it needs** (e.g. `UserRepository`).
* Depends only on the `domain` package, not on infra.

### 3. Repo (Infra) Packages

* Concrete implementations: SQL, MongoDB, in-memory.
* Map **driver errors → domain errors**:

  ```go
  if errors.Is(err, sql.ErrNoRows) {
      return nil, domain.ErrUserNotFound
  }
  ```

* Never leak driver-specific errors to service or domain.

---

## Dependency Direction

* `service` → depends on `domain`
* `repo` → depends on `domain`
* `domain` → depends on nothing (pure types + errors)

This ensures the **dependency graph points inward**:

```
       ┌─────────┐
       │  main   │
       └────┬────┘
            │ wires
    ┌───────┴───────┐
    │               │
┌───▼───┐       ┌───▼───┐
│service│       │  repo │
└───┬───┘       └───┬───┘
    │               │
    └───────┬───────┘
            │
       ┌────▼────┐
       │ domain  │
       └─────────┘
```

* Infra depends on domain
* Services depend on domain
* Domain depends on nothing

---

## Further Reading

* [Dependency Inversion (DIP)](../clean-code/dependency-inversion.md)
* [Single Responsibility (SRP)](../clean-code/single-responsibility.md)

# Go Best Practices

This document collects general coding practices that cut across multiple principles.
It complements the [Clean Code-focused docs](./clean-code/) (e.g. DIP, ISP, DRY) with conventions for interfaces, error handling, testing, and dependency management.

---

## Interfaces

### Named vs Anonymous Interfaces

* **Anonymous interfaces** (at call-sites) are great for one-off use:

  ```go
  func PrintUser(
      ctx context.Context,
      reader interface{ GetUser(context.Context, string) (*User, error) },
      id string,
  ) error {
      u, err := reader.GetUser(ctx, id)
      if err != nil { return err }
      fmt.Println(u.Email)
      return nil
  }
  ```

* **Named interfaces** should be used when:

  * The same contract is needed in multiple places
  * The interface is exported for use outside the package
  * The contract has more than a single narrow concern

### Where to Define Interfaces

* **Define interfaces in the consumer package** (where the dependency is used).
  This keeps them minimal and tailored to the business logic.

  ```go
  // package service
  type UserRepository interface {
      GetUser(ctx context.Context, id string) (*domain.User, error)
      CreateUser(ctx context.Context, u *domain.User) error
  }
  ```

* **Providers (infra packages)** should expose concrete types only.
  They just *happen* to implement consumer-defined interfaces.
* **Exception:** If an interface is so generic and broadly useful (e.g. `io.Reader`), define it in a shared package. But this is rare.

### Other interface guidelines

* Keep interfaces **small** (ideally 1 method, like `io.Reader`).
* Use **composition** to build larger contracts:

  ```go
  type UserReadWriter interface {
      UserReader
      UserWriter
  }
  ```

---

## Error Handling

### Sentinel Errors

* Use sentinel errors for domain concerns:

  ```go
  var ErrUserNotFound = errors.New("user not found")
  ```

### Map Driver Errors → Domain Errors

* Don’t leak infrastructure errors (e.g. `sql.ErrNoRows`) to business code:

  ```go
  if errors.Is(err, sql.ErrNoRows) {
      return nil, domain.ErrUserNotFound
  }
  return nil, err
  ```

### Wrapping

* Use `fmt.Errorf("context: %w", err)` to add context while preserving the original error.

### Checking

* Use `errors.Is` and `errors.As` for comparisons and unwrapping.

---

## Testing Practices

### Only test public methods

* Test behavior **via the public API**. If you can’t, that’s a **design smell**—adjust the API or refactor internals.

#### **Example (black-box, tests public API only)**

`counter/counter.go`

```go
package counter

type Counter struct{ n int }

func New() *Counter                 { return &Counter{} }
func (c *Counter) Add(delta int)    { c.n += clamp(delta) }
func (c *Counter) Value() int       { return c.n }

func clamp(x int) int { // unexported: not tested directly
 if x < -1000 { return -1000 }
 if x > 1000  { return 1000 }
 return x
}
```

`counter/counter_test.go`

```go
package counter_test

import (
 "testing"

 "example.com/project/counter"
)

func TestCounter_Add(t *testing.T) {
 c := counter.New()
 c.Add(5000)     // exercises upper clamp through public API
 if got := c.Value(); got != 1000 {
  t.Fatalf("Value()=%d, want 1000", got)
 }
}
```

> **Don’t do this:** `package counter` tests that call `clamp` directly. If a critical path is hard to reach, refactor.

### Fakes and In-Memory Repositories

* Prefer simple in-memory fakes over mocks:

  ```go
  type FakeUserRepo struct { m map[string]*domain.User }

  func (f *FakeUserRepo) GetUser(ctx context.Context, id string) (*domain.User, error) {
      if u, ok := f.m[id]; ok { return u, nil }
      return nil, domain.ErrUserNotFound
  }
  ```

* Keeps tests fast and realistic.

### Table-Driven Tests

* Standard Go idiom:

  ```go
  tests := []struct{
      name string
      input string
      wantErr error
  }{
      {"valid", "ok", nil},
      {"missing", "none", domain.ErrUserNotFound},
  }
  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) {
          err := svc.DoSomething(tt.input)
          if !errors.Is(err, tt.wantErr) { t.Fatalf("got %v, want %v", err, tt.wantErr) }
      })
  }
  ```

### Golden File Testing

* Use `.golden` files for testing code generation, rendering, or formatting.
* Keeps expected output separate from test code, and makes large outputs manageable.
* Example pattern:

  ```go
  func TestGenerateCode(t *testing.T) {
      got := GenerateCode(SomeInput{})
      
      goldenFile := "testdata/codegen.golden"
      want, err := os.ReadFile(goldenFile)
      if err != nil {
          t.Fatalf("failed to read golden file: %v", err)
      }
      
      if !bytes.Equal(got, want) {
          t.Errorf("output mismatch\n--- got ---\n%s\n--- want ---\n%s", got, want)
      }
  }
  ```

* Tip: add a `-update` flag to rewrite golden files when expectations change:

  ```go
  var update = flag.Bool("update", false, "update .golden files")

  if *update {
      os.WriteFile(goldenFile, got, 0644)
  }
  ```

### Mocking

* Use mocks sparingly — when you truly need to assert *how* a dependency is called.
* Favor fakes/stubs for most cases.

---

## Package Structure

A clean package structure helps enforce DIP and keep dependencies flowing in the right direction.

### Suggested layout

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

### Guidelines

1. **Domain package**

   * Contains entities (structs like `User`, `Order`) and domain-level errors.
   * Errors represent *business concerns*, not infra details.
   * Example:

     ```go
     package domain

     var (
         ErrUserNotFound     = errors.New("user not found")
         ErrEmailAlreadyUsed = errors.New("email already used")
     )
     ```

2. **Service package**

   * Contains high-level business logic (`UserService`).
   * Defines the **interfaces it needs** (e.g. `UserRepository`).
   * Depends only on the `domain` package, not on infra.

3. **Repo (infra) packages**

   * Concrete implementations: SQL, MongoDB, in-memory.
   * Map **driver errors → domain errors**:

     ```go
     if errors.Is(err, sql.ErrNoRows) {
         return nil, domain.ErrUserNotFound
     }
     ```

   * Never leak driver-specific errors to service or domain.

4. **Dependency direction**

   * `service` → depends on `domain`
   * `repo` → depends on `domain`
   * `domain` → depends on nothing (pure types + errors)

This ensures the **dependency graph points inward**:

* Infra depends on domain
* Services depend on domain
* Domain depends on nothing

---

## General Go Style

* **Accept interfaces, return structs** (helps composition and testing).
* **Prefer concrete types** for:

  * Configuration
  * Domain models
  * DTOs
* **Keep packages small and cohesive**:

  * Each package should have a single responsibility.
  * Don’t create “god packages” with unrelated concerns.

---

## Key Takeaways

* Use **anonymous interfaces** only for one-off function params.
* Define **interfaces in consumers**; providers expose concretes.
* Map **infrastructure errors** to **domain-level errors**.
* Prefer **fakes** and **table-driven tests**.
* Stick to **constructor injection** (with functional options when needed).
* Keep **interfaces small**, and define them **at usage points**.
* Keep the **dependency graph pointing inward**: infra → domain, service → domain.

---

## Further Reading

* [Go Proverbs](https://go-proverbs.github.io/)
* [Effective Go](https://go.dev/doc/effective_go)
* [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
* [Testable Examples in Go](https://go.dev/blog/examples)

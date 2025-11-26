# Interfaces

Guidelines for interface design and placement in Go.

---

## Named vs Anonymous Interfaces

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

---

## Where to Define Interfaces

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

* **Exception:** If an interface is so generic and broadly useful (e.g.
  `io.Reader`), define it in a shared package. But this is rare.

---

## Composition

* Keep interfaces **small** (ideally 1 method, like `io.Reader`).
* Use **composition** to build larger contracts:

  ```go
  type UserReadWriter interface {
      UserReader
      UserWriter
  }
  ```

---

## Further Reading

* [Interface Segregation (ISP)](../clean-code/interface-segregation.md)
* [Dependency Inversion (DIP)](../clean-code/dependency-inversion.md)

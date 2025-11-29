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

## Service Patterns

### Interface Wrappers for External Clients

When integrating external services (LLM clients, APIs, SDKs), wrap them with an interface
to enable dependency injection and testing:

```go
// Define the interface your service needs
type LLMClient interface {
    GenerateResponse(ctx context.Context, prompt string) (Response, error)
}

// Compile-time interface verification
var _ LLMClient = (*OpenAIClient)(nil)

// Concrete implementation wraps the external SDK
type OpenAIClient struct {
    sdk *openai.Client
}

func (c *OpenAIClient) GenerateResponse(ctx context.Context, prompt string) (Response, error) {
    return c.sdk.CreateCompletion(ctx, prompt)
}
```

The `var _ Interface = (*Impl)(nil)` pattern catches interface mismatches at compile time
rather than runtime.

### Pipeline Orchestration with Parallel Goroutines

For multi-step pipelines with independent operations, use goroutines with result channels:

```go
func (s *PipelineService) GenerateActions(ctx context.Context, input Input) (*Results, error) {
    type result struct {
        name   string
        output Output
        err    error
    }

    resultCh := make(chan result, 3)

    // Launch independent operations in parallel
    go func() {
        out, err := s.client.GenerateLightActions(ctx, input)
        resultCh <- result{name: "light", output: out, err: err}
    }()
    go func() {
        out, err := s.client.GenerateSleepActions(ctx, input)
        resultCh <- result{name: "sleep", output: out, err: err}
    }()
    go func() {
        out, err := s.client.GenerateNutritionActions(ctx, input)
        resultCh <- result{name: "nutrition", output: out, err: err}
    }()

    // Collect results with context cancellation support
    results := &Results{}
    for i := 0; i < 3; i++ {
        select {
        case r := <-resultCh:
            if r.err != nil {
                return nil, fmt.Errorf("%s failed: %w", r.name, r.err)
            }
            // assign to results based on r.name
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
    return results, nil
}
```

Key points:
* Buffered channel sized to expected result count
* Context cancellation support in the select
* First error fails fast (or collect all errors if preferred)

---

## Further Reading

* [Dependency Inversion (DIP)](../clean-code/dependency-inversion.md)
* [Single Responsibility (SRP)](../clean-code/single-responsibility.md)

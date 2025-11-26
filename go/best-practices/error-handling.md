# Error Handling

Patterns for domain errors, error mapping, and wrapping.

---

## Sentinel Errors

Use sentinel errors for domain concerns:

```go
var ErrUserNotFound = errors.New("user not found")
```

Define domain errors in a central location (e.g. `domain/errors.go`) to ensure
consistent error handling across the codebase.

---

## Map Driver Errors → Domain Errors

Don't leak infrastructure errors (e.g. `sql.ErrNoRows`, `pgx.ErrNoRows`) to
business code:

```go
func (r *UserRepo) GetByID(ctx context.Context, id string) (*domain.User, error) {
    row, err := r.queries.GetUserByID(ctx, id)
    if err != nil {
        if err == pgx.ErrNoRows {
            return nil, domain.ErrUserNotFound
        }
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return mapUser(row), nil
}
```

This keeps domain logic insulated from storage implementation details.

---

## Wrapping with Context

Use `fmt.Errorf("context: %w", err)` to add context while preserving the
original error:

```go
if err := r.queries.UpdateUser(ctx, params); err != nil {
    return fmt.Errorf("update user %s: %w", id, err)
}
```

---

## Checking Errors

Use `errors.Is` and `errors.As` for comparisons and unwrapping:

```go
if errors.Is(err, domain.ErrUserNotFound) {
    // Handle not found case
}

var pipeErr *domain.PipelineError
if errors.As(err, &pipeErr) {
    log.Printf("pipeline step %s failed", pipeErr.Step)
}
```

---

## Custom Error Types

For errors that carry structured context, define custom types:

```go
type PipelineError struct {
    Step    string
    Cause   error
    Details map[string]any
}

func (e *PipelineError) Error() string {
    return fmt.Sprintf("pipeline step %q failed: %v", e.Step, e.Cause)
}

func (e *PipelineError) Unwrap() error {
    return e.Cause
}
```

---

## Further Reading

* [Single Responsibility (SRP)](../clean-code/single-responsibility.md) — errors
  as a distinct concern

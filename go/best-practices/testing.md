# Testing

Patterns for effective, maintainable Go tests.

---

## Only Test Public Methods

Test behavior **via the public API**. If you can't, that's a **design
smell**—adjust the API or refactor internals.

### Example (black-box, tests public API only)

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

> **Don't do this:** `package counter` tests that call `clamp` directly. If a
> critical path is hard to reach, refactor.

---

## Fakes and In-Memory Repositories

Prefer simple in-memory fakes over mocks:

```go
type FakeUserRepo struct { m map[string]*domain.User }

func (f *FakeUserRepo) GetUser(ctx context.Context, id string) (
    *domain.User, error) {
    if u, ok := f.m[id]; ok {
        return u, nil
    }
    return nil, domain.ErrUserNotFound
}
```

Keeps tests fast and realistic. Verify fakes implement the interface at
compile time:

```go
var _ UserRepository = (*FakeUserRepo)(nil)
```

---

## Table-Driven Tests

Standard Go idiom:

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
        if !errors.Is(err, tt.wantErr) {
            t.Fatalf("got %v, want %v", err, tt.wantErr)
        }
    })
}
```

---

## Golden File Testing

Use `.golden` files for testing code generation, rendering, or formatting.
Keeps expected output separate from test code, and makes large outputs
manageable.

```go
func TestGenerateCode(t *testing.T) {
    got := GenerateCode(SomeInput{})

    goldenFile := "testdata/codegen.golden"
    want, err := os.ReadFile(goldenFile)
    if err != nil {
        t.Fatalf("failed to read golden file: %v", err)
    }

    if !bytes.Equal(got, want) {
        t.Errorf("output mismatch\n--- got ---\n%s\n--- want ---\n%s",
            got, want)
    }
}
```

Tip: add an `-update` flag to rewrite golden files when expectations change:

```go
var update = flag.Bool("update", false, "update .golden files")

if *update {
    os.WriteFile(goldenFile, got, 0644)
}
```

---

## Mocking

Use mocks sparingly—when you truly need to assert *how* a dependency is
called. Favor fakes/stubs for most cases.

---

## Further Reading

* [Liskov Substitution (LSP)](../clean-code/liskov-substitution.md) — contract
  tests for multiple implementations

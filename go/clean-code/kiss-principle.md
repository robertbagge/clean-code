# KISS Principle (Keep It Simple, Stupid) in Go

## Overview

KISS: favor the **simplest thing that works**. In Go that means clear, direct code using the standard library, with minimal abstractions.

> **Pragmatic note:** If you keep **SRP, OCP, ISP, and DIP** tight, you’re \~90% there. **KISS** is the day-to-day compass that stops over-engineering before it starts.

---

## Core Concept

* Prefer **functions over interfaces** for simple ops
* Use **constants**, small helpers, and the **stdlib**
* Add complexity **only when needed** (measured by real use)
* Make errors **explicit** and predictable

---

## Scaffolding (so snippets compile)

```go
package kiss

import (
    "errors"
    "os"
)
```

---

## Example 1 — Factorial (avoid clever abstractions)

### BAD — Over-engineered

```go
type MathOperation interface {
    Execute(interface{}) interface{}
    Validate(interface{}) error
}

type FactorialCalculator struct {
    cache map[int]uint64
}

func (f *FactorialCalculator) Execute(in interface{}) interface{} {
    n := in.(int) // runtime casts, vague contracts
    if n < 0 { return nil }
    if v, ok := f.cache[n]; ok { return v }
    // … recursion/strategy/caching …
    return uint64(1) // not actually correct
}
```

### GOOD — Simple and clear

```go
var (
    ErrFactorialNegative = errors.New("factorial undefined for negative numbers")
    ErrFactorialTooLarge = errors.New("factorial too large for uint64")
)

func Factorial(n int) (uint64, error) {
    if n < 0  { return 0, ErrFactorialNegative }
    if n > 20 { return 0, ErrFactorialTooLarge } // 21! overflows uint64
    res := uint64(1)
    for i := 2; i <= n; i++ { res *= uint64(i) }
    return res, nil
}
```

Why this is KISS:

* **Typed input/output**, no reflection/casts
* **Sentinel errors** document bounds
* Straightforward loop; easy to test

---

## Example 2 — Configuration (avoid over-abstraction)

### BAD — Complex framework

```go
type ConfigSource interface {
    Get(key string) (interface{}, error)
    Set(key string, value interface{}) error
}

type ConfigManager struct {
    sources []ConfigSource
    // validators, caches, graphs, reflection…
}
```

### GOOD — Simple struct + env helpers

```go
type Config struct {
    DatabaseURL string
    Port        string
    APIKey      string // required
}

var ErrMissingAPIKey = errors.New("API_KEY is required")

func getEnv(key, def string) string {
    if v := os.Getenv(key); v != "" { return v }
    return def
}

func LoadConfig() (*Config, error) {
    c := &Config{
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/db"),
        Port:        getEnv("PORT", "8080"),
        APIKey:      os.Getenv("API_KEY"),
    }
    if c.APIKey == "" { return nil, ErrMissingAPIKey }
    return c, nil
}
```

Why this is KISS:

* One **plain struct** as the source of truth
* Tiny **helper** for defaults; explicit required field
* No registries, reflection, or plugin machinery

---

## Anti-patterns to avoid

1. **Premature abstraction** (interfaces/strategies “just in case”)
2. **Reflection-heavy helpers** where simple code works
3. **Generic frameworks** for tiny problems
4. **Hidden magic** (implicit globals, side effects)

---

## Go-specific KISS techniques

* Prefer **free functions** to trivial interfaces
* Stick to the **standard library** first
* Use **named constants** instead of magic numbers/strings
* Keep **error paths explicit** (`fmt.Errorf("...: %w", err)`)
* Add **interfaces only at the consumer** and only when you have >1 impl or need seams for tests

---

## Key Takeaways

* Start simple; let **real needs** drive complexity
* Small, typed functions beat clever abstractions
* Keep configs as **plain structs** + tiny helpers
* Make rules and bounds **explicit** with constants and sentinel errors

---

## Related Best Practices

For package structure, where to define interfaces, error placement, and testing patterns (fakes, table-driven tests, golden files), see
👉 **[best-practices.md](./best-practices.md)**

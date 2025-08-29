# DRY Principle (Don't Repeat Yourself) in Go

## Overview

DRY = **one source of truth** for knowledge (rules, constants, behavior).
In Go, keep rules centralized and reuse them via **small types, constants,
and helpers** — while resisting over-abstraction. Prefer **explicit,
readable code** that avoids copy-paste drift.

---

## Core Concept

* Centralize **business rules** (validation, calculations, policies).
* Prefer **constants** over magic numbers/strings.
* Reuse **helpers** (validators, calculators) instead of duplicating logic.
* Keep DRY scoped to **knowledge**, not just lines of code.
* Balance DRY with clarity; avoid “abstracting too early.”

---

## Scaffolding (so snippets compile)

```go
package users

import (
    "context"
    "database/sql"
    "encoding/json"
    "errors"
    "net/http"
    "regexp"
    "strconv"
)

type UserData struct {
    Email    string
    Password string
    Age      int
}

// Single source of truth (constants)
const (
    EmailMinLength    = 5
    PasswordMinLength = 8
    MinimumAge        = 18
)

// Sentinel errors (domain-level)
var (
    ErrInvalidEmail    = errors.New("invalid email")
    ErrInvalidPassword = errors.New("password must be 8+ chars with uppercase")
    ErrAgeTooYoung     = errors.New("must be 18 or older")
)
```

---

## BAD — Repeated validation logic (copy-paste drift)

```go
// Same checks repeated in multiple handlers; rules will diverge over time.
func (uc *UserController) RegisterUser(data map[string]string) error {
    emailValid := len(data["email"]) >= 5 &&
        regexp.MustCompile(`@`).MatchString(data["email"])
    if !emailValid {
        return ErrInvalidEmail
    }
    pwValid := len(data["password"]) >= 8 &&
        regexp.MustCompile(`[A-Z]`).MatchString(data["password"])
    if !pwValid {
        return ErrInvalidPassword
    }
    age, _ := strconv.Atoi(data["age"])
    if age < 18 {
        return ErrAgeTooYoung
    }
    return nil
}

func (uc *UserController) UpdateProfile(_ string, data map[string]string) error {
    if email := data["email"]; email != "" {
        emailValid := len(email) >= 5 &&
            regexp.MustCompile(`@`).MatchString(email)
        if !emailValid {
            return ErrInvalidEmail
        }
    }
    if age, ok := data["age"]; ok {
        v, _ := strconv.Atoi(age)
        if v < 18 {
            return ErrAgeTooYoung
        }
    }
    return nil
}
```

---

## GOOD — DRY validation (single source + policy for create/update)

```go
// Policy clarifies required vs optional fields.
type UserPolicy int
const (
    PolicyCreate UserPolicy = iota
    PolicyUpdate
)

type Validator struct {
    emailRegex    *regexp.Regexp
    passwordRegex *regexp.Regexp
}

func NewValidator() *Validator {
    return &Validator{
        // compiled once
        emailRegex:    regexp.MustCompile(`^[^@]+@[^@]+\.[^@]+$`),
        passwordRegex: regexp.MustCompile(`[A-Z]`),
    }
}

func (v *Validator) ValidateEmail(email string) error {
    valid := len(email) >= EmailMinLength &&
        v.emailRegex.MatchString(email)
    if !valid {
        return ErrInvalidEmail
    }
    return nil
}

func (v *Validator) ValidatePassword(pw string) error {
    valid := len(pw) >= PasswordMinLength &&
        v.passwordRegex.MatchString(pw)
    if !valid {
        return ErrInvalidPassword
    }
    return nil
}

func (v *Validator) ValidateAge(age int) error {
    if age < MinimumAge {
        return ErrAgeTooYoung
    }
    return nil
}

// One entrypoint = one source of truth for user validation.
func (v *Validator) ValidateUser(
    u UserData, policy UserPolicy,
) error {
    if policy == PolicyCreate || u.Email != "" {
        if err := v.ValidateEmail(u.Email); err != nil {
            return err
        }
    }
    if policy == PolicyCreate || u.Password != "" {
        if err := v.ValidatePassword(u.Password); err != nil {
            return err
        }
    }
    if policy == PolicyCreate || u.Age != 0 {
        if err := v.ValidateAge(u.Age); err != nil {
            return err
        }
    }
    return nil
}

// Controller uses validator; maps errors predictably with errors.Is.
type UserController struct {
    validator *Validator
    db        *sql.DB
}

func (uc *UserController) RegisterUser(
    ctx context.Context, u UserData, w http.ResponseWriter,
) {
    if err := uc.validator.ValidateUser(u, PolicyCreate); err != nil {
        status := http.StatusBadRequest
        switch {
        case errors.Is(err, ErrInvalidEmail),
             errors.Is(err, ErrInvalidPassword),
             errors.Is(err, ErrAgeTooYoung):
            // keep 400
        default:
            status = http.StatusInternalServerError
        }
        http.Error(w, err.Error(), status)
        return
    }

    _, err := uc.db.ExecContext(ctx,
        "INSERT INTO users(email,password,age) VALUES($1,$2,$3)",
        u.Email, u.Password, u.Age)
    if err != nil {
        http.Error(w, "database error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    _ = json.NewEncoder(w).Encode(map[string]any{"ok": true})
}

func (uc *UserController) UpdateProfile(
    ctx context.Context, u UserData, w http.ResponseWriter,
) {
    if err := uc.validator.ValidateUser(u, PolicyUpdate); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    // ... apply partial updates ...
    _ = json.NewEncoder(w).Encode(map[string]any{"ok": true})
}
```

### Why this is DRY

* All rules live in **one place** (`Validator` + constants).
* **Create vs update** uses a **policy**, not duplicated branches.
* Callers use `errors.Is` for consistent error handling.

---

## Anti-patterns to Avoid

1. **Copy-paste programming** — extract helpers instead.
2. **Magic numbers/strings** — use **named constants**.
3. **Scattered regex/SQL snippets** — centralize patterns and queries.
4. **Over-DRY** — don’t hide logic behind generic abstractions that reduce clarity.

---

## Go-Specific DRY Techniques

### **Interfaces for shared behavior**

```go
type Validatable interface {
    Validate(context.Context) error
}

func ValidateAny(ctx context.Context, v Validatable) error {
    return v.Validate(ctx)
}
```

### **Embedding common fields**

```go
type BaseEntity struct {
    ID        string
    CreatedAt int64
    UpdatedAt int64
}
type User struct {
    BaseEntity
    Email string
}
```

### **Generics for reusable helpers** (Go 1.18+)

```go
func FindByID[T any](
    xs []T, want string, idOf func(T) string,
) *T {
    for i := range xs {
        if idOf(xs[i]) == want {
            return &xs[i]
        }
    }
    return nil
}
```

### **Centralized configuration/constants**

* Put thresholds, fees, hosts, etc., in one config/consts module.
* Map **driver errors → domain errors** in one place.

---

## Key Takeaways

* DRY the **knowledge** (rules/constants), not just code lines.
* Use **one validator** with **policies** for different workflows.
* Compile heavy artifacts (regex) **once**.
* Prefer **explicit, readable** helpers over premature abstraction.
* Check with `errors.Is` for uniform handling.

---

## Related Best Practices

For package structure, where to define interfaces, error placement,
and testing patterns (fakes, table-driven tests, golden files), see
👉 **[best-practices.md](../best-practices.md)**

# Single Responsibility Principle (SRP) in Go

## Overview

SRP: a type/function/package should have **one reason to change**. In Go this
falls out of **small packages, tiny interfaces, and composition**.

---

## Core Concept

* **Functions** do one job.
* **Types** model one concept.
* **Packages** focus on one theme.
* Split **transport / validation / business / persistence** concerns.

---

## Scaffolding (so snippets compile)

```go
package products

import (
    "context"
    "database/sql"
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "strconv"
)

var (
    ErrInvalidMinPrice = errors.New("invalid min_price")
    ErrDB              = errors.New("database error")
)

type Product struct {
    ID       int
    Name     string
    Price    float64
    Category string
}
```

---

## BAD — One handler, many responsibilities

```go
// Mixed concerns: HTTP parsing, validation, SQL building, DB IO, JSON rendering.
func (h *ProductHandler) GetProducts(w http.ResponseWriter, r *http.Request) {
    category := r.URL.Query().Get("category")
    minStr := r.URL.Query().Get("min_price")

    var min float64
    if minStr != "" {
        v, _ := strconv.ParseFloat(minStr, 64) // no validation
        min = v
    }

    // Ad-hoc SQL building
    query := "SELECT id,name,price,category FROM products WHERE 1=1"
    if category != "" {
        query += " AND category = '" + category + "'" // injection risk
    }
    if min > 0 {
        query += fmt.Sprintf(" AND price >= %f", min)
    }

    rows, err := h.db.Query(query)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()

    var out []Product
    for rows.Next() {
        var p Product
        rows.Scan(&p.ID, &p.Name, &p.Price, &p.Category)
        out = append(out, p)
    }
    w.Header().Set("Content-Type", "application/json")
    _ = json.NewEncoder(w).Encode(out)
}
```

---

## GOOD — Separated concerns (SRP-friendly)

```go
// --- Transport/validation concern ---
type ProductFilter struct {
    Category       string
    MinPrice, MaxPrice float64
}

func ParseProductFilter(r *http.Request) (ProductFilter, error) {
    f := ProductFilter{Category: r.URL.Query().Get("category")}

    if s := r.URL.Query().Get("min_price"); s != "" {
        v, err := strconv.ParseFloat(s, 64)
        if err != nil || v < 0 { return ProductFilter{}, ErrInvalidMinPrice }
        f.MinPrice = v
    }
    if s := r.URL.Query().Get("max_price"); s != "" {
        v, err := strconv.ParseFloat(s, 64)
        if err == nil && v > 0 { f.MaxPrice = v }
    }
    return f, nil
}

// --- Persistence concern (DIP-ready) ---
type ProductReader interface {
    Find(ctx context.Context, f ProductFilter) ([]Product, error)
}

type SQLProductRepository struct{ db *sql.DB }

func (r *SQLProductRepository) Find(ctx context.Context, f ProductFilter) (
    []Product, error) {
    q := "SELECT id,name,price,category FROM products WHERE 1=1"
    var args []any
    i := 1

    if f.Category != "" {
        q += fmt.Sprintf(" AND category = $%d", i)
        args = append(args, f.Category)
        i++
    }
    if f.MinPrice > 0 {
        q += fmt.Sprintf(" AND price >= $%d", i)
        args = append(args, f.MinPrice)
        i++
    }
    if f.MaxPrice > 0 {
        q += fmt.Sprintf(" AND price <= $%d", i)
        args = append(args, f.MaxPrice)
        i++
    }

    rows, err := r.db.QueryContext(ctx, q, args...)
    if err != nil { return nil, fmt.Errorf("%w: %v", ErrDB, err) }
    defer rows.Close()

    var out []Product
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Category); err != nil {
            return nil, err
        }
        out = append(out, p)
    }
    return out, rows.Err()
}

// --- Business concern ---
type ProductService struct{ repo ProductReader }

func NewProductService(repo ProductReader) *ProductService {
    return &ProductService{repo: repo}
}

func (s *ProductService) List(ctx context.Context, f ProductFilter) (
    []Product, error) {
    if f.MaxPrice > 0 && f.MaxPrice < f.MinPrice {
        return nil, ErrInvalidMinPrice // business rule lives here
    }
    return s.repo.Find(ctx, f)
}

// --- HTTP concern ---
type ProductHandler struct{ svc *ProductService }

func NewProductHandler(svc *ProductService) *ProductHandler {
    return &ProductHandler{svc: svc}
}

func (h *ProductHandler) GetProducts(w http.ResponseWriter, r *http.Request) {
    f, err := ParseProductFilter(r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    items, err := h.svc.List(r.Context(), f)
    if err != nil {
        code := http.StatusInternalServerError
        if errors.Is(err, ErrInvalidMinPrice) {
            code = http.StatusBadRequest
        }
        http.Error(w, err.Error(), code)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    _ = json.NewEncoder(w).Encode(items)
}
```

### Why this is SRP-friendly

* `ParseProductFilter` only parses/validates **HTTP inputs**.
* `ProductService` holds **business rules**.
* `SQLProductRepository` does **data access** only.
* `ProductHandler` handles **HTTP I/O** only.

---

## Testing SRP (lightweight pattern)

```go
// Fake repo for handler/service tests.
type FakeRepo struct{ out []Product; err error }
func (f *FakeRepo) Find(
    ctx context.Context, _ ProductFilter,
) ([]Product, error) {
    return f.out, f.err
}

// Example idea:
//  - NewProductService(&FakeRepo{out: []Product{{ID:1}}})
//  - NewProductHandler(svc)
//  - Call GetProducts with httptest; assert status/code/body.
//  - Swap FakeRepo.err to simulate DB error without touching handler/service code.
```

---

## Anti-patterns to Avoid

1. **God handlers/services** that mix transport, validation, business, and persistence.
2. **Cross-layer coupling** (e.g., HTTP types in repository signatures).
3. **Multiple reasons to change** inside one type/file/package.

---

## Go-Specific SRP Techniques

* **Package boundaries** enforce separation (domain / service / repo / http).
* **Tiny interfaces** (consumer-defined) to keep layers independent.
* **Composition over inheritance** for feature growth.
* **Constructor injection** to wire layers cleanly.

---

## Key Takeaways

* Give each function/type/package **one clear purpose**.
* Split **HTTP / validation / business / persistence**.
* Define **small interfaces** at the consumer; inject implementations.
* Test each concern **independently** with fakes.

---

## Related Best Practices

For package structure, where to define interfaces, error placement, and
testing patterns (fakes, table-driven tests, golden files), see
👉 **[best-practices.md](../best-practices.md)**

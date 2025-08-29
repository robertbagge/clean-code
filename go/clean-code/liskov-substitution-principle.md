# Liskov Substitution Principle (LSP) in Go

## Overview

LSP in Go is about **behavioral substitutability across interface
implementations** (not class hierarchies). If a type implements an interface,
you should be able to use it anywhere that interface is expected
**without breaking assumptions**.

> **Pragmatic note:** In most Go codebases, if you get **SRP, OCP, ISP,
> and DIP** right, you're \~90% there. LSP largely **falls out** of small,
> consumer-defined interfaces and clear contracts. Add explicit LSP/contract
> tests **selectively** for critical, multi-implementation interfaces.

---

## Core Concept

* Implementations must honor the **interface’s behavioral contract**.
* Don’t add **stricter preconditions** than the contract implies.
* Don’t return **weaker postconditions** than the contract promises.
* Keep **error semantics consistent** across implementations.
* Prefer **small, focused interfaces** (pairs well with ISP).

---

## Scaffolding (so snippets compile)

```go
package payments

import (
    "context"
    "errors"
    "fmt"
    "time"
)

var (
    ErrInvalidAmount   = errors.New("invalid amount")
    ErrOutOfRange      = errors.New("amount out of range")
    ErrMinimumTransfer = errors.New("amount below minimum transfer")
)

type PaymentResult struct {
    TransactionID string
    Status        string // "completed", "pending", "failed"
}

func generateID() string {
    return fmt.Sprintf("tx_%d", time.Now().UnixNano())
}
```

---

## Examples

### BAD — Behavioral inconsistency (violates LSP)

```go
type PaymentProcessor interface {
    // Vague: callers may assume "process now or error".
    ProcessPayment(amount float64) error
}

// Immediate settlement
type CreditCardProcessor struct{}
func (c *CreditCardProcessor) ProcessPayment(amount float64) error {
    if amount <= 0 {
        return ErrInvalidAmount
    }
    return nil // completed now
}

// Stricter preconditions + silent queuing (surprise!)
type BankTransferProcessor struct{}
func (b *BankTransferProcessor) ProcessPayment(amount float64) error {
    if amount < 100 {
        return ErrMinimumTransfer // hidden constraint
    }
    return nil // queued, not actually completed
}
```

### GOOD — Explicit, consistent contract (LSP-friendly)

```go
// Document behavior on the interface.
type PaymentProcessor interface {
    // ProcessPayment MUST:
    // - Validate amount against min/max; return ErrOutOfRange if invalid.
    // - Return PaymentResult with Status in
    //   {"completed","pending","failed"}.
    // - For async flows, return Status="pending" and a TransactionID.
    ProcessPayment(
        ctx context.Context,
        amount float64,
    ) (*PaymentResult, error)

    GetMinimumAmount() float64
    GetMaximumAmount() float64
}

// Immediate settlement
type CreditCardProcessor struct{}

func (c *CreditCardProcessor) ProcessPayment(
    ctx context.Context,
    amount float64,
) (*PaymentResult, error) {
    if amount < c.GetMinimumAmount() || amount > c.GetMaximumAmount() {
        return nil, ErrOutOfRange
    }
    return &PaymentResult{
        TransactionID: generateID(),
        Status:        "completed",
    }, nil
}
func (c *CreditCardProcessor) GetMinimumAmount() float64 {
    return 0.01
}
func (c *CreditCardProcessor) GetMaximumAmount() float64 {
    return 10_000
}

// Asynchronous settlement (explicit via "pending")
type BankTransferProcessor struct{}

func (b *BankTransferProcessor) ProcessPayment(
    ctx context.Context,
    amount float64,
) (*PaymentResult, error) {
    if amount < b.GetMinimumAmount() || amount > b.GetMaximumAmount() {
        return nil, ErrOutOfRange
    }
    return &PaymentResult{
        TransactionID: generateID(),
        Status:        "pending",
    }, nil
}
func (b *BankTransferProcessor) GetMinimumAmount() float64 {
    return 100
}
func (b *BankTransferProcessor) GetMaximumAmount() float64 {
    return 1_000_000
}

// (Optional compile-time checks)
// var _ PaymentProcessor = (*CreditCardProcessor)(nil)
// var _ PaymentProcessor = (*BankTransferProcessor)(nil)
```

---

## Testing LSP (behavioral compatibility)

Keep one **contract test** that runs the same scenarios across all
implementations.

```go
// Minimal contract test
func ContractTestProcessor(
    t TestingT,
    name string,
    mk func() PaymentProcessor,
) {
    t.Run(name, func(t *testing.T) {
        p := mk()
        ctx := context.Background()

        // Out-of-range -> ErrOutOfRange
        _, err := p.ProcessPayment(ctx, p.GetMinimumAmount()-0.01)
        if !errors.Is(err, ErrOutOfRange) {
            t.Fatalf("want ErrOutOfRange, got %v", err)
        }

        // Min amount -> valid status + transaction id
        res, err := p.ProcessPayment(ctx, p.GetMinimumAmount())
        if err != nil {
            t.Fatal(err)
        }
        switch res.Status {
        case "completed", "pending", "failed":
        default:
            t.Fatalf("invalid status: %q", res.Status)
        }
        if res.TransactionID == "" {
            t.Fatal("missing TransactionID")
        }
    })
}
```

### Pragmatic scope

Add contract/LSP tests **only when it pays off**:

* Public or widely-used interfaces; **≥2 implementations**
  (sql/mongo/memory; stripe/paypal).
* Correctness-critical (money/auth/security), or semantics like
  **async/retries** must be uniform.
* Skip for internal, single-implementation helpers where normal unit
  tests suffice.

---

## Key Takeaways

* LSP in Go = **consistent behavior across interface impls**.
* **Document the contract**, keep interfaces small, align error
  semantics.
* Use **capability discovery** (e.g., min/max) instead of hidden
  preconditions.
* Rely primarily on SRP/OCP/ISP/DIP; add **contract tests
  selectively**.

---

## Related Best Practices

For package structure, where to define interfaces, error placement,
and testing patterns (fakes, table-driven tests, golden files), see
👉 **[best-practices.md](../best-practices.md)**

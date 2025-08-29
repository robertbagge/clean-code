# Liskov Substitution Principle (LSP)

## Overview

Subtypes must be substitutable for their base types without
altering correctness. If S is a subtype of T, objects of
type T should be replaceable with objects of type S
without breaking the program.

## Core Concept

- Derived classes must honor the base class contract
- Don't strengthen preconditions or weaken postconditions
- Preserve expected behavior, not just method signatures
- Avoid type checking to determine behavior

## Example

### Scaffolding

```typescript
interface Account {
  balance: number;
}

interface PaymentResult {
  success: boolean;
  message: string;
}
```

### BAD — Breaking behavioral contracts

```typescript
class PaymentProcessor {
  processPayment(amount: number, account: Account): PaymentResult {
    if (account.balance >= amount) {
      account.balance -= amount; // Always deducts immediately
      return { success: true, message: 'Payment completed' };
    }
    return { success: false, message: 'Insufficient funds' };
  }
}

class DelayedPaymentProcessor extends PaymentProcessor {
  processPayment(amount: number, account: Account): PaymentResult {
    // VIOLATION: Changes behavior - doesn't actually deduct funds
    if (account.balance >= amount) {
      this.schedulePayment(amount, account); // Just schedules!
      return { success: true, message: 'Payment scheduled' };
    }
    return { success: false, message: 'Insufficient funds' };
  }
  
  private schedulePayment(amount: number, account: Account): void {
    // Funds not actually deducted
  }
}

// Client code expects funds to be deducted
function processOrder(processor: PaymentProcessor, account: Account): void {
  const result = processor.processPayment(100, account);
  
  if (result.success) {
    shipProduct(); // Ships immediately but DelayedProcessor didn't actually charge!
  }
}
```

### GOOD — Preserving contracts

```typescript
interface PaymentStrategy {
  canProcess(account: Account): boolean;
  execute(amount: number, account: Account): PaymentResult;
}

class ImmediatePayment implements PaymentStrategy {
  canProcess(account: Account): boolean {
    return account.balance >= 0;
  }
  
  execute(amount: number, account: Account): PaymentResult {
    if (account.balance >= amount) {
      account.balance -= amount;
      return { success: true, message: 'Payment completed' };
    }
    return { success: false, message: 'Insufficient funds' };
  }
}

class CreditPayment implements PaymentStrategy {
  canProcess(account: Account): boolean {
    return true; // Credit allows negative balance
  }
  
  execute(amount: number, account: Account): PaymentResult {
    account.balance -= amount; // Still deducts, maintaining contract
    return { success: true, message: 'Payment completed on credit' };
  }
}

// Separate interface for scheduling - different contract entirely
interface PaymentScheduler {
  schedule(amount: number, account: Account): PaymentResult;
}

class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}
  
  processPayment(amount: number, account: Account): PaymentResult {
    if (!this.strategy.canProcess(account)) {
      return { success: false, message: 'Payment method unavailable' };
    }
    return this.strategy.execute(amount, account);
  }
}
```

---

## Anti-patterns to Avoid

1. **Unexpected exceptions** in derived classes
2. **Stronger preconditions** than base class
3. **Weaker postconditions** than base class promises

---

## Key Takeaways

- Design **behavior-preserving** hierarchies
- Use **composition** when behavior differs significantly
- Make contracts **explicit** through interfaces
- Subtypes should **extend**, not alter base behavior
- If you need `instanceof` checks, reconsider your design

# Liskov Substitution Principle (LSP)

## Overview

Subtypes must be substitutable for their base types without altering correctness. If S is a subtype of T, objects of type T should be replaceable with objects of type S without breaking the program.

## Core Concept

- Derived classes must honor the base class contract
- Don't strengthen preconditions or weaken postconditions
- Preserve expected behavior, not just method signatures
- Avoid type checking to determine behavior

---

## Scaffolding

```typescript
// Types for payment processing examples
interface PaymentResult {
  success: boolean;
  transactionId?: string;
  errorMessage?: string;
}

interface Account {
  balance: number;
  frozen: boolean;
}
```

---

## BAD — Breaking behavioral contracts

```typescript
class PaymentProcessor {
  processPayment(amount: number, account: Account): PaymentResult {
    if (account.balance >= amount) {
      account.balance -= amount;
      return { success: true, transactionId: this.generateId() };
    }
    return { success: false, errorMessage: 'Insufficient funds' };
  }
  
  protected generateId(): string {
    return Date.now().toString();
  }
}

class RefundablePaymentProcessor extends PaymentProcessor {
  processPayment(amount: number, account: Account): PaymentResult {
    // VIOLATION: Parent succeeds with sufficient balance,
    // but child adds extra precondition
    if (account.frozen) {
      throw new Error('Cannot process payment on frozen account');
    }
    
    // VIOLATION: Parent deducts immediately,
    // but child only reserves funds
    if (account.balance >= amount) {
      // Different behavior - doesn't actually deduct!
      this.reserveFunds(account, amount);
      return { success: true, transactionId: 'PENDING_' + this.generateId() };
    }
    return { success: false, errorMessage: 'Insufficient funds' };
  }
  
  private reserveFunds(account: Account, amount: number): void {
    // Just marks as pending, doesn't deduct
  }
}

// Client code expects consistent behavior
function processOrder(processor: PaymentProcessor, account: Account): void {
  const result = processor.processPayment(100, account);
  
  if (result.success) {
    // Assumes payment is complete and funds are deducted
    shipProduct(); // But RefundablePaymentProcessor didn't actually charge!
  }
}
```

---

## GOOD — Preserving contracts

```typescript
// Define clear abstractions with consistent contracts
interface PaymentStrategy {
  canProcess(account: Account): boolean;
  execute(amount: number, account: Account): PaymentResult;
}

class StandardPayment implements PaymentStrategy {
  canProcess(account: Account): boolean {
    return !account.frozen;
  }
  
  execute(amount: number, account: Account): PaymentResult {
    if (account.balance >= amount) {
      account.balance -= amount;
      return { 
        success: true, 
        transactionId: Date.now().toString() 
      };
    }
    return { 
      success: false, 
      errorMessage: 'Insufficient funds' 
    };
  }
}

class RefundablePayment implements PaymentStrategy {
  private refundWindow = 30; // days
  
  canProcess(account: Account): boolean {
    return !account.frozen;
  }
  
  execute(amount: number, account: Account): PaymentResult {
    if (account.balance >= amount) {
      // Still deducts immediately, maintaining contract
      account.balance -= amount;
      const transactionId = Date.now().toString();
      
      // Additional capability: stores refund eligibility
      this.storeRefundEligibility(transactionId, this.refundWindow);
      
      return { success: true, transactionId };
    }
    return { 
      success: false, 
      errorMessage: 'Insufficient funds' 
    };
  }
  
  private storeRefundEligibility(id: string, days: number): void {
    // Store refund window metadata
  }
}

// Separate concern for deferred payments
class DeferredPayment implements PaymentStrategy {
  canProcess(account: Account): boolean {
    // Clearly different contract, not a subtype
    return !account.frozen && account.balance >= 0;
  }
  
  execute(amount: number, account: Account): PaymentResult {
    // Different behavior is explicit via interface choice
    if (this.canSchedule(account, amount)) {
      const reservationId = this.reserveFunds(account, amount);
      return { 
        success: true, 
        transactionId: 'DEFERRED_' + reservationId 
      };
    }
    return { 
      success: false, 
      errorMessage: 'Cannot schedule payment' 
    };
  }
  
  private canSchedule(account: Account, amount: number): boolean {
    return account.balance >= amount * 0.1; // 10% reserve
  }
  
  private reserveFunds(account: Account, amount: number): string {
    // Reserve logic
    return Date.now().toString();
  }
}

// Client code works consistently with any strategy
class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}
  
  processPayment(amount: number, account: Account): PaymentResult {
    if (!this.strategy.canProcess(account)) {
      return { 
        success: false, 
        errorMessage: 'Account cannot be processed' 
      };
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
# Dependency Inversion Principle (DIP)

## Overview

High-level modules should not depend on low-level modules.
Both should depend on abstractions.
This inverts traditional dependency flow, making systems
more flexible and testable.

## Core Concept

- Depend on abstractions (interfaces), not concrete classes
- High-level code defines interfaces it needs
- Low-level code implements those interfaces
- Use dependency injection to provide implementations

---

## Scaffolding

```typescript
interface Order {
  id: string;
  total: number;
  email: string;
}
```

---

## BAD — Direct dependency on concrete implementations

```typescript
class OrderService {
  private database: PostgreSQLDatabase;
  private emailer: GmailService;
  
  constructor() {
    // Creating concrete dependencies inside high-level module
    this.database = new PostgreSQLDatabase();
    this.emailer = new GmailService();
  }
  
  async processOrder(order: Order): Promise<void> {
    // Business logic tied to concrete implementations
    await this.database.query('INSERT INTO orders...', [order.id]);
    await this.emailer.sendEmail(order.email, 'Order confirmed');
  }
}

class PostgreSQLDatabase {
  async query(sql: string, params: any[]): Promise<void> {
    // PostgreSQL specific code
  }
}

class GmailService {
  async sendEmail(to: string, message: string): Promise<void> {
    // Gmail specific code
  }
}
```

---

## GOOD — Both depend on abstractions

```typescript
// High-level code defines interfaces it needs
interface OrderRepository {
  save(order: Order): Promise<void>;
}

interface NotificationService {
  notify(email: string, message: string): Promise<void>;
}

// High-level module depends on abstractions
class OrderService {
  constructor(
    private repository: OrderRepository,
    private notifier: NotificationService
  ) {}
  
  async processOrder(order: Order): Promise<void> {
    // Business logic uses abstractions
    await this.repository.save(order);
    await this.notifier.notify(order.email, 'Order confirmed');
  }
}

// Low-level modules implement the abstractions
class PostgreSQLOrderRepository implements OrderRepository {
  async save(order: Order): Promise<void> {
    await db.query('INSERT INTO orders...', [order.id, order.total]);
  }
}

class EmailNotificationService implements NotificationService {
  async notify(email: string, message: string): Promise<void> {
    await emailClient.send(email, message);
  }
}

// Dependency injection - easy to swap implementations
const repository = new PostgreSQLOrderRepository();
const notifier = new EmailNotificationService();
const service = new OrderService(repository, notifier);
```

---

## Anti-patterns to Avoid

1. **Creating dependencies inside constructors** instead of injecting them
2. **Using concrete types** in method parameters instead of interfaces
3. **Static dependencies** that can't be mocked or replaced

---

## Key Takeaways

- **Abstractions define contracts** between layers
- **High-level policies** aren't affected by low-level changes
- **Testing becomes trivial** with mock implementations
- **Swapping implementations** requires no business logic changes
- **Dependency injection** wires everything together

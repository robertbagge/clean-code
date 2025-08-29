# Dependency Inversion Principle (DIP)

## Overview

High-level modules should not depend on low-level modules. Both should depend on abstractions. This inverts traditional dependency flow, making systems more flexible and testable.

## Core Concept

- Depend on abstractions (interfaces), not concrete classes
- High-level code defines interfaces it needs
- Low-level code implements those interfaces
- Use dependency injection to provide implementations

---

## Scaffolding

```typescript
// Domain types for examples
interface Order {
  id: string;
  items: Array<{ productId: string; quantity: number }>;
  total: number;
  customerEmail: string;
}

interface NotificationResult {
  success: boolean;
  messageId?: string;
}
```

---

## BAD — Direct dependency on concrete implementations

```typescript
// High-level business logic directly depends on low-level details
class OrderService {
  private database: PostgreSQLDatabase;
  private emailer: GmailService;
  
  constructor() {
    // Creating concrete dependencies inside high-level module
    this.database = new PostgreSQLDatabase('localhost', 5432);
    this.emailer = new GmailService('smtp.gmail.com', 587);
  }
  
  async processOrder(order: Order): Promise<void> {
    // Business logic tied to PostgreSQL specifics
    const connection = await this.database.connect();
    await connection.query(
      'INSERT INTO orders (id, total, email) VALUES ($1, $2, $3)',
      [order.id, order.total, order.customerEmail]
    );
    
    // Business logic tied to Gmail specifics
    await this.emailer.authenticate('user@gmail.com', 'password');
    await this.emailer.sendEmail({
      to: order.customerEmail,
      subject: 'Order Confirmation',
      body: `Your order ${order.id} has been processed`
    });
  }
}

// Low-level modules (concrete implementations)
class PostgreSQLDatabase {
  constructor(private host: string, private port: number) {}
  
  async connect() {
    // PostgreSQL specific connection logic
    return pgClient.connect(this.host, this.port);
  }
}

class GmailService {
  constructor(private smtp: string, private port: number) {}
  
  async authenticate(user: string, pass: string) {
    // Gmail specific auth
  }
  
  async sendEmail(details: any) {
    // Gmail specific sending
  }
}
```

---

## GOOD — Both depend on abstractions

```typescript
// Define abstractions (interfaces) that high-level code needs
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

interface NotificationService {
  notify(recipient: string, message: string): Promise<NotificationResult>;
}

// High-level module depends on abstractions
class OrderService {
  constructor(
    private repository: OrderRepository,  // Abstraction
    private notifier: NotificationService  // Abstraction
  ) {}
  
  async processOrder(order: Order): Promise<void> {
    // Business logic uses abstractions, not concrete implementations
    await this.repository.save(order);
    
    const result = await this.notifier.notify(
      order.customerEmail,
      `Your order ${order.id} has been processed`
    );
    
    if (!result.success) {
      throw new Error('Failed to send notification');
    }
  }
}

// Low-level modules implement the abstractions
class PostgreSQLOrderRepository implements OrderRepository {
  constructor(private connectionString: string) {}
  
  async save(order: Order): Promise<void> {
    const client = await pg.connect(this.connectionString);
    await client.query(
      'INSERT INTO orders (id, total, email) VALUES ($1, $2, $3)',
      [order.id, order.total, order.customerEmail]
    );
  }
  
  async findById(id: string): Promise<Order | null> {
    const client = await pg.connect(this.connectionString);
    const result = await client.query('SELECT * FROM orders WHERE id = $1', [id]);
    return result.rows[0] || null;
  }
}

class MongoOrderRepository implements OrderRepository {
  constructor(private uri: string) {}
  
  async save(order: Order): Promise<void> {
    const client = await MongoClient.connect(this.uri);
    await client.db('shop').collection('orders').insertOne(order);
  }
  
  async findById(id: string): Promise<Order | null> {
    const client = await MongoClient.connect(this.uri);
    return await client.db('shop').collection('orders').findOne({ id });
  }
}

class EmailNotificationService implements NotificationService {
  constructor(private apiKey: string) {}
  
  async notify(recipient: string, message: string): Promise<NotificationResult> {
    const response = await emailApi.send({
      to: recipient,
      subject: 'Order Update',
      body: message,
      apiKey: this.apiKey
    });
    
    return {
      success: response.status === 200,
      messageId: response.messageId
    };
  }
}

class SmsNotificationService implements NotificationService {
  constructor(private accountSid: string, private authToken: string) {}
  
  async notify(recipient: string, message: string): Promise<NotificationResult> {
    const response = await smsClient.messages.create({
      to: recipient,
      body: message
    });
    
    return {
      success: !!response.sid,
      messageId: response.sid
    };
  }
}

// Dependency injection - easy to swap implementations
const repository = new PostgreSQLOrderRepository('postgresql://localhost/shop');
const notifier = new EmailNotificationService('api-key-123');
const orderService = new OrderService(repository, notifier);

// Testing is simple with test doubles
const testRepo = new InMemoryOrderRepository();
const testNotifier = new MockNotificationService();
const testService = new OrderService(testRepo, testNotifier);
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
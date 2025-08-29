# Single Responsibility Principle (SRP)

## Overview

A class/module/function should have **one reason to change**. This improves testability, reduces coupling, and makes code easier to understand and maintain.

## Core Concept

- Each unit of code does one job well
- Changes to one responsibility don't affect others
- High cohesion within modules, low coupling between them
- Separate different concerns into different modules

---

## Scaffolding

```typescript
// Domain types and helpers for examples
interface User {
  id: string;
  email: string;
  name: string;
}

interface EmailMessage {
  to: string;
  subject: string;
  body: string;
}

class ValidationError extends Error {}
```

---

## BAD — Mixed responsibilities

```typescript
class UserService {
  async createUser(email: string, name: string): Promise<User> {
    // Validation logic mixed in
    if (!email.includes('@')) {
      throw new ValidationError('Invalid email');
    }
    if (name.length < 2) {
      throw new ValidationError('Name too short');
    }
    
    // Database operations mixed in
    const user = { id: Date.now().toString(), email, name };
    await db.query('INSERT INTO users VALUES (?, ?, ?)', [user.id, email, name]);
    
    // Email sending mixed in
    const smtp = await connectSmtp('smtp.example.com', 587);
    await smtp.send({
      to: email,
      subject: 'Welcome!',
      body: `Hi ${name}, welcome to our service!`
    });
    
    // Audit logging mixed in
    await fs.appendFile('audit.log', `User created: ${user.id}\n`);
    
    return user;
  }
}
```

---

## GOOD — Separated concerns

```typescript
// Each class has one responsibility
class UserValidator {
  validate(email: string, name: string): void {
    if (!email.includes('@')) {
      throw new ValidationError('Invalid email');
    }
    if (name.length < 2) {
      throw new ValidationError('Name too short');
    }
  }
}

class UserRepository {
  async save(user: User): Promise<void> {
    await db.query('INSERT INTO users VALUES (?, ?, ?)', 
      [user.id, user.email, user.name]);
  }
}

class EmailService {
  async sendWelcome(user: User): Promise<void> {
    await this.smtp.send({
      to: user.email,
      subject: 'Welcome!',
      body: `Hi ${user.name}, welcome to our service!`
    });
  }
}

class AuditLogger {
  async logUserCreation(userId: string): Promise<void> {
    await fs.appendFile('audit.log', `User created: ${userId}\n`);
  }
}

// Orchestrator coordinates single-purpose services
class UserService {
  constructor(
    private validator: UserValidator,
    private repository: UserRepository,
    private emailService: EmailService,
    private logger: AuditLogger
  ) {}

  async createUser(email: string, name: string): Promise<User> {
    this.validator.validate(email, name);
    
    const user = { id: Date.now().toString(), email, name };
    await this.repository.save(user);
    await this.emailService.sendWelcome(user);
    await this.logger.logUserCreation(user.id);
    
    return user;
  }
}
```

---

## Anti-patterns to Avoid

1. **God classes** that handle everything from UI to database
2. **Business logic mixed with infrastructure** (e.g., validation in database layer)
3. **Multiple reasons to change** in a single module

---

## Key Takeaways

- Give each class/function **one clear purpose**
- Separate **validation, persistence, messaging, and logging**
- Test each responsibility **independently** with focused unit tests
- Changes become **localized** - modify email logic without touching user creation
- Code becomes **reusable** - use EmailService for other notifications
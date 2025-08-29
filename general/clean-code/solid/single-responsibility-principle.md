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
interface User {
  id: string;
  email: string;
  name: string;
}
```

---

## BAD — Mixed responsibilities

```typescript
class UserService {
  async createUser(email: string, name: string): Promise<User> {
    // Validation mixed in
    if (!email.includes('@')) throw new Error('Invalid email');
    if (name.length < 2) throw new Error('Name too short');
    
    // Database operations mixed in
    const user = { id: Date.now().toString(), email, name };
    await db.query('INSERT INTO users VALUES (?, ?, ?)', [user.id, email, name]);
    
    // Email sending mixed in
    await emailClient.send(email, 'Welcome!', `Hi ${name}!`);
    
    // Logging mixed in
    console.log(`User created: ${user.id}`);
    
    return user;
  }
}
```

---

## GOOD — Separated concerns

```typescript
class UserValidator {
  validate(email: string, name: string): void {
    if (!email.includes('@')) throw new Error('Invalid email');
    if (name.length < 2) throw new Error('Name too short');
  }
}

class UserRepository {
  async save(user: User): Promise<void> {
    await db.query('INSERT INTO users VALUES (?, ?, ?)', 
      [user.id, user.email, user.name]);
  }
}

class NotificationService {
  async sendWelcome(user: User): Promise<void> {
    await emailClient.send(user.email, 'Welcome!', `Hi ${user.name}!`);
  }
}

class UserService {
  constructor(
    private validator: UserValidator,
    private repository: UserRepository,
    private notifications: NotificationService
  ) {}

  async createUser(email: string, name: string): Promise<User> {
    this.validator.validate(email, name);
    
    const user = { id: Date.now().toString(), email, name };
    await this.repository.save(user);
    await this.notifications.sendWelcome(user);
    
    return user;
  }
}
```

---

## Anti-patterns to Avoid

1. **God classes** that handle everything from UI to database
2. **Business logic mixed with infrastructure**
(e.g., validation in database layer)
3. **Multiple reasons to change** in a single module

---

## Key Takeaways

- Give each class/function **one clear purpose**
- Separate **validation, persistence, messaging, and logging**
- Test each responsibility **independently** with focused unit tests
- Changes become **localized** - modify email logic
without touching user creation
- Code becomes **reusable** - use NotificationService for other features

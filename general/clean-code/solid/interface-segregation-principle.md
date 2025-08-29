# Interface Segregation Principle (ISP)

## Overview

Clients should not be forced to depend on interfaces they don't use.
Many small, focused interfaces are better than one large interface.

## Core Concept

- Keep interfaces small and client-specific
- Don't force implementations of unused methods
- Prefer role-based interfaces over header interfaces
- Compose multiple interfaces when needed

---

## Scaffolding

```typescript
type DocumentContent = string;
```

---

## BAD — Fat interface forcing unwanted implementations

```typescript
interface DocumentProcessor {
  read(): DocumentContent;
  write(content: DocumentContent): void;
  print(): void;
  scan(): DocumentContent;
  fax(number: string): void;
  email(address: string): void;
  encrypt(key: string): void;
  compress(): void;
}

class SimpleTextFile implements DocumentProcessor {
  private content = '';
  
  read(): DocumentContent {
    return this.content;
  }
  
  write(content: DocumentContent): void {
    this.content = content;
  }
  
  // Forced to implement irrelevant methods!
  print(): void { throw new Error('Cannot print'); }
  scan(): DocumentContent { throw new Error('Cannot scan'); }
  fax(number: string): void { throw new Error('Cannot fax'); }
  email(address: string): void { throw new Error('Cannot email'); }
  encrypt(key: string): void { throw new Error('No encryption'); }
  compress(): void { throw new Error('No compression'); }
}
```

---

## GOOD — Segregated, focused interfaces

```typescript
interface Readable {
  read(): DocumentContent;
}

interface Writable {
  write(content: DocumentContent): void;
}

interface Printable {
  print(): void;
}

interface Encryptable {
  encrypt(key: string): void;
}

// Implementations use only what they need
class SimpleTextFile implements Readable, Writable {
  private content = '';
  
  read(): DocumentContent {
    return this.content;
  }
  
  write(content: DocumentContent): void {
    this.content = content;
  }
}

class SecureDocument implements Readable, Writable, Encryptable {
  private content = '';
  private encrypted = false;
  
  read(): DocumentContent {
    if (this.encrypted) throw new Error('Document encrypted');
    return this.content;
  }
  
  write(content: DocumentContent): void {
    this.content = content;
  }
  
  encrypt(key: string): void {
    this.encrypted = true;
  }
}

class Printer implements Printable {
  print(): void {
    console.log('Printing...');
  }
}

// Clients depend only on what they need
function saveDocument(doc: Writable, content: string): void {
  doc.write(content);
}

function backupDocument(source: Readable, target: Writable): void {
  target.write(source.read());
}
```

---

## Anti-patterns to Avoid

1. **God interfaces** with dozens of methods
2. **Throwing NotImplementedException** in implementations
3. **Empty method bodies** to satisfy interface requirements

---

## Key Takeaways

- **Design interfaces from the client's perspective**
- **Split large interfaces** into smaller, cohesive ones
- **Compose interfaces** when implementations need multiple capabilities
- **Avoid forcing** classes to implement unused methods
- **Reduces coupling** and improves maintainability

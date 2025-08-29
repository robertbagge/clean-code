# Interface Segregation Principle (ISP)

## Overview

Clients should not be forced to depend on interfaces they don't use. Many small, focused interfaces are better than one large interface.

## Core Concept

- Keep interfaces small and client-specific
- Don't force implementations of unused methods
- Prefer role-based interfaces over header interfaces
- Compose multiple interfaces when needed

---

## Scaffolding

```typescript
// Types for document processing examples
type DocumentContent = string;
type DocumentMetadata = {
  created: Date;
  modified: Date;
  size: number;
};
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
  ocr(): string;
}

class SimpleTextFile implements DocumentProcessor {
  read(): DocumentContent {
    return this.content;
  }
  
  write(content: DocumentContent): void {
    this.content = content;
  }
  
  // Forced to implement irrelevant methods!
  print(): void {
    throw new Error('Text files cannot print themselves');
  }
  
  scan(): DocumentContent {
    throw new Error('Text files cannot scan');
  }
  
  fax(number: string): void {
    throw new Error('Text files cannot fax');
  }
  
  email(address: string): void {
    throw new Error('Text files cannot email themselves');
  }
  
  encrypt(key: string): void {
    throw new Error('No encryption support');
  }
  
  compress(): void {
    throw new Error('No compression support');
  }
  
  ocr(): string {
    throw new Error('OCR not applicable to text files');
  }
}
```

---

## GOOD — Segregated, focused interfaces

```typescript
// Small, focused interfaces
interface Readable {
  read(): DocumentContent;
}

interface Writable {
  write(content: DocumentContent): void;
}

interface Printable {
  print(): void;
}

interface Scannable {
  scan(): DocumentContent;
}

interface Encryptable {
  encrypt(key: string): void;
  decrypt(key: string): void;
}

interface Compressible {
  compress(): void;
  decompress(): void;
}

// Implementations use only what they need
class SimpleTextFile implements Readable, Writable {
  private content: DocumentContent = '';
  
  read(): DocumentContent {
    return this.content;
  }
  
  write(content: DocumentContent): void {
    this.content = content;
  }
}

class SecureDocument implements Readable, Writable, Encryptable {
  private content: DocumentContent = '';
  private encrypted = false;
  
  read(): DocumentContent {
    if (this.encrypted) throw new Error('Document is encrypted');
    return this.content;
  }
  
  write(content: DocumentContent): void {
    if (this.encrypted) throw new Error('Document is encrypted');
    this.content = content;
  }
  
  encrypt(key: string): void {
    this.content = this.performEncryption(this.content, key);
    this.encrypted = true;
  }
  
  decrypt(key: string): void {
    this.content = this.performDecryption(this.content, key);
    this.encrypted = false;
  }
  
  private performEncryption(data: string, key: string): string {
    // Encryption logic
    return data;
  }
  
  private performDecryption(data: string, key: string): string {
    // Decryption logic
    return data;
  }
}

class Scanner implements Scannable, Printable {
  scan(): DocumentContent {
    // Scan document
    return 'scanned content';
  }
  
  print(): void {
    // Print document
  }
}

// Clients depend only on what they need
function saveDocument(doc: Writable, content: string): void {
  doc.write(content);
}

function backupDocument(source: Readable, target: Writable): void {
  const content = source.read();
  target.write(content);
}

function secureBackup(
  source: Readable,
  target: Writable & Encryptable,
  key: string
): void {
  const content = source.read();
  target.write(content);
  target.encrypt(key);
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
# Clean Code Docs — Start Here

These docs help you make pragmatic Go design choices. If you only read one
thing, read **Dependency Inversion (DIP)**.

## Why lead with DIP?

- **Testability:** Inject fakes/in-memory deps.
- **Portability:** Swap drivers (SQL/Mongo/HTTP) without touching business
  logic.
- **Boundaries:** Consumers define interfaces; providers implement them.

👉 **Start here:** [Dependency Inversion (DIP)](clean-code/dependency-inversion.md)

---

## Jump to the right guide

- **New service or refactor?**  
  Read: [DIP](clean-code/dependency-inversion.md) →
  [SRP](clean-code/single-responsibility.md)  
  Also see: [best-practices.md › Package Structure](./best-practices.md#package-structure)
  and [› Dependency Injection](./best-practices.md#dependency-injection)

- **Business logic imports `database/sql` or an SDK?**  
  Read: [DIP](clean-code/dependency-inversion.md)

- **Huge interface / hard to test?**  
  Read: [ISP](clean-code/interface-segregation.md)

- **Adding a backend means editing a switch?**  
  Read: [OCP](clean-code/open-closed.md)

- **Two implementations behave differently?**  
  Read: [LSP](clean-code/liskov-substitution.md)

- **Rules duplicated across handlers/services?**  
  Read: [DRY](clean-code/dry.md)

- **Feels over-engineered?**  
  Read: [KISS](clean-code/kiss.md)

---

## Fast recipes (anchors into docs)

- **Where to define interfaces** → [best-practices.md › Interfaces](./best-practices.md#interfaces)
- **Domain errors & driver→domain mapping** → [best-practices.md › Error Handling](./best-practices.md#error-handling)
- **Constructor injection / functional options** →
  [best-practices.md › Dependency Injection](./best-practices.md#dependency-injection)
- **Testing (fakes, table, golden, contract)** →
  [best-practices.md › Testing Practices](./best-practices.md#testing-practices)
- **Package layout** → [best-practices.md › Package Structure](./best-practices.md#package-structure)

---

## TL;DRs

- **DIP:** Depend on interfaces defined by consumers; inject implementations.  
- **SRP:** One reason to change per function/type/package.  
- **ISP:** Small, capability-focused interfaces.  
- **OCP:** Add new behavior via new implementations, not edits.  
- **LSP:** Implementations honor the same behavior and error semantics.  
- **DRY:** Single source of truth for rules/config.  
- **KISS:** Simplest thing that works; avoid premature abstraction.

## Check yourself before you wreck yourself

Use the quick [design-checklist.md](./design-checklist.md) before shipping.

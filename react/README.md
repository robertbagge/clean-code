# Clean Code Docs — Start Here

These docs help you make pragmatic React design choices. If you only read one
thing, read **Dependency Inversion (DIP)**.

## Why lead with DIP?

- **Testability:** Inject dependencies into hooks/components for easy testing.
- **Portability:** Swap implementations (REST/GraphQL/WebSocket) without touching component logic.
- **Boundaries:** Components define interfaces; services implement them.

👉 **Start here:** [Dependency Inversion (DIP)](clean-code/dependency-inversion-principle.md)

---

## Jump to the right guide

- **New component or refactor?**  
  Read: [DIP](clean-code/dependency-inversion-principle.md) →
  [SRP](clean-code/single-responsibility-principle.md)  
  Also see: [best-practices.md › Component Structure](./best-practices.md#component-structure)
  and [› Dependency Injection](./best-practices.md#dependency-injection)

- **Component directly fetches data or calls APIs?**  
  Read: [DIP](clean-code/dependency-inversion-principle.md)

- **Component has too many props / hard to test?**  
  Read: [ISP](clean-code/interface-segregation-principle.md)

- **Adding a feature means editing existing components?**  
  Read: [OCP](clean-code/open-closed-principle.md)

- **Child components break when swapped?**  
  Read: [LSP](clean-code/liskov-substitution-principle.md)

- **Business logic duplicated across components?**  
  Read: [DRY](clean-code/dry-principle.md)

- **Feels over-engineered?**  
  Read: [KISS](clean-code/kiss-principle.md)

---

## Fast recipes (anchors into docs)

- **Where to define interfaces** → [best-practices.md › TypeScript Interfaces](./best-practices.md#typescript-interfaces)
- **Error handling & boundaries** → [best-practices.md › Error Handling](./best-practices.md#error-handling)
- **Dependency injection patterns** →
  [best-practices.md › Dependency Injection](./best-practices.md#dependency-injection)
- **Testing (stubs, mocks, render testing)** →
  [best-practices.md › Testing Practices](./best-practices.md#testing-practices)
- **Component organization** → [best-practices.md › Component Structure](./best-practices.md#component-structure)

---

## TL;DRs

- **DIP:** Components depend on interfaces; inject implementations via props/context.  
- **SRP:** One responsibility per component/hook.  
- **ISP:** Lean prop interfaces; components only receive what they use.  
- **OCP:** Extend via composition and children, not modification.  
- **LSP:** Child components honor parent contracts without breaking.  
- **DRY:** Single source of truth for business rules/validation.  
- **KISS:** Simplest component that works; avoid premature abstraction.

## Check yourself before you ship

Use the quick [design-checklist.md](./design-checklist.md) before shipping.

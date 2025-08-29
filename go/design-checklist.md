# Design Checklist (Pre-PR)

## Structure

- [ ] Package has a single, cohesive purpose (SRP)
- [ ] High-level code depends on interfaces, not concrete drivers (DIP)
- [ ] Interfaces are small and defined in the consumer (ISP)
- [ ] New backends don’t require editing consumers (OCP)
- [ ] For critical schemas and DTOs strong-typing is maintained and preferred
      over clean-code interfaces

## Behavior

- [ ] Implementations share the same contract and error semantics (LSP)
- [ ] Domain errors are centralized; driver errors are mapped (best-practices.md)
- [ ] Validation/business rules live in one place (DRY)
- [ ] Simplest approach chosen; no premature abstraction (KISS)

## Testing

- [ ] Only public methods are tested
- [ ] Table-driven tests cover happy + edge paths
- [ ] Fakes/in-memory repos used instead of heavy mocks
- [ ] Golden files used for generator/renderer output (when applicable)
- [ ] Contract/LSP tests only where there are ≥2 impls and behavior matters

## Wiring

- [ ] Dependencies injected via constructors; optional ones via functional options
- [ ] No construction of infra deps inside business logic
- [ ] Config is a plain struct with explicit required fields

## Go idioms

- [ ] Errors wrapped with context (`fmt.Errorf("...: %w", err)`)
- [ ] Avoid reflection unless strictly needed
- [ ] Prefer functions over interfaces for simple operations

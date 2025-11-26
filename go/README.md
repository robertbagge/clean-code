# Clean Code – Go

## What this is

A practical, opinionated guide to clean Go: SOLID-style principles adapted for
Go and a set of best practices you can use across
services, CLIs, and libraries.

## Who it's for

Engineers and AI agents building or refactoring Go codebases.

## Scope & portability

Framework-agnostic: applies to standard library code, microservices, CLIs, and libraries.

### Opinions

We have established patterns for:

* **Logging**: `log/slog` with JSON handlers — see [logging.md](./best-practices/logging.md)
* **Database**: sqlc for type-safe queries with pgx — see [database.md](./best-practices/database.md)
* **Repository**: Domain-scoped repositories with DB-to-domain mapping — see [repository.md](./best-practices/repository.md)
* **Configuration**: Environment-based config — see [configuration.md](./best-practices/configuration.md)

No strong opinions yet on:

* frameworks/routers
* DI tooling
* channels

## How to use this guide

### For AI agent research

Unless otherwise instructed, read all documents to get a full picture of best
practices & clean code before distilling for your task.

### For implementation guidance

Lead with the [Best Practices](./best-practices/),
then apply these principles first:  
[DIP](./clean-code/dependency-inversion.md),
[SRP](./clean-code/single-responsibility.md),
[ISP](./clean-code/interface-segregation.md),
[DRY](./clean-code/dry.md),
[KISS](./clean-code/kiss.md).  

Refer to [OCP](./clean-code/open-closed.md) and
[LSP](./clean-code/liskov-substitution.md) as needed.

## Clean code principles

* [Single Responsibility (SRP)](./clean-code/single-responsibility.md)
  – One reason to change per function/type/package.
* [Interface Segregation (ISP)](./clean-code/interface-segregation.md)
  – Small, capability-focused interfaces; avoid fat “god” interfaces.
* [Dependency Inversion (DIP)](./clean-code/dependency-inversion.md)
  – Consumers define interfaces; inject implementations for testability/portability.
* [Open–Closed (OCP)](./clean-code/open-closed.md)
  – Add behavior via new implementations, not by editing callers.
* [Liskov Substitution (LSP)](./clean-code/liskov-substitution.md)
  – Implementations honor the same contracts, errors, and semantics.
* [Don’t Repeat Yourself (DRY)](./clean-code/dry.md)
  – Centralize rules, conversions, and cross-cutting utilities.
* [Keep It Simple (KISS)](./clean-code/kiss.md)
  – Prefer the simplest working design; avoid premature abstraction.

## Check yourself before you ship

Use the quick [design-checklist.md](./design-checklist.md) before shipping.

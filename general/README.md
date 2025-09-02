# Clean Code – General

## What this is

A practical, opinionated guide to clean code: SOLID-style principles adapted
for broad use across services, CLIs, UIs, and libraries.

## Who it's for

Engineers and AI agents building or refactoring codebases in any language.

## Scope & portability

Language-/framework-agnostic: applies across backends, CLIs, and UIs.

No strong opinions on:

- folder structure
- build systems or tooling
- testing frameworks
- logging/tracing stacks

## How to use this guide

### For AI agent research

Unless otherwise instructed, read all documents to get a full picture of best
practices & clean code before distilling for your task.

### For implementation guidance

Lead with these principles first: [DIP](./clean-code/dependency-inversion.md),
[SRP](./clean-code/single-responsibility.md),
[ISP](./clean-code/interface-segregation.md),
[DRY](./clean-code/dry.md),
[KISS](./clean-code/kiss.md).  

Refer to [OCP](./clean-code/open-closed.md) and
[LSP](./clean-code/liskov-substitution.md) as needed.

## Clean code principles

- [Single Responsibility (SRP)](./clean-code/single-responsibility.md)
  – One reason to change per module/type/function.
- [Interface Segregation (ISP)](./clean-code/interface-segregation.md)
  – Small, capability-focused interfaces; avoid “god” interfaces.
- [Dependency Inversion (DIP)](./clean-code/dependency-inversion.md)
  – Consumers define contracts; inject implementations for testability/portability.
- [Open–Closed (OCP)](./clean-code/open-closed.md)
  – Add behavior via extension, not edits to stable modules.
- [Liskov Substitution (LSP)](./clean-code/liskov-substitution.md)
  – Implementations honor the same contracts and error semantics.
- [Don’t Repeat Yourself (DRY)](./clean-code/dry.md)
  – Centralize shared rules, conversions, and patterns.
- [Keep It Simple (KISS)](./clean-code/kiss.md)
  – Prefer the simplest working design; refactor when pressure appears.

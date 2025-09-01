# Clean Code – React

## What this is

A practical, opinionated guide to clean React: SOLID-style principles adapted
for React + a set of best practices you can drop into any React-based codebase
(React, Next.js, React Native).

## Scope & portability

Framework-agnostic: applies to React DOM/Native/Next.js. No assumptions on
data-fetch libraries, routing, forms, or styling system.

## How to use this guide

### For AI agent research

Unless otherwise instructred, read all documents to get a full picture of best
practices & clean code before distilling for your task.

### For implementation guidance

Lead with the [Best Practices index](./best-practices/index.md),
then dip into the clean-code principles when necessary. Common starting points:
[SRP](./clean-code/single-responsibility-principle.md) and
[DIP](./clean-code/dependency-inversion-principle.md).

## Start here

* Read the [Best Practices index](./best-practices/index.md) for
day-to-day conventions, then consult a principle when shaping an API or refactoring.

## Clean code principles

* [Single Responsibility (SRP)](./clean-code/single-responsibility-principle.md)
– One responsibility per component/hook. Split fetching/formatting/presentation.
* [Interface Segregation (ISP)](./clean-code/interface-segregation-principle.md)
– Lean props; composition over fat prop bags.
* [Dependency Inversion (DIP)](./clean-code/dependency-inversion-principle.md)
– Depend on interfaces; inject implementations via props/context.
* [Open–Closed (OCP)](./clean-code/open-closed-principle.md)
– Extend via composition/children; avoid modifying core components.
* [Liskov Substitution (LSP)](./clean-code/liskov-substitution-principle.md)
– Variants honor the same contract; consistent callbacks/semantics.
* [Don’t Repeat Yourself (DRY)](./clean-code/dry-principle.md)
– Centralize repeating patterns (cards, date formatting, rules).
* [Keep It Simple (KISS)](./clean-code/kiss-principle.md)
– Start simple; refactor when pressure appears.

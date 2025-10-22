# Clean Code – React

## What this is

A practical, opinionated guide to clean React: SOLID-style principles adapted
for React + a set of best practices you can drop into any React-based codebase
(React, Next.js, React Native).

## Who it's for

Engineers and AI agents building or refactoring React/Next.js/React Native apps.

## Scope & portability

Framework-agnostic: applies to React DOM/Native/Next.js.

No strong opinions on:

* folder structure
* data-fetching libraries
* routing
* forms
* styling system.

## How to use this guide

### For AI agent research

Unless otherwise instructed, read all documents to get a full picture of best
practices & clean code before distilling for your task.

### For implementation guidance

Lead with the [Best Practices index](./best-practices/index.md),
then dip into the clean-code principles as needed.
Common starting points:
[SRP](./clean-code/single-responsibility.md) and
[DIP](./clean-code/dependency-inversion.md).

### For React Native / Expo projects

Start with [React Native Best Practices](./best-practices/react-native.md)
for platform-specific patterns, then refer to general principles.

## Clean code principles

* [Single Responsibility (SRP)](./clean-code/single-responsibility.md)
– One responsibility per component/hook. Split fetching/formatting/presentation.
* [Interface Segregation (ISP)](./clean-code/interface-segregation.md)
– Lean props; composition over fat prop bags.
* [Dependency Inversion (DIP)](./clean-code/dependency-inversion.md)
– Depend on interfaces; inject implementations via props/context.
* [Open–Closed (OCP)](./clean-code/open-closed.md)
– Extend via composition/children; avoid modifying core components.
* [Liskov Substitution (LSP)](./clean-code/liskov-substitution.md)
– Variants honor the same contract; consistent callbacks/semantics.
* [Don’t Repeat Yourself (DRY)](./clean-code/dry.md)
– Centralize repeating patterns (cards, date formatting, rules).
* [Keep It Simple (KISS)](./clean-code/kiss.md)
– Start simple; refactor when pressure appears.

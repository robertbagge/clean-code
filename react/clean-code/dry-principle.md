# DRY Principle (Don't Repeat Yourself) in React

## Overview

The DRY principle states that every piece of knowledge should have a single,
unambiguous representation in the system. In React, this means centralizing
business logic, validation rules, and UI patterns to avoid duplication and
ensure consistency across your application.

## Core Concept

In React, DRY means:

* Extract common UI patterns into reusable components
* Use custom hooks for shared logic

The above together with the other clean code and best practices will get
you to a clean codebase. Using libraries for data loading, forms etc
will get you a long way.

## Implementation Examples

### BAD — Duplicated definition for core UI component (DRY violation)

> Duplicated card definition. Easily results in inconsistencies
> Note: class-based styling only used to show concept

```typescript
function UserCard({ name, bio }: { name: string; bio: string }) {
  return (
    <div className="rounded-xl border p-4 shadow-sm bg-background">
      <h3 className="text-lg font-semibold">{name}</h3>
      <p className="mt-2 text-sm text-muted-foreground">{bio}</p>
      <div className="mt-4 text-xs text-muted-foreground">View profile →</div>
    </div>
  )
}

function OrderCard({ id, total }: { id: string; total: string }) {
  return (
    <div className="rounded-xl border p-4 shadow-sm bg-background">
      <h3 className="text-lg font-semibold">Order #{id}</h3>
      <p className="mt-2 text-sm text-muted-foreground">Total: {total}</p>
      <div className="mt-4 text-xs text-muted-foreground">Manage →</div>
    </div>
  )
}


```

### GOOD — Shared component definition

> Cards will look the same regardless of where they are used
> Note: class-based styling is only used to show the concept

```typescript
type CardProps = {
  header?: React.ReactNode
  footer?: React.ReactNode
  children?: React.ReactNode
  className?: string
}

function Card({ header, children, footer, className }: CardProps) {
  return (
    <section
      className={`rounded-xl border p-4 shadow-sm bg-background ${className ?? ''}`}
    >
      {header && <div className="text-lg font-semibold">{header}</div>}
      {children && <div className="mt-2 text-sm text-muted-foreground">{children}</div>}
      {footer && <div className="mt-4 text-xs text-muted-foreground">{footer}</div>}
    </section>
  )
}

const UserCard = ({ name, bio }: { name: string; bio: string }) =>
  <Card header={name} footer="View profile →">{bio}</Card>

const OrderCard = ({ id, total }: { id: string; total: string }) =>
  <Card header={`Order #${id}`} footer="Manage →">Total: {total}</Card>

```

## When to Apply DRY in React

### Apply DRY for

* Defining core components that build up the UI
* Common hooks that are used in multiple places
* Api clients and data loading
* Validation rules
* and more

### Balance with Clarity

* Some duplication for readability is acceptable
* Don't over-abstract simple patterns
* Consider maintenance cost vs. duplication cost

## Anti-patterns to Avoid

1. **Scattered validation**: Same rules in multiple components
2. **Duplicated formatting**: Date/number formatting everywhere
3. **Repeated conditionals**: Same business logic checks
4. **Copy-paste components**: Nearly identical components

## React-Specific DRY Techniques

1. **Custom hooks** for shared logic
2. **Utility classes** for business rules
3. **Component composition** for UI patterns
4. **Context providers** for cross-cutting concerns
5. **Higher-order components** for common behaviors

## Key Takeaways

* Create **single sources of truth** for business logic
* **Centralize validation** and formatting rules
* Use **custom hooks** for reusable stateful logic
* Extract **common patterns** into utilities
* Balance DRY with **code clarity**

## Related Best Practices

For reusable patterns and utilities, see
👉 [best-practices.md](../best-practices/best-practices.md)

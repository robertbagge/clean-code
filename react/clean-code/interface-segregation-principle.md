# Interface Segregation Principle (ISP) in React

## Overview

The Interface Segregation Principle states that components should not be forced
to depend on props they don't use. In React, this means creating focused prop
interfaces where components only receive the data and callbacks they actually
need, improving reusability and testability, and making it easier to find
performance bottlenecks.

## Core Concept

In React, ISP means:

* Components receive only the props they use
* Split large prop interfaces into focused ones
* Use composition to combine capabilities
* Avoid "prop drilling" unnecessary data
* Create lean, specific interfaces for each use case

## Implementation Examples

### Scaffolding for examples

```typescript
interface User {
  name: string;
  email: string;
  role: 'user' | 'guest'
  profile?: { 
    avatar?: string 
  }
}
```

### BAD — Fat Props passed around

> Components receive entire objects and props they don't use.

```typescript
interface Props {
  user: User
  onLogout: () => void
}

// Uses only name, avatar and onLogout, but receives a fat prop bag
function ProfileHeader({
  user,
  onLogout,
}: Props) {
  return (
    <header>
      <img src={user.profile?.avatar ?? '/avatar.png'} alt={user.name} />
      <span>{user.name}</span>
      <button onClick={onLogout}>Logout</button>
    </header>
  )
}

```

### GOOD — Segregated Interfaces (Only what's needed)

> Components receive only the props they actually use.

```typescript
type AvatarProps = { src?: string; alt: string }
function Avatar({ src, alt }: AvatarProps) {
  return (<img src={src ?? '/avatar.png'} alt={alt} />)
}

type HeaderProps = {
  userName: User['name']
  userAvatar?: NonNullable<User['profile']>['avatar']
  onLogout: () => void
}

function ProfileHeader({ userName, userAvatar, onLogout }: HeaderProps) {
  return (
    <header>
      <Avatar src={userAvatar} alt={userName} />
      <span>{userName}</span>
      <button onClick={onLogout}>Logout</button>
    </header>
  )
}

```

## When to Apply ISP in React

### Apply ISP for

* Components receiving large objects but using few fields
* Reusable components in different contexts
* Components with many optional props
* Deep component trees with prop drilling
* Component libraries and design systems

### Balance with Practicality

* Small, simple components can receive full objects
* Tightly coupled components can share interfaces
* Performance-critical paths may bundle props

## Anti-patterns to Avoid

1. **God objects as props**: Passing entire state objects everywhere
2. **Prop drilling**: Passing props through components that don't use them
3. **Kitchen sink interfaces**: Components with 10+ props
4. **Unclear dependencies**: Can't tell what data component actually needs

## React-Specific ISP Techniques

1. **Destructure at call site** to show what's used
2. **Custom hooks** to provide focused data subsets
3. **Component composition** over prop passing
4. **Context splitting** for different concerns
5. **Prop spreading** with pick/omit utilities to get end-to-end type safety

## Key Takeaways

* Components should **only receive props they use**
* **Split fat interfaces** into focused ones
* Use **composition** to combine capabilities
* Create **custom hooks** for data selection
* Keep prop interfaces **lean and specific**
* Lean, stable props pair well with React.memo for fewer re-renders

## Related Best Practices

For component composition and prop patterns, see
👉 [best-practices.md](../best-practices/best-practices.md)

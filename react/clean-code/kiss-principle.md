# KISS Principle (Keep It Simple, Stupid) in React

## Overview

The KISS principle advocates for simplicity in design and implementation. In
React, this means choosing the simplest solution that works, avoiding premature
optimization, and not over-engineering components. Simple code is easier to
understand, maintain, and debug.

## Core Concept

In React, KISS means:

* Start with the simplest implementation
* Avoid premature abstraction
* Use built-in React features before external libraries
* Keep component hierarchies flat when possible
* Choose clarity over cleverness

## Implementation Example

### BAD — Over-engineered simple counter

```typescript
type CounterAction = { type: 'inc' | 'dec' | 'reset' }
function counterReducer(state: number, action: CounterAction) {
  switch (action.type) {
    case 'inc': return state + 1
    case 'dec': return state - 1
    case 'reset': return 0
  }
}
function Counter() {
  const [count, dispatch] = useReducer(counterReducer, 0)
  return (
    <div>
      <button onClick={() => dispatch({ type: 'dec' })}>-</button>
      <span>{count}</span>
      <button onClick={() => dispatch({ type: 'inc' })}>+</button>
    </div>
  )
}

```

### GOOD — Keep it simple with useState

```typescript
function Counter() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <button onClick={() => setCount((c) => c - 1)}>-</button>
      <span>{count}</span>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
    </div>
  )
}

```

> Note: `useReducer` is great when transitions are complex
> (e.g., undo/redo, multiple event sources).
> For a simple counter, `useState` is simpler.

### BAD — factory/classes for simple theme toggling

```typescript
type Theme = { primary: string; background: string }
abstract class ThemeFactory { abstract create(): Theme }
class LightFactory extends ThemeFactory {
  create() { return { primary: '#00f', background: '#fff' } }
}
class DarkFactory extends ThemeFactory {
  create() { return { primary: '#0af', background: '#111' } }
}
function useTheme(factory: ThemeFactory) {
  const [theme] = useState(() => factory.create())
  return theme
}
```

### GOOD — plain state + lookup map

```typescript
const themes = {
  light: { primary: '#00f', background: '#fff' },
  dark: { primary: '#0af', background: '#111' },
} as const

function useTheme() {
  const [mode, setMode] = useState<'light' | 'dark'>('light')
  return {
    colors: themes[mode], mode,
    toggle: () => setMode((m) => (m === 'light' ? 'dark' : 'light'))
  }
}
```

## When to Apply KISS in React

### Keep Simple for

* Prototypes and MVPs
* Internal tools
* Components with single responsibility
* Straightforward CRUD operations
* Simple state management

### Consider Complexity When

* Building reusable libraries
* Performance is critical
* Complex business requirements
* Multiple team collaboration
* Long-term maintenance is crucial

## Anti-patterns to Avoid

1. **Premature optimization**: Complex solutions for simple problems
2. **Over-abstraction**: Too many layers of indirection
3. **Pattern overuse**: Applying patterns where not needed
4. **Library addiction**: Using libraries for trivial tasks

## React-Specific KISS Techniques

1. Start with **useState** before reaching for reducers
2. Use **built-in browser APIs** before external libraries
3. Keep **component trees shallow**
4. Write **inline handlers** before extracting them
5. Prefer inline handlers early; if passing down to memoized children or
big lists, stabilize with `useCallback`.

## Key Takeaways

* **Start simple**, refactor when needed
* Avoid **premature abstraction**
* Use **built-in features** first
* Choose **clarity over cleverness**
* **Simple code** is maintainable code

## Related Best Practices

For pragmatic approaches and patterns, see
👉 [best-practices.md](../best-practices/index.md)

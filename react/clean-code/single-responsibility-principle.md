# Single Responsibility Principle (SRP) in React

## Overview

The Single Responsibility Principle states that a component or hook should have
one reason to change. In React, this means each component should handle one
specific aspect of the UI, and each hook should encapsulate one specific piece
of business logic. This makes components more reusable, testable, and maintainable.

## Core Concept

In React, SRP means:

* Components handle either presentation OR logic, not both
* Hooks encapsulate a single piece of business logic
* Separate data fetching from data display
* Extract complex state management into custom hooks
* Keep validation, formatting, and API calls in dedicated modules

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface Activity {
  id: string
  user: string
  message: string
  createdAt: Date
}

interface ActivityApi {
  listRecent(): Promise<Activity[]>
}

```

### BAD — Multiple Responsibilities (SRP violation)

> Component handles data fetching, state management, validation, and presentation.

```typescript
import dayjs from 'dayjs'
import relativeTime from 'dayjs/plugin/relativeTime'
dayjs.extend(relativeTime)

function ActivityPanel({ activityApi }: { activityApi: ActivityApi }) {
  const [items, setItems] = React.useState<Activity[]>([])
  const [loading, setLoading] = React.useState(true)
  const [error, setError] = React.useState<string | null>(null)
  const [filter, setFilter] = React.useState<'all' | 'mine'>('all')
  const me = 'amy' // pretend from auth

  React.useEffect(() => {
    let id: number | undefined
    const load = async () => {
      try {
        setLoading(true)
        setError(null)
        const data = await activityApi.listRecent()
        // inline sorting
        setItems(data.slice().sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime()))
      } catch (e) {
        setError(e instanceof Error ? e.message : String(e))
      } finally {
        setLoading(false)
      }
    }
    load()
    id = window.setInterval(load, 15000) // polling mixed in
    return () => clearInterval(id)
  }, [activityApi])

  const visible = items.filter((a) => (filter === 'mine' ? a.user === me : true))

  if (loading) return <div>Loading…</div>
  if (error) return <div role="alert">Failed to load: {error}</div>

  return (
    <div className="card">
      <div className="card-header">
        <strong>Activity</strong>
        <div>
          <button onClick={() => setFilter('all')}>All</button>
          <button onClick={() => setFilter('mine')}>Mine</button>
        </div>
      </div>
      <ul className="card-body">
        {visible.map((a) => (
          <li key={a.id}>
            <span>{a.user}</span> — <span>{a.message}</span> <em>· {dayjs(a.createdAt).fromNow()}</em>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### GOOD — Single Responsibility (SRP compliant)

> Each component/hook has one clear responsibility.

```typescript
import dayjs from 'dayjs'
import relativeTime from 'dayjs/plugin/relativeTime'
dayjs.extend(relativeTime)

// 1) Formatting (single responsibility)
function RelativeTime({ date }: { date: Date }) {
  return <em>· {dayjs(date).fromNow()}</em>
}

// 2) Data hook: fetching + polling only
function useActivityFeed(activityApi: ActivityApi, pollMs = 15000) {
  const [items, setItems] = React.useState<Activity[]>([])
  const [loading, setLoading] = React.useState(true)
  const [error, setError] = React.useState<Error | null>(null)

  const load = React.useCallback(async () => {
    try {
      setLoading(true)
      setError(null)
      const data = await activityApi.listRecent()
      setItems(data.slice().sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime()))
    } catch (e) {
      setError(e as Error)
    } finally {
      setLoading(false)
    }
  }, [activityApi])

  React.useEffect(() => {
    const id = window.setInterval(load, pollMs)
    load()
    return () => clearInterval(id)
  }, [load, pollMs])

  return { items, loading, error, refresh: load }
}

// 3) Pure display: presentation only
function ActivityList({ items }: { items: Activity[] }) {
  return (
    <ul className="card-body">
      {items.map((a) => (
        <li key={a.id}>
          <span>{a.user}</span> — <span>{a.message}</span> <RelativeTime date={a.createdAt} />
        </li>
      ))}
    </ul>
  )
}

// 4) Small container: orchestrates local UI state and renders loading/error
function ActivityPanel({ activityApi, me }: { activityApi: ActivityApi; me: string }) {
  const { items, loading, error, refresh } = useActivityFeed(activityApi)
  const [filter, setFilter] = React.useState<'all' | 'mine'>('all')

  const visible = React.useMemo(
    () => items.filter((a) => (filter === 'mine' ? a.user === me : true)),
    [items, filter, me]
  )

  if (loading) return <div>Loading…</div>
  if (error) return <div role="alert">Failed to load: {error.message}</div>

  return (
    <div className="card">
      <div className="card-header">
        <strong>Activity</strong>
        <div>
          <button onClick={() => setFilter('all')}>All</button>
          <button onClick={() => setFilter('mine')}>Mine</button>
          <button onClick={refresh}>Refresh</button>
        </div>
      </div>
      <ActivityList items={visible} />
    </div>
  )
}
```

### When to Apply SRP in React

* Components mixing data fetching with presentation
* Hooks handling multiple unrelated state pieces
* Components that both orchestrate effects (polling/subscriptions) and render UI
* Business logic mixed with UI components
* Components that are hard to test in isolation

### Acceptable Coupling

* Simple controlled inputs can handle their own state
* Small components can combine closely related presentation concerns
* Light, inline formatting in display components is fine if it’s trivial and consistent
* Container components can orchestrate multiple single-purpose children
* Certain hooks that needs to intermingle business logic with ui logic
to achieve there goals. Always decouple where possible.

### Anti-patterns to Avoid

1. **God components**: Components doing everything (fetching, state, formatting, rendering)
2. **Business logic in components**: Calculations, validations, permissions in render/effects
3. **Mixed concerns in hooks**: One hook managing unrelated state/effects
4. **Inline complex logic**: Large transformations, filtering, or sorting in JSX
5. **Growing reducers/hooks**: “One hook to rule them all” accumulating flags and modes

### React-Specific SRP Techniques

1. **Custom hooks** for business logic and effects (test with injected deps)
2. **Presentation/Container** split to isolate UI from orchestration
3. **Utility functions/components** for formatting (e.g., `RelativeTime`)
4. **API client/service modules** for network and persistence concerns
5. **Tiny effect hooks** for timers/polling/subscriptions that can be composed
6. **Error Boundaries** to isolate failure handling near a feature/container,
keeping display components pure
7. **Suspense Boundaries** to isolate loading states at section/page edges;
avoid sprinkling loading UI inside leaf components

### Key Takeaways

* Each component or hook should have **one reason to change**
* Extract **business logic into hooks**; keep **presentation components pure**
* Use **composition** to combine small, single-purpose pieces
* **Inject dependencies** (APIs, loggers) to keep hooks/components testable
* **Test each piece** in isolation (utils → unit; hooks → with stubs; UI → behavior)

## Related Best Practices

For component structure, testing patterns, and state management, see
👉 [best-practices.md](../best-practices/best-practices.md)

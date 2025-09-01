# Error Handling

## Error Taxonomy

* Never surface network errors directly. Translate to UI-level errors that only disclose what they need to
* Surface domain errors to the UI; log infra errors with context (never to console in prod)

### UI errors

```typescript
class NotFound extends Error {
  constructor(public id: string, entityType: string) { super("Not Found") }
}

class Unauthorized extends Error {
  constructor(public id: string, entityType: string) { super("Unauthorized") }
}

class Forbidden extends Error {
  constructor(public id: string, entityType: string) { super("Forbidden") }
}

class BadRequest extends Error {
  constructor(public id: string, entityType: string) { super(`NotFound`) }
}

class Conflict extends Error {
  constructor(public id: string, entityType: string) { super(`Conflict`) }
}
// ...
```

## Boundaries

* Use Error Boundaries around page/section roots.
* Let non-recoverable render errors bubble to a boundary.
* For async flows, either:

  * return a discriminated union state, or
  * throw promises/errors for Suspense + boundaries.

```typescript
// Discriminated union example
type Loading = { status: 'loading' }
type Failure = { status: 'error'; error: Error }
type Success<T> = { status: 'success'; data: T }
type AsyncState<T> = Loading | Failure | Success<T>
```

## Policy Inversion

* Invert “what to do on error” via props or context.
* Keep UI generic; delegate toast/log/redirect.

```typescript
type Props = { onError?: (error: Error) => void }

function SaveButton({ onError }: Props) {
  const save = useSave()

  const handleClick = async () => {
    try { await save() }
    catch (e) { onError?.(e as Error) }
  }

  return <button onClick={handleClick}>Save</button>
}
```

## Logging and Telemetry

* Centralize logging behind an interface.
* Inject it where needed; avoid direct `console.*` in components.

```typescript
interface Logger { error: (msg: string, meta?: unknown) => void }
```

## Retriable vs Non-Retriable

* Provide retry affordances when the error is likely transient.
* Don’t auto-retry endlessly; backoff and cap attempts.

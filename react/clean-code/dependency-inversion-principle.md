# Dependency Inversion Principle (DIP) in React

## Overview

The Dependency Inversion Principle states that high-level components should not
depend on low-level implementations; both should depend on abstractions. In
React, this is achieved through dependency injection via props and context,
allowing components to depend on interfaces rather than concrete implementations.
This makes components more testable, portable, and maintainable.

## Core Concept

In React, DIP means:

* Components depend on interfaces, not implementations
* Inject services via props or context
* Hooks declare the contracts they need
* Keep API/database details out of components
* Use dependency injection for testability
* Allows separation of concerns which great for using business/persistence logic for different platforms - Web/React Native etc.

## Implementation Examples

### Scaffolding for Examples

```typescript
// types.ts
interface User {
  id: string
  name: string
  email: string
  createdAt: Date
}

interface AuthToken {
  token: string
  expiresAt: Date
}

class UserNotFoundError extends Error {
  constructor(public userId: string) {
    super(`User ${userId} not found`)
  }
}
```

### BAD - Component handles complete/error actions on its own

```typescript
function FeedbackForm() {
  const [title, setTitle] = useState('')
  const [description, setDescription] = useState('')
  const { sendFeedback } = useFeedback()

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    try {
      await sendFeedback(title, description)

      // VIOLATION: UI component decides next UI state
      window.location.href = '/dashboard'
    } catch (error) {
      // VIOLATION: UI component handles error
      console.error('Send feedback failed:', error)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
    </form>
  )
}
```

### GOOD - component accepts success/error hook

```typescript
type Props = {
  onFeedbackSent: (args: { estimatedResponseTime: string }) => void
  onFeedbackError: (error: Error) => void
}

function FeedbackForm({ onFeedbackSent, onFeedbackError }: Props) {
  const [title, setTitle] = useState('')
  const [description, setDescription] = useState('')
  const { sendFeedback } = useFeedback()

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    try {
      const { estimatedResponseTime } = await sendFeedback(title, description)
      onFeedbackSent({ estimatedResponseTime })
    } catch (error) {
      onFeedbackError(error instanceof Error ? error : new Error('Unknown error'))
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
    </form>
  )
}

function FeedbackSection() {
  const { toast } = useToast()

  const onFeedbackSent = ({ estimatedResponseTime }: { estimatedResponseTime: string }) => {
    toast.success(`Feedback sent - we will respond in ${estimatedResponseTime}`)
  }
  const onFeedbackError = (error: Error) => {
    toast.error('Could not send feedback')
    // Optionally bubble to an error boundary
    // throw error
  }

  return (
    <>
      <FeedbackForm onFeedbackSent={onFeedbackSent} onFeedbackError={onFeedbackError} />
    </>
  )
}
```

#### Feedback context for injection (no `jest.mock` needed)

```typescript
// feedback-context.tsx (app code)
import React, { createContext } from 'react'

type FeedbackResult = { estimatedResponseTime: string }
type FeedbackAPI = { sendFeedback: (title: string, description: string) => Promise<FeedbackResult> }

export const FeedbackContext = createContext<FeedbackAPI | null>(null)
export const FeedbackProvider = FeedbackContext.Provider

export function useFeedback(): FeedbackAPI {
  const api = React.useContext(FeedbackContext)
  if (!api) throw new Error('useFeedback must be used within FeedbackProvider')
  return api
}
```

#### Testing the GOOD pattern (behavioural test via injection)

```typescript
// FeedbackForm.test.tsx
import React from 'react'
import { render, fireEvent, waitFor } from '@testing-library/react'
import '@testing-library/jest-dom'
import { FeedbackForm } from './FeedbackForm'
import { FeedbackProvider } from './feedback-context'

test('calls onFeedbackSent on successful submit', async () => {
  const onFeedbackSent = jest.fn()
  const onFeedbackError = jest.fn()
  const feedbackApi = {
    sendFeedback: jest.fn().mockResolvedValue({ estimatedResponseTime: '24h' })
  }

  const { container } = render(
    <FeedbackProvider value={feedbackApi}>
      <FeedbackForm onFeedbackSent={onFeedbackSent} onFeedbackError={onFeedbackError} />
    </FeedbackProvider>
  )

  const form = container.querySelector('form') as HTMLFormElement
  fireEvent.submit(form)

  await waitFor(() => {
    expect(onFeedbackSent).toHaveBeenCalledWith({ estimatedResponseTime: '24h' })
  })
  expect(onFeedbackError).not.toHaveBeenCalled()
})
```

### BAD — Hook directly depends on axios and API details

```typescript
import axios from 'axios'

function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    axios
      .get(`https://api.example.com/users/${userId}`, {
        headers: {
          Authorization: `Bearer ${localStorage.getItem('token')}`,
        },
      })
      .then((response) => {
        setUser({
          id: response.data.id,
          name: response.data.full_name,
          email: response.data.email_address,
          createdAt: new Date(response.data.created_timestamp),
        })
      })
      .catch((err) => {
        if (err.response?.status === 404) {
          setError(new Error('User not found'))
        } else {
          setError(err)
        }
      })
      .finally(() => setLoading(false))
  }, [userId])

  return { user, loading, error }
}
```

### GOOD — Hook depends on interfaces

> Hook depend on interfaces; implementations are injected.

```typescript
interface UserApi {
  getUser(id: string): Promise<User>
  updateUser(id: string, data: Partial<User>): Promise<User>
  deleteUser(id: string): Promise<void>
}

// Preferred: pass the dependency in (great for tests)
function useUser(userId: string, userApi: UserApi) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    let cancelled = false
    setLoading(true)
    setError(null)

    userApi
      .getUser(userId)
      .then((u) => !cancelled && setUser(u))
      .catch((e) => !cancelled && setError(e as Error))
      .finally(() => !cancelled && setLoading(false))

    return () => {
      cancelled = true
    }
  }, [userId, userApi])

  const updateUser = useCallback(
    async (data: Partial<User>) => {
      const updated = await userApi.updateUser(userId, data)
      setUser(updated)
      return updated
    },
    [userId, userApi]
  )

  return { user, loading, error, updateUser }
}

// Optional adapter: pull the dependency from context, then delegate
const UserApiContext = React.createContext<UserApi | null>(null)
const useUserApi = (): UserApi => {
  const api = React.useContext(UserApiContext)
  if (!api) throw new Error('useUserApi must be used within a Provider')
  return api
}

function useUserWithApi(userId: string) {
  const api = useUserApi()
  return useUser(userId, api)
}
```

#### The good pattern allows us to test the interaction between UI state and the API with dependency injection

```typescript
// useUser.test.tsx
import { renderHook, act, waitFor } from '@testing-library/react'
import '@testing-library/jest-dom'
import type { UserApi, User } from './user-api-types'
import { useUser } from './useUser'

describe('useUser', () => {
  test('loads user and exposes updateUser', async () => {
    const mockUser: User = {
      id: 'u1',
      name: 'Amy',
      email: 'amy@example.com',
      createdAt: new Date('2024-01-01T00:00:00Z'),
    }

    const updatedUser: User = { ...mockUser, name: 'Amy Pond' }

    const mockApi: jest.Mocked<UserApi> = {
      getUser: jest.fn().mockResolvedValue(mockUser),
      updateUser: jest.fn().mockResolvedValue(updatedUser),
      deleteUser: jest.fn().mockResolvedValue(undefined),
    }

    const { result } = renderHook(() => useUser('u1', mockApi))

    // initial state
    expect(result.current.loading).toBe(true)
    expect(result.current.user).toBeNull()
    expect(result.current.error).toBeNull()

    // resolves getUser
    await waitFor(() => expect(result.current.loading).toBe(false))
    expect(mockApi.getUser).toHaveBeenCalledWith('u1')
    expect(result.current.user).toEqual(mockUser)

    // update path
    await act(async () => {
      await result.current.updateUser({ name: 'Amy Pond' })
    })
    expect(mockApi.updateUser).toHaveBeenCalledWith('u1', { name: 'Amy Pond' })
    expect(result.current.user).toEqual(updatedUser)
  })
})
```

## When to Apply DIP in React

### Use DIP for

* API/backend service calls
* Authentication and authorization
* Data persistence (localStorage, IndexedDB)
* Third-party service integrations
* Real-time connections (WebSocket, SSE)
* Complex business logic
* Separate concerns between UI presentation - UI behaviour - Business logic - Persistence

### Keep Concrete for

* Simple utility functions
* Pure transformations
* UI-only logic
* Component-specific helpers

## Anti-patterns to Avoid

1. **Direct API calls in components**: fetch/axios in components
2. **Hard-coded URLs**: API endpoints in component code
3. **Direct storage access**: localStorage/sessionStorage in components
4. **Tight coupling to libraries**: Direct use of specific libraries throughout

## React-Specific DIP Techniques

1. **Props injection** for component-level dependencies
2. **Context providers** for app-wide services
3. **Custom hooks** as service facades
4. **Factory functions** for creating configured services
5. **Higher-order components** for service injection

## Key Takeaways

* Components should **depend on interfaces**, not implementations
* **Inject dependencies** via props or context
* Keep **infrastructure details** out of components
* Use **abstraction layers** for external services
* **Testing becomes trivial** with mock implementations

## Related Best Practices

For dependency injection patterns and testing, see
👉 [best-practices.md](../best-practices/best-practices.md)

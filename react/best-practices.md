# React Best Practices

This document collects general coding practices that cut across multiple
principles. It complements the [Clean Code-focused docs](./clean-code/) (e.g.
DIP, ISP, DRY) with conventions for TypeScript interfaces, error handling, testing, and
component management.

---

## TypeScript Interfaces

### Discriminated Unions vs Optional Fields

#### BAD - Fat type with optional fields

  ```typescript
  // BAD - Unclear which fields are valid when
  type AsyncState<T> = {
    status: 'loading' | 'error' | 'success'
    data?: T
    error?: Error
  }
  ```

#### Good discriminated union with strongly defined types

  ```typescript
  // GOOD - Type-safe, no optional fields
  type LoadingState = { status: 'loading' }
  type ErrorState = { status: 'error'; error: Error }
  type SuccessState<T> = { status: 'success'; data: T }
  
  type AsyncState<T> = LoadingState | ErrorState | SuccessState<T>
  
  function handleState<T>(state: AsyncState<T>) {
    switch (state.status) {
      case 'loading':
        return <Spinner />
      case 'error':
        return <ErrorMessage error={state.error} />
      case 'success':
        return <DataDisplay data={state.data} />
    }
  }
  ```

## Loading and Error states

### Use ErrorBoundaries and Suspense to handle loading states

```typescript
function UserInfoError() {
  // error boundary implementation
}

function UserInfoLoading() {
  // loading state iplementation
}

interface Props {
  userID: string
}

function UserInfoDisplay({user}: {user: User}) {
  // display
}

function UserInfo({ userId }: Props) {
  const user = useUser(userId) // data loading with suspense/error boundary support

  return (<UserInfoDisplay user={user} />)
}

function UserInfoSection({ userID }: Props) {

  return (
    <ErrorBoundary fallback={<UserInfoError />}>
      <Suspense fallback={UserInfoLoading />}>
        <UserInfo userID={userID} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

## Component Composition to allow for reuse and efficient renders

- Use composition over inheritance
- Leverage children and render props:

```typescript
function Layout({ header, sidebar, children }: LayoutProps) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </div>
  )
}
```

## Component structure

### 1. Display Components at the core

- Pure functions that render UI based on props
- Only cares about displaying information and other components
- No business logic or side effects
- Example:

```typescript
interface UserCardProps {
  user: User
  onEdit: (user: User) => void
  onDelete: (id: string) => void
}

function UserCardDisplay({ user, onEdit, onDelete }: UserCardProps) {
  return (
    <div className="user-card">
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <button onClick={() => onEdit(user)}>Edit</button>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </div>
  )
}
```

### 2. Container Components that sets up async state and callbacks

- Handle data fetching and render logic via hooks
- Pass data and callbacks to display components
- Does not care about display at all
- Example:

```typescript
function UserCard({userID}: Props) {
  const {user, updateUser, deleteUser} = useUser(userId)

  return (<UserCardDisplay
    user={user}
    onEdit={updateUser}
    onDelete={deleteUser}
  />)
}
```

### 3. (Optional) loading/error boundary close to the component

- Handle error and loading states close to the page/screen section

```typescript
function UserSection({userID}: Props) {
  return (<ErrorBoundary fallback={<UserCardError />}>)
    <Suspense fallback={<UserCardLoading />}>
      <UserCard userId={userId} />
    </Suspense>
  </ErrorBoundary>)
}
```

### 4. Custom Hooks that encapsulates async data loading more complex ui state

- Encapsulate all data loading logic and side effects
- Return data and functions for components to use
- Accept dependencies for testability
- Wrap in hook that loads dependencies from provider

```typescript
const useUsersWithApi = (userApi: UserApi) => {
  const [users, setUsers] = useState<User[]>([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)
  
  useEffect(() => {
    userApi.getUsers()
      .then(setUsers)
      .catch(setError)
      .finally(() => setLoading(false))
  }, [userApi])
  
  return { users, loading, error }
}

const useUsers = () => {
  const usersApi = useUsersApi()

  return useUsersWithApi(usersApi)
}
```

## Testing Strategy

### Component Testing

#### 1. Focus on testing behaviour, not display details

- Test initial loading/error/render if component is loading data
- Test success/error/loading of user actions

```typescript
// UserCreationForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

test('submit form fails with validation errors when form is incomplete', async () => {
  const userApi = {
    createUser: jest.fn()
  }
  
  const onUserCreated = jest.fn()
  const onCreateUserError = jest.fn()
  
  const { container } = render(
    <APIProvider value={{userApi}}}>
      <UserCreationForm onUserCreated={onUserCreated} onCreateUserError={onCreateUserError} />
    </FeedbackProvider>
  )
  
  const formButton = container.querySelector('FormButton') as HTMLFormElement
  formButton.click()
  
  expect(onUserCreated).not.toHaveBeenCalled()
  expect(onCreateUserError).not.toHaveBeenCalled()
  // assert validation errors
})

test('submit form fails to error boundary when createUser return error', async () => {
  // test implementation asserting on UI error state and onCreateUserError called
})

test('submit form succeeds when form is valid and createUser returns success, async () => {
  // test implementation asserting on UI success state and onUserCreated called
})
```

#### 2. Test render behaviour

- Test initial renders
  - First render
  - Render after data has loaded
- Test to check for unexpected re-renders. Component should not re-render if props stays the same.

#### 3. For critical UI components different display states can be regression tested with snapshots

### Hook Testing

#### 1. Test hooks in isolation with renderHook

```typescript
import { renderHook, act } from '@testing-library/react'

test('useCounter increments value', () => {
  const { result } = renderHook(() => useCounter())
  
  expect(result.current.count).toBe(0)
  
  act(() => {
    result.current.increment()
  })
  
  expect(result.current.count).toBe(1)
})
```

#### 2. Dependency Injection for Testing

> **NO jest.mock()** - Use dependency injection instead

```typescript
// BAD - Using jest.mock
jest.mock('../apiClients/userApi')

// GOOD - Dependency injection
function useUsers(userApi: UserApi) {
  const [users, setUsers] = useState<User[]>([])
  
  useEffect(() => {
    userApi.getUsers().then(setUsers)
  }, [userApi])
  
  return users
}

// In tests, pass a stub
const userApi: UserApi = {
  getUsers: jest.fn().mockResolvedValue(mockUsers)
}
```

## Dependency Injection

### Context for Cross-cutting Concerns

- Use React Context for app-wide dependencies:

```typescript
const ApiContext = createContext<ApiClients | null>(null)

function ApiProvider({ children, apiClients }: ApiProviderProps) {
  return (
    <ApiContext.Provider value={apiClients}>
      {children}
    </ApiContext.Provider>
  )
}

function useMessageApi(): ApiClients {
  const apiClients = useContext(ApiContext)
  if (!apiClients?.messageApi) {
    throw new Error('useMessageClient must be used within ApiProvider')
  }
  return apiClients.messageApi
}
```

## Performance Optimization

### Memoization

Use `useMemo` for expensive computations:

```typescript
function DataTable({ data, filters }: DataTableProps) {
  const filteredData = useMemo(
    () => applyFilters(data, filters),
    [data, filters]
  )
  
  return <Table data={filteredData} />
}
```

## State Management

### Local State

Keep state as close to where it's used as possible:

```typescript
function SearchBar() {
  const [query, setQuery] = useState('')
  
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  )
}
```

### Lifting State

Lift state only when multiple components need it:

```typescript
function Parent() {
  const [selectedUser, setSelectedUser] = useState<User | null>(null)
  
  return (
    <>
      <UserList onSelect={setSelectedUser} />
      <UserDetails user={selectedUser} />
    </>
  )
}
```

### Global State

- Use Context for truly glboal state. Shared UI state belongs in a container component/hook:

```typescript
// Only for truly global state
const AuthContext = createContext<AuthState | null>(null)

function useAuth() {
  const auth = useContext(AuthContext)
  if (!auth) throw new Error('Auth not provided')
  return auth
}
```

## Key Takeaways

- Use **discriminated unions** over types with optional fields
- **No jest.mock()** - use dependency injection with stubs
- Encapsulate **async data loading** and **complex render** logic in hooks with injected dependencies
- Keep **components focused** - separate display from logic
- Test **behavior, not implementation**
- Keep **state local** unless sharing is necessary

---

## Further Reading

- [React Documentation](https://react.dev)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [Testing Library Best Practices](https://testing-library.com/docs/guiding-principles)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app)
